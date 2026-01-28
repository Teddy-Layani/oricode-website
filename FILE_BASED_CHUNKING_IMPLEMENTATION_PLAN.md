# File-Based Chunking Implementation Plan
## January 22, 2026

---

## Executive Summary

**Current Problem**: AI hits 8,192 token output limit when generating large code chunks, causing CHUNK_MISSING_CODE errors.

**Solution**: Store chunks as local files on client machine (C:\temp) instead of sending to backend immediately. This eliminates token limits and provides better debugging, recovery, and user control.

**Advantages**:
- ✅ No token limits (files don't have token constraints)
- ✅ Zero backend infrastructure cost (each user uses their own machine)
- ✅ Natural microservicing (scales with users automatically)
- ✅ Better error recovery (files persist through timeouts)
- ✅ User visibility (can inspect files at any time)
- ✅ Offline capability (work persists even if backend is down)

---

## Architecture Overview

### Current Flow (Problematic)
```
AI generates chunk → Send to Railway backend → Store in memory → Repeat
→ Backend assembles chunks → Deploy to SAP
```

**Issues**:
- Token limit prevents sending large chunks
- Memory-based (lost on backend restart)
- No user visibility
- Poor error recovery

### New Flow (File-Based)
```
AI generates chunk → Client writes to C:\temp\oricode\{session}\chunk_001.abap
AI generates chunk → Client writes to C:\temp\oricode\{session}\chunk_002.abap
...
AI generates chunk → Client writes to C:\temp\oricode\{session}\chunk_015.abap

→ Client merges all chunks → C:\temp\oricode\{session}\complete.abap
→ Client sends complete code to backend for syntax check
→ If errors: AI reads complete.abap, fixes, writes back, repeat
→ When OK: Send complete code to backend for deployment
→ Client cleans up temp folder
```

**Benefits**:
- No token limits
- Persistent storage
- User can inspect/debug
- Better error recovery
- Works offline

---

## Implementation Components

### 1. New Eclipse Tools (ToolRouter.java)

Add these new tools to `SUPPORTED_ECLIPSE_TOOLS`:

```java
// File-based chunking tools (add to line 94 in ToolRouter.java)
SUPPORTED_ECLIPSE_TOOLS.add("write_chunk_file");       // Write chunk to local file
SUPPORTED_ECLIPSE_TOOLS.add("merge_chunk_files");      // Merge all chunks into complete file
SUPPORTED_ECLIPSE_TOOLS.add("read_chunk_file");        // Read a specific chunk or complete file
SUPPORTED_ECLIPSE_TOOLS.add("list_chunk_files");       // List all chunks in session
SUPPORTED_ECLIPSE_TOOLS.add("cleanup_chunk_session");  // Delete session folder
```

### 2. New Tool Implementation (EclipseTools.java)

**Location**: `c:\Users\teddy.layani\eclipse-workspace\oricode-poc\src\com\oricode\eclipse\ai\tools\EclipseTools.java`

#### Tool 1: write_chunk_file
```java
/**
 * Write a code chunk to a local file
 *
 * @param session_id - Unique session identifier (e.g., "ZCL_CLASS_1737493000")
 * @param chunk_index - Zero-based chunk index (0, 1, 2, ...)
 * @param total_chunks - Total number of chunks expected
 * @param code_chunk - The ABAP code for this chunk
 * @param object_name - Class/program name
 * @param object_type - "class" or "program"
 * @return Success message with file path
 */
public String writeChunkFile(
    String sessionId,
    int chunkIndex,
    int totalChunks,
    String codeChunk,
    String objectName,
    String objectType
) {
    // Create session directory: C:\temp\oricode\{session_id}\
    Path sessionDir = Paths.get("C:", "temp", "oricode", sessionId);
    Files.createDirectories(sessionDir);

    // Write chunk file: chunk_001.abap, chunk_002.abap, etc.
    String fileName = String.format("chunk_%03d.abap", chunkIndex + 1);
    Path chunkFile = sessionDir.resolve(fileName);
    Files.writeString(chunkFile, codeChunk, StandardCharsets.UTF_8);

    // Write metadata file: chunk_001.meta.json
    String metaFileName = String.format("chunk_%03d.meta.json", chunkIndex + 1);
    Path metaFile = sessionDir.resolve(metaFileName);
    String metadata = String.format(
        "{\"chunk_index\":%d,\"total_chunks\":%d,\"object_name\":\"%s\",\"object_type\":\"%s\",\"size\":%d,\"timestamp\":\"%s\"}",
        chunkIndex, totalChunks, objectName, objectType, codeChunk.length(), Instant.now()
    );
    Files.writeString(metaFile, metadata, StandardCharsets.UTF_8);

    // Log success
    logger.info(String.format("[FileChunk] Stored chunk %d/%d for session %s (%d chars)",
        chunkIndex + 1, totalChunks, sessionId, codeChunk.length()));

    return String.format("✅ Chunk %d/%d written to: %s\n📦 Size: %d characters\n📁 Session: %s",
        chunkIndex + 1, totalChunks, chunkFile, codeChunk.length(), sessionDir);
}
```

#### Tool 2: merge_chunk_files
```java
/**
 * Merge all chunk files into a single complete file
 *
 * @param session_id - Session identifier
 * @return Success message with complete file path and validation results
 */
public String mergeChunkFiles(String sessionId) {
    Path sessionDir = Paths.get("C:", "temp", "oricode", sessionId);

    // Verify session exists
    if (!Files.exists(sessionDir)) {
        return "❌ Session not found: " + sessionId;
    }

    // Read metadata from first chunk to get total_chunks
    Path firstMeta = sessionDir.resolve("chunk_001.meta.json");
    if (!Files.exists(firstMeta)) {
        return "❌ No chunks found in session: " + sessionId;
    }

    String metaJson = Files.readString(firstMeta);
    // Parse JSON to get total_chunks (simple string parsing or use JSON library)
    int totalChunks = extractTotalChunksFromMeta(metaJson);

    // Verify all chunks exist
    List<String> missingChunks = new ArrayList<>();
    for (int i = 1; i <= totalChunks; i++) {
        String fileName = String.format("chunk_%03d.abap", i);
        if (!Files.exists(sessionDir.resolve(fileName))) {
            missingChunks.add(String.valueOf(i));
        }
    }

    if (!missingChunks.isEmpty()) {
        return "❌ Missing chunks: " + String.join(", ", missingChunks);
    }

    // Merge all chunks
    StringBuilder completeCode = new StringBuilder();
    int totalSize = 0;

    for (int i = 1; i <= totalChunks; i++) {
        String fileName = String.format("chunk_%03d.abap", i);
        Path chunkFile = sessionDir.resolve(fileName);
        String chunkCode = Files.readString(chunkFile, StandardCharsets.UTF_8);
        completeCode.append(chunkCode);
        totalSize += chunkCode.length();
    }

    // Write complete file
    Path completeFile = sessionDir.resolve("complete.abap");
    Files.writeString(completeFile, completeCode.toString(), StandardCharsets.UTF_8);

    // Write merge metadata
    Path mergeMeta = sessionDir.resolve("complete.meta.json");
    String metadata = String.format(
        "{\"total_chunks\":%d,\"total_size\":%d,\"timestamp\":\"%s\",\"status\":\"merged\"}",
        totalChunks, totalSize, Instant.now()
    );
    Files.writeString(mergeMeta, metadata, StandardCharsets.UTF_8);

    // Validate structure
    String validationResult = validateAbapStructure(completeCode.toString());

    logger.info(String.format("[FileChunk] Merged %d chunks into complete file (%d chars)",
        totalChunks, totalSize));

    String result = String.format("✅ Merged %d chunks into complete file\n", totalChunks);
    result += String.format("📦 Total size: %d characters\n", totalSize);
    result += String.format("📁 File: %s\n", completeFile);
    result += String.format("🔍 Structure validation: %s", validationResult);

    return result;
}
```

#### Tool 3: read_chunk_file
```java
/**
 * Read a chunk file or the complete merged file
 *
 * @param session_id - Session identifier
 * @param file_name - Optional: "complete.abap" or "chunk_001.abap" (default: "complete.abap")
 * @return File contents
 */
public String readChunkFile(String sessionId, String fileName) {
    if (fileName == null || fileName.isEmpty()) {
        fileName = "complete.abap";
    }

    Path sessionDir = Paths.get("C:", "temp", "oricode", sessionId);
    Path targetFile = sessionDir.resolve(fileName);

    if (!Files.exists(targetFile)) {
        return "❌ File not found: " + targetFile;
    }

    String content = Files.readString(targetFile, StandardCharsets.UTF_8);

    return String.format("📄 File: %s\n📦 Size: %d characters\n\n%s",
        targetFile, content.length(), content);
}
```

#### Tool 4: list_chunk_files
```java
/**
 * List all chunk files in a session
 *
 * @param session_id - Session identifier
 * @return List of files with metadata
 */
public String listChunkFiles(String sessionId) {
    Path sessionDir = Paths.get("C:", "temp", "oricode", sessionId);

    if (!Files.exists(sessionDir)) {
        return "❌ Session not found: " + sessionId;
    }

    StringBuilder result = new StringBuilder();
    result.append("📋 CHUNK FILES FOR SESSION: ").append(sessionId).append("\n");
    result.append("═".repeat(60)).append("\n\n");

    // List all .abap files
    List<Path> abapFiles = Files.list(sessionDir)
        .filter(p -> p.toString().endsWith(".abap"))
        .sorted()
        .collect(Collectors.toList());

    for (Path file : abapFiles) {
        long size = Files.size(file);
        String lastModified = Files.getLastModifiedTime(file).toString();
        result.append(String.format("📄 %s (%d chars, modified: %s)\n",
            file.getFileName(), size, lastModified));
    }

    result.append("\n📁 Session directory: ").append(sessionDir);

    return result.toString();
}
```

#### Tool 5: cleanup_chunk_session
```java
/**
 * Delete all files in a session folder
 *
 * @param session_id - Session identifier
 * @return Success message
 */
public String cleanupChunkSession(String sessionId) {
    Path sessionDir = Paths.get("C:", "temp", "oricode", sessionId);

    if (!Files.exists(sessionDir)) {
        return "✅ Session already cleaned up: " + sessionId;
    }

    // Delete all files in session
    Files.walk(sessionDir)
        .sorted(Comparator.reverseOrder())
        .forEach(Files::delete);

    logger.info("[FileChunk] Cleaned up session: " + sessionId);

    return "✅ Session cleaned up: " + sessionId;
}
```

---

### 3. Updated System Prompt

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-backend\src\utils\systemPromptComposable.ts`

**Lines to update**: 42-70 (chunking workflow section)

**NEW WORKFLOW**:
```markdown
### For LOCAL CLASS Methods (lcl_* classes in locals include)
1. **Always use file-based chunking for locals >= 4K chars**:
   - Read: mcp_get_class_locals(class_name, include_type="implementations")
   - Modify the ENTIRE locals include file (all local classes together)
   - If locals >= 4K chars → USE FILE-BASED CHUNKING (see below)

## FILE-BASED CHUNKING WORKFLOW (New Strategy for Large Code)

**When to use**: Any code >= 4,000 characters

**Steps**:

1. **Create session ID**: Use format: `{OBJECT_NAME}_{timestamp}`
   Example: `ZCL_BM_LEARNING_S4D401_1737493000`

2. **Calculate chunks**: `chunk_count = ceil(code_length / 3000)`
   - Use 3K chars per chunk (smaller = safer within token limits)
   - Example: 60K code = 20 chunks

3. **Write chunks to local files** (NO backend calls during chunking!):
   ```
   For each chunk (index 0 to N-1):
     write_chunk_file(
       session_id = session_id,
       chunk_index = i,
       total_chunks = chunk_count,
       code_chunk = chunk_code,
       object_name = object_name,
       object_type = "class"
     )
   ```

4. **Merge chunks into complete file**:
   ```
   merge_chunk_files(session_id = session_id)

   This creates: C:\temp\oricode\{session_id}\complete.abap
   ```

5. **Validate syntax**:
   ```
   # Read complete file
   complete_code = read_chunk_file(session_id = session_id, file_name = "complete.abap")

   # Send to backend for syntax check
   syntax_result = mcp_check_syntax(
     object_type = "class",
     object_name = object_name,
     source_code = complete_code,
     include_type = "implementations"
   )
   ```

6. **If syntax errors found**:
   ```
   # Read complete file
   current_code = read_chunk_file(session_id = session_id, file_name = "complete.abap")

   # Fix the error in the code
   fixed_code = fix_syntax_error(current_code, syntax_error)

   # Write back to complete file (overwrite chunk_001.abap as complete file)
   write_chunk_file(
     session_id = session_id,
     chunk_index = 0,  # Use index 0 for complete file
     total_chunks = 1,  # Only 1 "chunk" (the complete file)
     code_chunk = fixed_code,
     object_name = object_name,
     object_type = "class"
   )

   # Rename to complete.abap
   # Then re-check syntax (repeat step 5)
   ```

7. **When syntax is correct, deploy**:
   ```
   # Read complete file
   final_code = read_chunk_file(session_id = session_id, file_name = "complete.abap")

   # Send to backend for deployment (use existing chunked deployment)
   # Backend will handle chunking for transmission
   mcp_deploy_code(
     object_type = "class",
     object_name = object_name,
     source_code = final_code,
     include_type = "implementations",
     package = package_name,
     transport = transport_request,
     user_confirmed = false
   )
   ```

8. **Cleanup session**:
   ```
   cleanup_chunk_session(session_id = session_id)
   ```

**CRITICAL DIFFERENCES FROM OLD CHUNKING**:
- ❌ No more mcp_store_code_chunk calls
- ❌ No more mcp_deploy_stored calls
- ❌ No more backend memory storage
- ✅ Write chunks as LOCAL FILES (no token limits!)
- ✅ Merge locally, then validate
- ✅ Deploy only when syntax is correct
- ✅ User can inspect files at any time

**Benefits**:
- No 8,192 token output limit issues
- Persistent storage (survives timeouts)
- Better error recovery
- User visibility into progress
- Can fix errors by editing complete file
- Natural microservicing (each user's machine)
```

---

### 4. Tool Definitions (EclipseTools.getToolDefinitionsJson())

**Location**: `c:\Users\teddy.layani\eclipse-workspace\oricode-poc\src\com\oricode\eclipse\ai\tools\EclipseTools.java`

Add these tool definitions to the JSON array:

```json
{
  "name": "write_chunk_file",
  "description": "Write a code chunk to a local file on the user's machine. Use this for file-based chunking workflow (large code >= 4K chars). Files are written to C:\\temp\\oricode\\{session_id}\\chunk_NNN.abap",
  "input_schema": {
    "type": "object",
    "properties": {
      "session_id": {
        "type": "string",
        "description": "Unique session identifier (format: {OBJECT_NAME}_{timestamp})"
      },
      "chunk_index": {
        "type": "integer",
        "description": "Zero-based chunk index (0, 1, 2, ...)"
      },
      "total_chunks": {
        "type": "integer",
        "description": "Total number of chunks expected in this session"
      },
      "code_chunk": {
        "type": "string",
        "description": "The ABAP code for this chunk"
      },
      "object_name": {
        "type": "string",
        "description": "Name of the class/program being modified"
      },
      "object_type": {
        "type": "string",
        "description": "Type of object: 'class' or 'program'"
      }
    },
    "required": ["session_id", "chunk_index", "total_chunks", "code_chunk", "object_name", "object_type"]
  }
},
{
  "name": "merge_chunk_files",
  "description": "Merge all chunk files in a session into a single complete.abap file. This combines chunk_001.abap + chunk_002.abap + ... into complete.abap in the session directory.",
  "input_schema": {
    "type": "object",
    "properties": {
      "session_id": {
        "type": "string",
        "description": "Session identifier"
      }
    },
    "required": ["session_id"]
  }
},
{
  "name": "read_chunk_file",
  "description": "Read a chunk file or the complete merged file from a session. Use this to read complete.abap for syntax checking or error fixing.",
  "input_schema": {
    "type": "object",
    "properties": {
      "session_id": {
        "type": "string",
        "description": "Session identifier"
      },
      "file_name": {
        "type": "string",
        "description": "Optional: File name to read (default: 'complete.abap'). Can be 'chunk_001.abap', 'chunk_002.abap', etc."
      }
    },
    "required": ["session_id"]
  }
},
{
  "name": "list_chunk_files",
  "description": "List all chunk files in a session directory. Use this to verify chunks were written correctly.",
  "input_schema": {
    "type": "object",
    "properties": {
      "session_id": {
        "type": "string",
        "description": "Session identifier"
      }
    },
    "required": ["session_id"]
  }
},
{
  "name": "cleanup_chunk_session",
  "description": "Delete all files in a session directory. Call this after successful deployment to clean up temporary files.",
  "input_schema": {
    "type": "object",
    "properties": {
      "session_id": {
        "type": "string",
        "description": "Session identifier"
      }
    },
    "required": ["session_id"]
  }
}
```

---

### 5. Backend Changes (Optional Enhancement)

**File**: `c:\Users\teddy.layani\eclipse-workspace\mcp-abap-adt-crud\src\index.ts`

**Option A**: Keep existing chunked deployment tools, just change how they're called
- Client sends complete code via existing `mcp_deploy_code` or chunked tools
- Backend handles transmission chunking (network layer only)
- No changes needed to backend!

**Option B**: Add new single-call deployment for complete files
```typescript
// New tool: Deploy complete code in one call
{
  name: "DeployCompleteCode",
  description: "Deploy complete code (already validated) to SAP system. Use this after file-based chunking workflow when code is ready.",
  inputSchema: {
    type: "object",
    properties: {
      object_type: { type: "string" },
      object_name: { type: "string" },
      source_code: { type: "string" },
      include_type: { type: "string" },
      package: { type: "string" },
      transport: { type: "string" },
      user_confirmed: { type: "boolean" }
    }
  }
}
```

---

## Migration Strategy

### Phase 1: Implement New Tools (Week 1)
1. Add 5 new tools to Eclipse plugin
2. Add tool definitions to EclipseTools
3. Update ToolRouter to route new tools
4. Test locally with Eclipse

### Phase 2: Update System Prompt (Week 1)
1. Update systemPromptComposable.ts with new workflow
2. Document file-based chunking process
3. Deploy to Railway
4. Test with small class (< 10K chars)

### Phase 3: Test with Real Workload (Week 2)
1. Test with 60K+ local class modification
2. Verify no CHUNK_MISSING_CODE errors
3. Verify complete file is correct
4. Verify syntax checking works
5. Verify deployment succeeds

### Phase 4: Production Rollout (Week 2)
1. Update VS Code extension with same tools
2. Monitor error rates
3. Collect user feedback
4. Document in user guide

---

## Success Metrics

### Before File-Based Chunking (Current State)
- ❌ CHUNK_MISSING_CODE errors on large code
- ❌ Token limit hit frequently (8,192 tokens)
- ❌ Manual "continue" prompts needed
- ❌ No user visibility into progress
- ❌ Poor error recovery (lost on timeout)

### After File-Based Chunking (Expected)
- ✅ Zero CHUNK_MISSING_CODE errors
- ✅ No token limit issues (files don't have limits)
- ✅ No manual prompts needed
- ✅ Full user visibility (C:\temp\oricode\)
- ✅ Perfect error recovery (files persist)
- ✅ Better debugging (can inspect files)
- ✅ Faster workflow (no backend overhead during chunking)

---

## Testing Checklist

### Unit Tests
- [ ] write_chunk_file creates files correctly
- [ ] merge_chunk_files combines chunks correctly
- [ ] read_chunk_file reads content correctly
- [ ] list_chunk_files shows all files
- [ ] cleanup_chunk_session deletes session
- [ ] Missing chunks detected properly
- [ ] Metadata files created correctly

### Integration Tests
- [ ] Small code (< 4K): Direct deployment still works
- [ ] Medium code (4K-20K): File-based chunking works
- [ ] Large code (60K+): 20+ chunks work correctly
- [ ] Syntax errors detected and fixed
- [ ] Deployment succeeds after validation
- [ ] Session cleanup works
- [ ] Timeouts don't lose work (files persist)

### Edge Cases
- [ ] Empty chunks handled
- [ ] Duplicate chunks overwritten correctly
- [ ] Missing chunks detected
- [ ] Disk full error handling
- [ ] Permission denied error handling
- [ ] Session ID conflicts handled

---

## Rollback Plan

If file-based chunking has issues:

1. **Immediate**: Revert system prompt to old chunking workflow
2. **Keep**: New Eclipse tools (they don't break anything)
3. **Investigate**: Why file-based chunking failed
4. **Fix**: Address issues and re-deploy

Old workflow remains intact and functional.

---

## Documentation Updates

### User Documentation
- [ ] Update user guide with file-based chunking explanation
- [ ] Document C:\temp\oricode\ folder usage
- [ ] Explain how to inspect chunk files
- [ ] Document cleanup process
- [ ] Add troubleshooting for file permission issues

### Developer Documentation
- [ ] Document new Eclipse tools
- [ ] Update architecture diagrams
- [ ] Document testing procedures
- [ ] Add code examples

---

## Estimated Effort

### Development
- Eclipse tools implementation: 2 days
- System prompt update: 0.5 days
- Testing: 2 days
- **Total**: 4.5 days

### Deployment
- Railway deployment: 5 minutes (auto-deploy)
- Eclipse plugin bundle: 10 minutes
- VS Code extension: 1 day (if needed immediately)

### Risk Level
**LOW** - New tools are additive, don't break existing functionality

---

## Next Steps

1. ✅ Create implementation plan (this document)
2. ⏳ Review plan with stakeholder
3. ⏳ Implement Eclipse tools
4. ⏳ Update system prompt
5. ⏳ Test with 60K+ local class
6. ⏳ Deploy to production
7. ⏳ Monitor and iterate

---

**Created**: January 22, 2026
**Author**: Claude Code
**Status**: READY FOR REVIEW
**Impact**: HIGH - Solves critical CHUNK_MISSING_CODE issue permanently
