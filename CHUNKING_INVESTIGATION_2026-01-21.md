# Chunking Investigation - January 21, 2026

## Problem Statement

User reported: "can you see why it stopped?" and "this time only chunk"

At 11:40:05, a deployment appeared successful in the chunking phase (4 chunks stored, 57,084 total chars) but failed at SAP level with:
```
❌ The statement CLASS ... DEFINITION ... . is unexpected
```

## Root Cause Analysis

### Issue 1: Inconsistent Chunk Sizes in System Prompt

**Evidence**: systemPromptComposable.ts had conflicting instructions:
- Lines 97-98: Use **8000 char chunks** ✅
- Line 121: Says NOT to use "3000-char chunks instead of **15000-char chunks**" ❌
- Line 126: Says to use "**15000-char chunks**" ❌
- Lines 202-211: Says to use **15000 char chunks** ❌
- Lines 207-210: Says "Send chunks in batches (max 2-3 chunks per response)" ❌

**Result**: Claude followed the wrong instructions and generated:
- Chunk 1: 16,657 chars (2x too large)
- Chunk 2: 19,147 chars (2.4x too large)
- Chunk 3: 19,430 chars (2.4x too large)
- Chunk 4: 1,850 chars

### Issue 2: Multiple Chunks Per Message

**Evidence**: System prompt said "Send chunks in batches (max 2-3 chunks per response)"

**Problem**: With 19K char chunks + tool call overhead, sending multiple chunks in one message causes:
- Response size: ~20K chars per chunk × 2-3 chunks = 40-60K chars
- Token count: ~10K-15K tokens
- Result: Hits 8192 max_tokens limit, response truncates

### Issue 3: Check Syntax Still Being Skipped

**Evidence from logs**:
```
11:30:34 - [CHUNK_MISSING_CODE] Claude sent chunk without code_chunk parameter!
11:32:27 - [CHUNK_MISSING_CODE] Claude sent chunk without code_chunk parameter!
11:36:10 - mcp_store_code_chunk (chunk 0/4) - NO mcp_check_syntax before this!
```

**Problem**: Claude is STILL skipping `mcp_check_syntax` before calling `mcp_store_code_chunk`, even though the "CHECK SYNTAX FIRST" section exists.

**Why**: The CHECK SYNTAX FIRST section didn't explicitly mention chunk size requirements, so Claude may have thought:
1. "I need to check syntax" ✓
2. "But I'll use the chunk size mentioned later in the prompt" (15K) ✗

## Fixes Applied

### Fix 1: Make System Prompt Consistent (Commit ad7272f)

**File**: oricode-backend/src/utils/systemPromptComposable.ts

**Changes**:
1. Line 121-128: Changed from "15000-char chunks" to "8000-char chunks"
2. Line 183-185: Changed threshold from 15K to 10K
3. Lines 200-211: Replaced "Send chunks in batches" with "Send ONLY ONE chunk per message"

### Fix 2: Add Chunk Size to CHECK SYNTAX Section (Commit eff477e)

**File**: oricode-backend/src/utils/systemPromptComposable.ts

**Changes**:
1. Line 139: Added step 5️⃣ "If chunking needed: Use EXACTLY 8000-char chunks, send ONE chunk per message"
2. Lines 145-146: Added explicit prohibitions:
   - ❌ Use 15K, 16K, 19K, or any chunk size other than 8000 chars
   - ❌ Send multiple chunks in one message
3. Lines 151-156: Updated CORRECT workflow example to show:
   - 25000 chars → 4 chunks of 8000 chars
   - MESSAGE 1: chunk 0 → STOP
   - MESSAGE 2: chunk 1 → STOP
   - MESSAGE 3: chunk 2 → STOP
   - MESSAGE 4: chunk 3 → STOP
   - MESSAGE 5: deploy

## Why 8000 Characters?

**Math**:
- Average char-to-token ratio for code: ~4:1
- 8000 chars = ~2000 tokens
- Tool call overhead: ~200 tokens
- Response total: ~2200 tokens per chunk
- With 8192 max_tokens limit: Safe margin of 3.7x

**Benefits**:
- ✅ Fits comfortably in 8192 token response limit
- ✅ Allows room for tool call overhead and metadata
- ✅ One chunk per message = no truncation risk
- ✅ Small enough to avoid timeouts
- ✅ Large enough to minimize API calls (90K code = ~12 chunks = 12 messages)

## Expected Behavior After Fixes

### For 90K Character Code (User's Use Case):

**Before fixes** (WRONG):
```
chunk_count = ceil(90000 / 15000) = 6 chunks
Chunk sizes: 16K, 19K, 19K, 16K, 12K, 8K
Messages: Try to send 2-3 chunks per message → TRUNCATION
Result: CHUNK_MISSING_CODE errors
```

**After fixes** (CORRECT):
```
1. mcp_check_syntax with full 90K code → ✓ OK
2. Calculate: chunk_count = ceil(90000 / 8000) = 12 chunks
3. MESSAGE 1: chunk 0 (8000 chars) → STOP → WAIT
4. MESSAGE 2: chunk 1 (8000 chars) → STOP → WAIT
5. MESSAGE 3: chunk 2 (8000 chars) → STOP → WAIT
... (continue for chunks 3-11, ONE per message)
14. MESSAGE 12: chunk 11 (2000 chars) → STOP → WAIT
15. MESSAGE 13: mcp_deploy_stored_code(user_confirmed=false) → PROPOSE
16. User clicks "Accept & Deploy"
17. Eclipse calls mcp_deploy_stored_code(user_confirmed=true) → DEPLOY ✓
```

## SAP Deployment Error Analysis

**Error**: "The statement CLASS ... DEFINITION ... . is unexpected"

**Possible Causes**:
1. **Chunk boundary breaks CLASS statement** - If chunk 0 ends mid-line during "CLASS ZCL_EXAMPLE DEFINITION"
2. **Duplicate CLASS DEFINITION** - If chunks overlap or MCP server duplicates headers
3. **Extra period** - "CLASS ... DEFINITION ... ." has two periods instead of one
4. **Character encoding issue** - Chunk concatenation introduces invalid characters

**Status**: NEEDS INVESTIGATION IN MCP SERVER
- Need to examine MCP server's chunk assembly logic
- Need to verify that chunks are concatenated with proper line breaks
- Need to ensure no header/footer duplication

## Testing Plan

### Priority 1: Verify Railway Deployment
1. Check Railway build logs to confirm commits deployed
2. Verify systemPromptComposable.ts has updated instructions in production

### Priority 2: Test with Known Good Code
1. Use ABAP code that's verified syntactically correct
2. Test with ~25K chars (should create 4 chunks of 8K each)
3. Monitor logs for:
   - mcp_check_syntax called FIRST ✓
   - Chunk sizes = ~8000 chars each ✓
   - ONE chunk per message ✓
   - No CHUNK_MISSING_CODE errors ✓
   - Deployment succeeds ✓

### Priority 3: Test with 90K Code (User's Use Case)
1. Use the user's 90K char, 30K line ABAP code
2. Monitor logs for:
   - 12 chunks of ~8K chars
   - 12 separate messages (one per chunk)
   - No response truncation
   - Successful deployment

### Priority 4: Investigate SAP Error
1. If SAP still returns "unexpected CLASS DEFINITION":
   - Examine MCP server chunk assembly code
   - Add logging to show concatenated result before sending to SAP
   - Verify chunk boundaries don't break ABAP statements
   - Check for duplicate headers/footers

## Log Markers to Watch

### Successful Chunking (After Fix):
```
[MCP] TOOL_ROUTED: mcp_check_syntax               ← FIRST
[MCP] Syntax check: ✓ OK
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk
[MCP] CHUNK_PARAMS: chunk_index=0, code_length=8000  ← Correct size
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk
[MCP] CHUNK_PARAMS: chunk_index=1, code_length=8000  ← Correct size
...
[MCP] Chunks assembled: 12 chunks, 90000 characters
[MCP] TOOL_ROUTED: mcp_deploy_stored
[MCP] user_confirmed=false (propose)
[User clicks Accept]
[MCP] user_confirmed=true (deploy)
[MCP] ✓ Deployment successful
```

### Failed Chunking (Before Fix):
```
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk     ← NO syntax check!
[CHUNK_MISSING_CODE] Claude sent chunk without code_chunk parameter!
[MCP] CHUNK_PARAMS: chunk_index=0, code_length=19000  ← Too large
[MCP] CHUNK_PARAMS: chunk_index=1, code_length=19000  ← Too large
```

## Commits

```
eff477e - CRITICAL: Add explicit 8000-char chunk size to CHECK SYNTAX section, update examples
ad7272f - Fix system prompt inconsistencies: Use 8K chunks consistently, ONE chunk per message
0ca01e5 - (previous commits)
```

## Next Steps

1. ⏳ **Wait for Railway to deploy** commits eff477e
2. 🧪 **Test with 25K code** to verify 4 chunks of 8K each
3. 🧪 **Test with 90K code** to verify 12 chunks work end-to-end
4. 🔍 **If SAP error persists**: Investigate MCP server chunk assembly logic

---

**Status**: ✅ System Prompt Fixes Committed and Pushed
**Waiting On**: Railway deployment of commits ad7272f and eff477e
**Date**: January 21, 2026, 11:50 AM
**Analyst**: Claude (Sonnet 4.5)
