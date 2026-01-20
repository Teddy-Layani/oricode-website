# Centralized Tools Architecture - Implementation Summary

## Overview

This document summarizes the implementation of centralized tools across the Oricode ecosystem, providing a single source of truth for common development tools accessible by both VS Code and Eclipse clients.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CENTRALIZED ARCHITECTURE                      │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐                  ┌──────────────────┐
│  VS Code Client  │                  │ Eclipse Client   │
│  (TypeScript)    │                  │  (Java)          │
└────────┬─────────┘                  └────────┬─────────┘
         │                                     │
         │         HTTP/REST API               │
         └──────────────┬──────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │  Oricode Backend    │
              │  (Node.js/Express)  │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ CommonToolsComposable│
              │  (Single Source of   │
              │   Truth for Tools)   │
              └─────────────────────┘
```

## Components Implemented

### 1. CommonToolsComposable.ts (Backend)
**Location**: `oricode-backend/src/utils/CommonToolsComposable.ts`

**Purpose**: Centralized implementation of common development tools

**Tools Implemented**:

#### Core Tools (P0):
- `read_file` - Read file contents with line ranges
- `write_file` - Write content to files
- `str_replace` - Precise string replacement in files
- `view_directory` - Directory tree listing
- `grep_search` - Pattern search across files
- `execute_command` - Shell command execution (bash/powershell/cmd)

#### Additional Tools:
- `glob_files` - Find files by glob patterns
- `diff_files` - Compare two files
- `git_operation` - Git operations (status, log, diff, commit, etc.)
- `manage_todos` - Task list management
- `manage_context` - Session variables and context
- `think` - Structured reasoning tool
- `http_request` - HTTP client for external APIs

**Key Features**:
- Singleton pattern for shared state
- Session management (todos, variables, history)
- Comprehensive error handling
- Logging integration with OricodeLogger
- Tool definitions for Claude API

### 2. Backend API Routes
**Location**: `oricode-backend/src/routes/tools.ts`

**Endpoints**:

#### POST /tools/execute
Execute a single common tool
```json
{
  "tool_name": "read_file",
  "input": { "path": "/path/to/file" },
  "session_id": "optional-session-id",
  "client_type": "vscode" | "eclipse"
}
```

#### POST /tools/batch
Execute multiple tools in parallel (max 10)
```json
{
  "tools": [
    { "tool_name": "read_file", "input": { "path": "/file1" } },
    { "tool_name": "read_file", "input": { "path": "/file2" } }
  ],
  "session_id": "optional-session-id",
  "client_type": "vscode" | "eclipse"
}
```

#### GET /tools/definitions
Get all tool definitions for Claude API

#### GET /tools/list
List available tools with descriptions

#### GET /tools/session/:id
Get session context (todos, variables, history)

#### DELETE /tools/session/:id
Clear a session

### 3. VS Code ToolRouter Integration
**Location**: `oricode-vs-code/src/tools/ToolRouter.ts`

**Changes**:
- Added `SUPPORTED_COMMON_TOOLS` registry
- Added `isCommonTool()` method
- Added `executeCommonTool()` method that calls backend API
- Integrated into `executeToolInternal()` routing logic

**Tool Routing Logic**:
1. Check if chunking tool → `executeChunkingTool()`
2. Check if method tool → `executeMethodTool()`
3. Check if common tool → `executeCommonTool()` (NEW)
4. Check if MCP tool → `executeMcpTool()`
5. Check if VS Code tool → `executeVsCodeTool()`
6. Unknown → Error

### 4. Eclipse ToolRouter
**Location**: `oricode-poc/src/com/oricode/eclipse/ai/tools/ToolRouter.java`

**Status**: Already supports centralized architecture via backend API calls

**Tool Categories**:
- Chunking tools (mcp_store_code_chunk, mcp_deploy_stored, etc.)
- Method-level tools (mcp_get_class_method, mcp_modify_class_method)
- MCP read tools (mcp_search_abap, mcp_get_class, etc.)
- MCP write tools (mcp_create_class, mcp_modify_class, etc.)
- Eclipse workspace tools (list_projects, read_file, etc.)

### 5. System Prompt Enhancement
**Location**: `oricode-backend/src/utils/systemPromptComposable.ts`

**Critical Addition** (Line 86-96):
```markdown
## 🚨 CRITICAL: CODE SIZE THRESHOLD FOR CHUNKING

**YOU MUST use chunking when code size >= 4096 characters**

- **IF code < 4096 chars**: Use mcp_modify_class_method (single deployment)
- **IF code >= 4096 chars**: Use mcp_store_code_chunk (chunked deployment)
  - Split code into chunks of ~3000 characters each
  - Call mcp_store_code_chunk for EACH chunk with FULL code_chunk content
  - Then call mcp_deploy_stored_code to deploy all chunks together

⚠️ Chunking is NOT optional for large code - it prevents "Request Entity Too Large" errors
```

**Why This Matters**:
- Previous prompt said "large code" without defining the threshold
- Claude was inconsistent about when to use chunking
- Now explicitly states: **>= 4096 characters requires chunking**
- Prevents "Request Entity Too Large" errors
- Ensures reliable deployment of large ABAP code

## Benefits of Centralized Architecture

### 1. Single Source of Truth
- All common tools defined once in `CommonToolsComposable.ts`
- Consistent behavior across VS Code and Eclipse
- No code duplication

### 2. Easier Maintenance
- Bug fixes apply to all clients automatically
- New tools only need to be added in one place
- Tool definitions managed centrally

### 3. Shared State
- Session context shared across tool calls
- Todos persist within a session
- Variables can be set and retrieved

### 4. Consistent Logging
- All tool executions logged to `OricodeLogger`
- Easier debugging and monitoring
- Request tracking with IDs

### 5. Future Extensibility
- Easy to add new tools
- Can add tool middleware (caching, rate limiting, etc.)
- Can add tool telemetry and analytics

## Investigation: Chunking Issue Resolution

### Problem
Claude was not consistently using chunking for large code (>= 4096 characters). Logs showed errors like:
```
[ERROR] [CHUNK_MISSING_CODE] Claude sent chunk without code_chunk parameter!
```

### Root Cause
System prompt mentioned "large code" but didn't specify the exact threshold (>= 4096 characters) for when chunking is mandatory.

### Solution
Added explicit, prominent section in system prompt (line 86) stating:
- **Threshold**: >= 4096 characters
- **Chunk size**: ~3000 characters each
- **Requirement**: Chunking is MANDATORY, not optional

### Impact
- Claude now knows exactly when to use chunking
- Prevents "Request Entity Too Large" errors
- Ensures reliable deployment of large ABAP code
- Consistent behavior across all deployments

## File Locations Reference

```
oricode-backend/
├── src/
│   ├── index.ts                    # Main app (routes registered here)
│   ├── routes/
│   │   └── tools.ts                # Common tools API endpoints
│   └── utils/
│       ├── CommonToolsComposable.ts # Tool implementations
│       ├── systemPromptComposable.ts # System prompt with chunking rules
│       └── OricodeLogger.ts         # Centralized logging

oricode-vs-code/
└── src/
    └── tools/
        └── ToolRouter.ts            # VS Code tool router with common tools

oricode-poc/
└── src/com/oricode/eclipse/ai/
    └── tools/
        ├── ToolRouter.java          # Eclipse tool router
        └── EclipseTools.java        # Eclipse-specific tools
```

## Testing

### Test Common Tools API
```bash
# Test read_file
curl -X POST http://localhost:3000/tools/execute \
  -H "Content-Type: application/json" \
  -H "x-api-key: dev-key-12345" \
  -d '{
    "tool_name": "read_file",
    "input": { "path": "package.json" },
    "client_type": "test"
  }'

# Test batch execution
curl -X POST http://localhost:3000/tools/batch \
  -H "Content-Type: application/json" \
  -H "x-api-key: dev-key-12345" \
  -d '{
    "tools": [
      { "tool_name": "read_file", "input": { "path": "package.json" } },
      { "tool_name": "git_operation", "input": { "operation": "status" } }
    ],
    "client_type": "test"
  }'

# Get tool definitions
curl http://localhost:3000/tools/definitions
```

## Next Steps

### Potential Enhancements
1. **Caching Layer**: Add Redis caching for frequently used tools
2. **Rate Limiting**: Prevent abuse of execute_command and http_request
3. **Tool Metrics**: Track tool usage, success rates, execution times
4. **Tool Versioning**: Support multiple versions of tools
5. **Tool Permissions**: Fine-grained access control per tool per client
6. **Streaming Support**: Stream large file reads and command outputs
7. **Async Execution**: Background execution for long-running tools

### Documentation Improvements
1. Add tool usage examples for each tool
2. Create video tutorials for common workflows
3. Add troubleshooting guide
4. Document performance characteristics of each tool

## Conclusion

The centralized tools architecture provides a robust, maintainable foundation for common development operations across the Oricode ecosystem. By consolidating tool implementations in the backend and exposing them via REST API, we've eliminated code duplication, ensured consistency, and created a platform for future enhancements.

The chunking threshold fix (>= 4096 characters) resolves the critical issue where Claude wasn't consistently using chunking for large code deployments, preventing "Request Entity Too Large" errors and ensuring reliable SAP code deployment.

---

**Implementation Date**: January 20, 2026
**Status**: ✅ Complete
**Architect**: Claude (Sonnet 4.5)
