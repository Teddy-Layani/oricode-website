# Oricode Central Bug Reference

> **For AI Assistants:** When debugging Oricode issues, scan this document first to check if the bug has already been documented and fixed.

## Quick Reference - Key Files

| Component | File Path | Common Issues |
|-----------|-----------|---------------|
| **MCP Server (SOURCE OF TRUTH)** | `mcp-abap-adt-crud/src/index.ts` | Tool definitions, DDL generation |
| **Eclipse - DDIC Tools** | `oricode-poc/.../tools/EclipseTools.java` | Tool schemas for Claude |
| **Eclipse - Tool Input Forwarding** | `oricode-poc/.../api/OricodeApiClient.java` | Parameter extraction & forwarding |
| **Eclipse - MCP Client** | `oricode-poc/.../mcp/McpAbapClient.java` | MCP communication |
| **VS Code - Tool Router** | `oricode-vs-code/src/tools/ToolRouter.ts` | VS Code tool definitions |
| **Backend - System Prompt** | `oricode-backend/src/utils/systemPromptComposable.ts` | AI behavior instructions |
| **Backend - Chat API** | `oricode-backend/src/routes/chat.ts` | API timeouts, streaming |

---

## Bug Index

| # | Bug Name | Status | Documentation |
|---|----------|--------|---------------|
| 1 | Table Creation - Fields Not Forwarded | FIXED | [TABLE_CREATION_FIELDS_MISSING_BUG.md](TABLE_CREATION_FIELDS_MISSING_BUG.md) |
| 2 | Table Creation - Integer Parse Error | FIXED | This document |
| 3 | Chunking - Orange UI Still Appearing | FIXED | [CHUNKING_FIX_2026-01-21.md](../CHUNKING_FIX_2026-01-21.md) |
| 4 | Chunking - Missing ENDCLASS | FIXED | [README_CHUNKING_FIXES.md](../README_CHUNKING_FIXES.md) |
| 5 | Timeout Errors | FIXED | [TIMEOUT_FIX_SUMMARY.md](../TIMEOUT_FIX_SUMMARY.md) |
| 6 | Local Class - include_type Missing | FIXED | [LOCAL_CLASS_DEPLOYMENT_FIX_2026-01-21.md](../LOCAL_CLASS_DEPLOYMENT_FIX_2026-01-21.md) |
| 7 | System Prompt Bloat | FIXED | [STABILITY_ISSUE_2026-01-21.md](../STABILITY_ISSUE_2026-01-21.md) |
| 8 | File-Based Chunking | IMPLEMENTED | [FILE_BASED_CHUNKING_IMPLEMENTATION_COMPLETED.md](../FILE_BASED_CHUNKING_IMPLEMENTATION_COMPLETED.md) |

---

## Bug Details

### 1. Table Creation - Fields Array Not Forwarded to MCP

**Error:** `fields array is required and must not be empty`

**Root Cause:** OricodeApiClient.java was extracting only 4 parameters (table_name, description, package, transport) and NOT extracting the `fields` array before forwarding to MCP server.

**Fix Location:** `oricode-poc/.../api/OricodeApiClient.java`
- Added `parseFieldsArray()` method
- Added `findMatchingBracket()` method
- Modified `mcp_create_table` case to extract and forward `fields` array

**Related Docs:** [TABLE_CREATION_FIELDS_MISSING_BUG.md](TABLE_CREATION_FIELDS_MISSING_BUG.md)

---

### 2. Table Creation - NumberFormatException on Empty String

**Error:** `For input string: ''`

**Root Cause:** When extracting `length` or `decimals` from JSON, `extractJsonNumber()` could return an empty string, and `Integer.parseInt("")` throws NumberFormatException.

**Fix Location:** `oricode-poc/.../api/OricodeApiClient.java`
- Added empty string check before `Integer.parseInt()`
- Added try-catch blocks around all integer parsing
- Fixed in `parseFieldsArray()`, `mcp_create_data_element`, `mcp_create_domain`

**Code Pattern:**
```java
// BEFORE (broken)
if (lengthStr != null) field.put("length", Integer.parseInt(lengthStr));

// AFTER (fixed)
if (lengthStr != null && !lengthStr.isEmpty()) {
    try { field.put("length", Integer.parseInt(lengthStr)); }
    catch (NumberFormatException e) { /* ignore */ }
}
```

---

### 3. Chunking Workflow - Old Orange UI Appearing

**Error:** Orange confirmation UI appeared instead of pill buttons

**Root Cause:** Eclipse ToolRouter.java intercepted `mcp_deploy_stored_code(user_confirmed=false)` BEFORE it reached Claude.

**Fix Location:** `oricode-poc/.../tools/ToolRouter.java`
- Removed interception code (lines 328-365)

**Commit:** `bd63978`

---

### 4. Chunking - Missing Final ENDCLASS

**Error:** Code structure validation failed

**Root Cause:** AI split code at exact character boundaries without respecting ABAP structure.

**Fix Location:** `oricode-backend/src/utils/systemPromptComposable.ts`
- Added explicit instructions to split at safe boundaries (after ENDMETHOD)
- Ensure final chunk includes ENDCLASS

**Commit:** `09c9db4`

---

### 5. Timeout Errors - Read Timed Out

**Error:** `java.net.SocketException: Read timed out`

**Root Cause:** Railway timeout (120s) vs Anthropic SDK timeout (10min) mismatch.

**Fix Location:** `oricode-backend/src/routes/chat.ts`
- Set Anthropic SDK timeout to 5 minutes (300,000ms)
- Added `maxRetries: 2`

**Commit:** `d011620`

---

### 6. Local Class Deployment - Expected Main Class

**Error:** `Expected main class source but got local implementations`

**Root Cause:** `include_type="implementations"` parameter not passed in chunking workflow.

**Fix Location:** `oricode-backend/src/utils/systemPromptComposable.ts`
- Documented that `include_type` is REQUIRED for local class deployment

**Commit:** `54e6713`

---

### 7. System Prompt Bloat

**Error:** Timeouts and max_tokens errors

**Root Cause:** System prompt grew to 35,645 chars (8,900 tokens).

**Fix Location:** `oricode-backend/src/utils/systemPromptComposable.ts`
- Reduced to 5,158 chars (86% reduction)

**Commit:** `498d07e`

---

## Architecture Issues (Design Flaws)

### Tool Definition Duplication

**Problem:** Tool definitions are hardcoded in 4+ places instead of centralized:
1. MCP Server (source of truth)
2. Eclipse EclipseTools.java
3. Eclipse McpAbapClient.java
4. VS Code ToolRouter.ts

**Solution:** Centralize in backend `/api/tools/definitions` endpoint.

**Related Docs:** [TABLE_CREATION_FIELDS_MISSING_BUG.md](TABLE_CREATION_FIELDS_MISSING_BUG.md)

---

## AI Assistant Commands

When debugging, use these grep commands:

```bash
# Find tool definitions across codebase
grep -rn "mcp_create_table\|CreateTable" --include="*.ts" --include="*.java"

# Check Eclipse tool input forwarding
grep -n "case \"mcp_" oricode-poc/.../api/OricodeApiClient.java

# Check MCP server tool handlers
grep -n "case \"Create" mcp-abap-adt-crud/src/index.ts

# Find debug logs
Get-Content 'C:\temp\oricode_debug.log' -Tail 100 | Select-String "Error|fields"
```

---

## Debugging Checklist

When a tool fails:

1. **Check logs:** `C:\temp\oricode_debug.log`
2. **Verify tool.input:** Is the parameter in the raw JSON from Claude?
3. **Check extraction:** Is OricodeApiClient.java extracting the parameter?
4. **Check forwarding:** Is the parameter being passed to MCP?
5. **Check MCP handling:** Is the MCP server receiving it correctly?
6. **Check SAP response:** Is SAP returning an error?

---

## Recent Fixes (2026-02-02)

| Time | Fix | File |
|------|-----|------|
| 09:30 | Added `parseFieldsArray()` for table creation | OricodeApiClient.java |
| 10:00 | Fixed NumberFormatException on empty strings | OricodeApiClient.java |
| 10:00 | Added debug logging for fields extraction | OricodeApiClient.java |

---

*Last Updated: 2026-02-02*
