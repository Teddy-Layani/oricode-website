# Tool Duplication Analysis - VS Code & Eclipse

## Problem Statement

Both VS Code and Eclipse have **local implementations** of common tools like `read_file`, `write_file`, etc., while a centralized **CommonToolsComposable** was created in the backend. This creates:

1. **Code Duplication** - Same tools implemented 3 times (VS Code, Eclipse, Backend)
2. **Inconsistent Behavior** - Each implementation may behave differently
3. **Maintenance Overhead** - Bug fixes need to be applied in 3 places
4. **Incomplete Migration** - Backend has some tools, clients have others

## Current State Analysis

### Backend - CommonToolsComposable (CENTRALIZED)
**Location**: `oricode-backend/src/utils/CommonToolsComposable.ts`

**Tools Implemented**:
- ✅ `read_file` - Read file contents with line ranges
- ✅ `write_file` - Write content to files
- ✅ `str_replace` - Precise string replacement
- ✅ `view_directory` - Directory tree listing
- ✅ `grep_search` - Pattern search across files
- ✅ `execute_command` - Shell command execution
- ✅ `glob_files` - File pattern matching
- ✅ `diff_files` - File comparison
- ✅ `git_operation` - Git operations
- ✅ `manage_todos` - Task list management
- ✅ `manage_context` - Session variables
- ✅ `think` - Structured reasoning
- ✅ `http_request` - HTTP client

**Total**: 13 tools

---

### VS Code - ToolRouter.ts

#### SUPPORTED_VSCODE_TOOLS (LOCAL - lines 181-200)
**Location**: `oricode-vs-code/src/tools/ToolRouter.ts`

**Tools**:
- ❌ `read_file` - **DUPLICATE** (also in backend)
- ❌ `write_file` - **DUPLICATE** (also in backend)
- ⚠️ `create_file` - Should be removed (write_file can create)
- ⚠️ `list_files` - Backend has `view_directory` instead
- ⚠️ `list_directory` - Backend has `view_directory` instead
- ⚠️ `create_folder` - Backend has `write_file` with mkdirp
- ⚠️ `create_directory` - Duplicate of create_folder
- ⚠️ `delete_file` - Not in backend (should add?)
- ⚠️ `mcp__filesystem__*` - MCP-style aliases
- ✓ `add_diagnostic` - VS Code specific (keep local)
- ✓ `add_diagnostics` - VS Code specific (keep local)
- ✓ `clear_diagnostics` - VS Code specific (keep local)
- ✓ `get_diagnostics` - VS Code specific (keep local)

#### SUPPORTED_COMMON_TOOLS (BACKEND - lines 237-250)
**Calls backend API for these tools**:
- ✅ `str_replace` - ✓ Correctly using backend
- ✅ `view_directory` - ✓ Correctly using backend
- ✅ `grep_search` - ✓ Correctly using backend
- ✅ `glob_files` - ✓ Correctly using backend
- ✅ `diff_files` - ✓ Correctly using backend
- ✅ `git_operation` - ✓ Correctly using backend
- ✅ `manage_todos` - ✓ Correctly using backend
- ✅ `manage_context` - ✓ Correctly using backend
- ✅ `think` - ✓ Correctly using backend
- ✅ `http_request` - ✓ Correctly using backend

**Missing from COMMON_TOOLS**:
- ❌ `read_file` - Should be moved from VSCODE_TOOLS to COMMON_TOOLS
- ❌ `write_file` - Should be moved from VSCODE_TOOLS to COMMON_TOOLS
- ❌ `execute_command` - Should be added to COMMON_TOOLS

---

### Eclipse - ToolRouter.java

#### SUPPORTED_ECLIPSE_TOOLS (LOCAL)
**Location**: `oricode-poc/src/com/oricode/eclipse/ai/tools/ToolRouter.java` (lines 82-95)

**Tools**:
- ⚠️ `list_projects` - Eclipse specific (keep local)
- ⚠️ `list_files` - Backend has `view_directory` instead
- ❌ `read_file` - **DUPLICATE** (also in backend)
- ⚠️ `search_code` - Backend has `grep_search` instead
- ✓ `get_problems` - Eclipse specific (keep local)
- ✓ `get_selection` - Eclipse specific (keep local)
- ✓ `get_current_file` - Eclipse specific (keep local)
- ✓ `insert_at_cursor` - Eclipse specific (keep local)
- ✓ `insert_at_cursor_base64` - Eclipse specific (keep local)
- ✓ `replace_selection` - Eclipse specific (keep local)
- ✓ `replace_selection_base64` - Eclipse specific (keep local)
- ⚠️ `create_file` - Should use backend `write_file`
- ⚠️ `create_folder` - Should use backend `write_file` or add to backend

**Missing**: Eclipse doesn't have a COMMON_TOOLS set yet!

---

## Tool Classification

### Category 1: Common Tools (SHOULD BE CENTRALIZED)
These tools work the same across all clients and should use the backend:

| Tool | Backend | VS Code | Eclipse | Status |
|------|---------|---------|---------|--------|
| `read_file` | ✅ | ❌ LOCAL | ❌ LOCAL | **MIGRATE** |
| `write_file` | ✅ | ❌ LOCAL | ❌ MISSING | **MIGRATE** |
| `str_replace` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `view_directory` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `grep_search` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `execute_command` | ✅ | ❌ MISSING | ❌ MISSING | **ADD TO BOTH** |
| `glob_files` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `diff_files` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `git_operation` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `manage_todos` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `manage_context` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `think` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |
| `http_request` | ✅ | ✅ BACKEND | ❌ MISSING | **ADD TO ECLIPSE** |

### Category 2: Client-Specific Tools (KEEP LOCAL)
These tools are specific to each client and should remain local:

**VS Code Specific**:
- `add_diagnostic`, `add_diagnostics`, `clear_diagnostics`, `get_diagnostics`

**Eclipse Specific**:
- `list_projects`, `get_problems`, `get_selection`, `get_current_file`
- `insert_at_cursor`, `replace_selection` (and base64 variants)

### Category 3: Redundant/Deprecated Tools (REMOVE)
These tools are redundant and should be removed:

**VS Code**:
- `create_file` → Use `write_file` instead
- `list_files` → Use `view_directory` instead
- `list_directory` → Use `view_directory` instead
- `create_folder` → Backend `write_file` creates directories automatically
- `create_directory` → Duplicate of `create_folder`
- `mcp__filesystem__*` → Unnecessary aliases

**Eclipse**:
- `list_files` → Use `view_directory` instead
- `search_code` → Use `grep_search` instead
- `create_file` → Use `write_file` instead
- `create_folder` → Backend `write_file` creates directories automatically

---

## Recommended Actions

### Priority 1: Fix Duplicates (Critical)

#### VS Code Changes
**File**: `oricode-vs-code/src/tools/ToolRouter.ts`

1. **Move to COMMON_TOOLS** (line 237):
```typescript
private static readonly SUPPORTED_COMMON_TOOLS = new Set([
    // Core tools (ADD THESE)
    'read_file',           // ← MOVE from VSCODE_TOOLS
    'write_file',          // ← MOVE from VSCODE_TOOLS
    'execute_command',     // ← ADD NEW
    'str_replace',
    'view_directory',
    'grep_search',
    // Additional tools
    'glob_files',
    'diff_files',
    'git_operation',
    'manage_todos',
    'manage_context',
    'think',
    'http_request'
]);
```

2. **Remove from VSCODE_TOOLS** (line 181):
```typescript
private static readonly SUPPORTED_VSCODE_TOOLS = new Set([
    // REMOVE: 'read_file',        → Moved to COMMON_TOOLS
    // REMOVE: 'write_file',        → Moved to COMMON_TOOLS
    // REMOVE: 'create_file',       → Redundant (use write_file)
    // REMOVE: 'list_files',        → Redundant (use view_directory)
    // REMOVE: 'list_directory',    → Redundant (use view_directory)
    // REMOVE: 'create_folder',     → Redundant (write_file creates dirs)
    // REMOVE: 'create_directory',  → Redundant (duplicate)
    // REMOVE: 'delete_file',       → Add to backend first
    // REMOVE: 'mcp__filesystem__*' → Unnecessary aliases

    // KEEP: Diagnostic tools (VS Code specific)
    'add_diagnostic',
    'add_diagnostics',
    'clear_diagnostics',
    'get_diagnostics',
]);
```

#### Eclipse Changes
**File**: `oricode-poc/src/com/oricode/eclipse/ai/tools/ToolRouter.java`

1. **Add SUPPORTED_COMMON_TOOLS** (after line 36):
```java
private static final Set<String> SUPPORTED_COMMON_TOOLS = new HashSet<>();

static {
    // ... existing tool sets ...

    // Common tools (centralized in backend)
    SUPPORTED_COMMON_TOOLS.add("read_file");
    SUPPORTED_COMMON_TOOLS.add("write_file");
    SUPPORTED_COMMON_TOOLS.add("str_replace");
    SUPPORTED_COMMON_TOOLS.add("view_directory");
    SUPPORTED_COMMON_TOOLS.add("grep_search");
    SUPPORTED_COMMON_TOOLS.add("execute_command");
    SUPPORTED_COMMON_TOOLS.add("glob_files");
    SUPPORTED_COMMON_TOOLS.add("diff_files");
    SUPPORTED_COMMON_TOOLS.add("git_operation");
    SUPPORTED_COMMON_TOOLS.add("manage_todos");
    SUPPORTED_COMMON_TOOLS.add("manage_context");
    SUPPORTED_COMMON_TOOLS.add("think");
    SUPPORTED_COMMON_TOOLS.add("http_request");
}
```

2. **Remove from ECLIPSE_TOOLS**:
```java
// REMOVE: SUPPORTED_ECLIPSE_TOOLS.add("read_file");        → Moved to COMMON
// REMOVE: SUPPORTED_ECLIPSE_TOOLS.add("list_files");       → Use view_directory
// REMOVE: SUPPORTED_ECLIPSE_TOOLS.add("search_code");      → Use grep_search
// REMOVE: SUPPORTED_ECLIPSE_TOOLS.add("create_file");      → Use write_file
// REMOVE: SUPPORTED_ECLIPSE_TOOLS.add("create_folder");    → Use write_file

// KEEP: Eclipse-specific tools
SUPPORTED_ECLIPSE_TOOLS.add("list_projects");
SUPPORTED_ECLIPSE_TOOLS.add("get_problems");
SUPPORTED_ECLIPSE_TOOLS.add("get_selection");
SUPPORTED_ECLIPSE_TOOLS.add("get_current_file");
SUPPORTED_ECLIPSE_TOOLS.add("insert_at_cursor");
SUPPORTED_ECLIPSE_TOOLS.add("insert_at_cursor_base64");
SUPPORTED_ECLIPSE_TOOLS.add("replace_selection");
SUPPORTED_ECLIPSE_TOOLS.add("replace_selection_base64");
```

3. **Add helper methods**:
```java
public static boolean isCommonTool(String toolName) {
    return toolName != null && SUPPORTED_COMMON_TOOLS.contains(toolName);
}
```

4. **Add routing in executeToolInternal()**:
```java
// === COMMON TOOLS (via backend API) ===
if (isCommonTool(toolName)) {
    logger.logMcp("COMMON_TOOL_ROUTED", toolName, null);
    return executeCommonTool(toolName, toolInput);
}
```

5. **Implement executeCommonTool()** - Call backend `/api/tools/execute`

---

### Priority 2: Add Missing Backend Tool

**File**: `oricode-backend/src/utils/CommonToolsComposable.ts`

Add `delete_file` tool (currently missing):
```typescript
{
    name: 'delete_file',
    description: 'Delete a file from the filesystem.',
    input_schema: {
        type: 'object',
        properties: {
            path: { type: 'string', description: 'Path to the file to delete' }
        },
        required: ['path']
    }
}
```

---

## Expected Outcome

After completing these changes:

### Backend (Single Source of Truth)
- **14 Common Tools** centralized in `CommonToolsComposable.ts`
- All common operations go through backend API

### VS Code
- **4 VS Code-specific tools** (diagnostics)
- **14 Common tools** via backend API
- **Total**: 18 tools (down from 30+)

### Eclipse
- **8 Eclipse-specific tools** (projects, selection, cursor, etc.)
- **14 Common tools** via backend API
- **Total**: 22 tools (down from 14 local)

### Benefits
1. ✅ **No duplication** - Common tools implemented once
2. ✅ **Consistent behavior** - Same tool, same result across clients
3. ✅ **Easier maintenance** - Fix bugs in one place
4. ✅ **Shared state** - Todos/context work across clients
5. ✅ **Cleaner code** - Each client only implements client-specific tools

---

## Migration Checklist

- [ ] **VS Code**
  - [ ] Move `read_file`, `write_file`, `execute_command` to COMMON_TOOLS
  - [ ] Remove redundant tools from VSCODE_TOOLS
  - [ ] Test all common tools via backend API

- [ ] **Eclipse**
  - [ ] Add SUPPORTED_COMMON_TOOLS set
  - [ ] Add `isCommonTool()` helper method
  - [ ] Add routing for common tools in executeToolInternal()
  - [ ] Implement `executeCommonTool()` method
  - [ ] Remove redundant tools from ECLIPSE_TOOLS
  - [ ] Test all common tools via backend API

- [ ] **Backend**
  - [ ] Add `delete_file` tool to CommonToolsComposable
  - [ ] Update tool definitions API response

- [ ] **Documentation**
  - [ ] Update tool documentation with centralization info
  - [ ] Add migration guide for users
  - [ ] Update architecture diagrams

---

**Status**: Analysis Complete - Ready for Implementation
**Date**: January 20, 2026
**Analyst**: Claude (Sonnet 4.5)
