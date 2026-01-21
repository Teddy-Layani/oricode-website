# Check and Propose Workflow - Oricode AI

## Overview

The **Check and Propose** workflow is Oricode's smart deployment system that validates ABAP code before deployment and provides a user-friendly approval interface.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  CHECK AND PROPOSE WORKFLOW                      │
└─────────────────────────────────────────────────────────────────┘

1. READ          2. CHECK           3. PROPOSE         4. USER REVIEW    5. DEPLOY
   PHASE            PHASE              PHASE              PHASE             PHASE

┌─────────┐    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Claude  │───▶│ Check Syntax │──▶│ Propose Code │──▶│ User Reviews │──▶│ Deploy Code  │
│ Improves│    │ Validate     │   │ (user_       │   │ Accepts/     │   │ (user_       │
│ Code    │    │ Structure    │   │ confirmed=   │   │ Rejects      │   │ confirmed=   │
│         │    │              │   │ false)       │   │              │   │ true)        │
└─────────┘    └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
                     │                   │                   │                   │
                     │ Errors?           │ Shows:            │ Reject?           │ Success?
                     └──▶ Retry          │ - Diff            └──▶ Stop          └──▶ Done
                         (max 3x)        │ - Warnings                                  │
                                        │ - Summary                                   │
                                        └────────────────────────────────────────────▶│
                                                                                       ▼
                                                                              ✅ Code Deployed
```

## Phase 1: Read

**Tools Used**:
- `mcp_get_class` - Get entire class source
- `mcp_get_class_method` - Get specific method source
- `mcp_get_program` - Get program source

**Purpose**: Fetch current code to understand what needs improvement.

## Phase 2: Check (Syntax Validation)

**Tool Used**: `mcp_check_syntax`

**Parameters**:
```json
{
  "object_uri": "/sap/bc/adt/oo/classes/zcl_example/source/main",
  "modified_source": "<full improved code>"
}
```

**Validation Steps**:
1. **Syntax Check**: ABAP syntax validation via ADT
2. **Structure Check**: Verify CLASS DEFINITION + IMPLEMENTATION structure
3. **String Template Check**: Validate balanced pipe characters (|)
4. **ABAP Cloud Check**: Validate allowed APIs for ABAP Cloud

**Error Handling**:
- If errors found: Claude fixes and retries (max 3 attempts)
- If still failing: Stop and explain to user
- **NEVER propose code with syntax errors**

## Phase 3: Propose (user_confirmed=false)

### For Small Code (< 4096 chars)

**Tool Used**: `mcp_modify_class_method`

```json
{
  "class_name": "ZCL_EXAMPLE",
  "method_name": "CALCULATE",
  "new_source": "<improved code>",
  "user_confirmed": false  ← PROPOSE MODE
}
```

### For Large Code (>= 4096 chars)

**Tools Used**:
1. `mcp_store_code_chunk` (multiple times)
2. `mcp_deploy_stored_code`

```json
// Step 1: Store chunks
{
  "chunk_index": 0,
  "total_chunks": 3,
  "code_chunk": "CLASS zcl_example DEFINITION...",
  "session_id": "deploy_X_20250120",
  "user_confirmed": false  ← Not used in chunking, only in deploy
}

// Step 2: Deploy stored code
{
  "session_id": "deploy_X_20250120",
  "user_confirmed": false  ← PROPOSE MODE
}
```

**Response** (user_confirmed=false):
```json
{
  "status": "proposal",
  "message": "Code ready for deployment. Review changes and click 'Accept & Deploy'.",
  "diff": {
    "old_code": "...",
    "new_code": "...",
    "changes_summary": "Modified 1 method, added error handling"
  },
  "warnings": [
    "Method signature changed - check callers",
    "Performance: Consider buffering for large loops"
  ],
  "validation": {
    "syntax": "✓ OK",
    "structure": "✓ OK",
    "abap_cloud_compliant": "✓ OK"
  }
}
```

## Phase 4: User Review

**Eclipse UI Shows**:
- ✅ **Side-by-side diff** (old vs new code)
- ⚠️ **Warnings** (if any)
- 📊 **Change summary** (methods modified, lines changed, etc.)
- 🔍 **Validation results** (syntax, structure, compliance)

**User Actions**:
- **✅ Accept & Deploy** → Proceeds to Phase 5 with `user_confirmed=true`
- **❌ Reject** → Stops deployment, user can request modifications

## Phase 5: Deploy (user_confirmed=true)

### For Small Code

**Tool Used**: `mcp_modify_class_method` (same tool, different flag)

```json
{
  "class_name": "ZCL_EXAMPLE",
  "method_name": "CALCULATE",
  "new_source": "<improved code>",
  "user_confirmed": true  ← DEPLOY MODE
}
```

### For Large Code

**Tool Used**: `mcp_deploy_stored_code`

```json
{
  "session_id": "deploy_X_20250120",
  "user_confirmed": true  ← DEPLOY MODE
}
```

**Deployment Steps** (performed by MCP server):
1. **Lock** object in SAP system
2. **Modify** source code
3. **Syntax Check** (one more time)
4. **Activate** object
5. **Unlock** object
6. **Transport** (if transport request provided)

**Response** (user_confirmed=true):
```json
{
  "status": "success",
  "message": "✓ Code deployed successfully",
  "details": {
    "object": "ZCL_EXAMPLE",
    "type": "class",
    "activated": true,
    "transport": "DEVK900123"
  }
}
```

## Error Handling & Retry

### Syntax Errors (Phase 2)
```
1. Check syntax → Errors found
2. Parse error message
3. Fix code automatically
4. Retry check syntax
5. Repeat up to 3 times
6. If still failing: Stop and explain to user
```

### Deployment Errors (Phase 5)
```
1. Deploy with user_confirmed=true → Fails
2. Check error type:
   - Lock error? → Release lock and retry
   - Syntax error? → Go back to Phase 2
   - Transport error? → Suggest creating transport
   - Authority error? → Explain to user
3. Maximum 3 retry attempts
4. If still failing: Rollback and explain to user
```

## Smart Features

### 1. Automatic Structure Validation

**ABAP Classes**:
```abap
CLASS zcl_example DEFINITION.    ← Must start with this
  " ... class definition ...
ENDCLASS.                         ← Must have this

CLASS zcl_example IMPLEMENTATION. ← Must have this
  " ... method implementations ...
ENDCLASS.                         ← Must end with this
```

If structure is invalid, deployment is rejected **before** proposing to user.

### 2. Diff Generation

Shows exactly what changed:
```diff
- Old line: lv_result = lv_value * 2.
+ New line: lv_result = lv_value * 2 + lv_offset.
```

### 3. Warning Detection

**Automatically warns about**:
- Method signature changes (might break callers)
- Database operations without error handling
- Performance anti-patterns (nested loops, SELECT in loops)
- ABAP Cloud incompatibilities

### 4. Transport Integration

If object requires transport:
```json
{
  "transport_required": true,
  "available_transports": [
    "DEVK900123 - My Feature Branch",
    "DEVK900124 - Bugfix Sprint 42"
  ]
}
```

User can select transport or create new one.

## System Prompt Instructions

**Location**: `oricode-backend/src/utils/systemPromptComposable.ts`

**Key Instructions** (lines 140-161):

```
1. READ - Use mcp_get_class_method to get current method code
2. MODIFY - Generate the improved code internally
3. CHECK SYNTAX - MANDATORY - Call mcp_check_syntax with object URI
4. FIX ERRORS LOOP (up to 3 attempts) - If syntax check returns errors
5. PROPOSE - After syntax check passes:
   - IF code < 4096: Call mcp_modify_class_method(user_confirmed=false)
   - IF code >= 4096: Call mcp_store_code_chunk + mcp_deploy_stored_code(user_confirmed=false)
6. DEPLOY - User clicks "Accept & Deploy", Eclipse calls with user_confirmed=true
```

**Critical Rules**:
- ❌ NEVER call deployment tools without first calling mcp_check_syntax
- ❌ NEVER propose deployment with syntax errors
- ❌ NEVER skip the user_confirmed=false (propose) step
- ✅ ALWAYS explain what you changed and why

## Example Workflows

### Example 1: Simple Method Fix

```
User: "Fix the division by zero bug in calculate method"

1. Claude calls: mcp_get_class_method(class="ZCL_CALC", method="CALCULATE")
   Response: Current code with bug

2. Claude generates fix internally

3. Claude calls: mcp_check_syntax(object_uri="/sap/bc/.../zcl_calc", modified_source="<fixed code>")
   Response: ✓ Syntax OK

4. Claude calls: mcp_modify_class_method(class="ZCL_CALC", method="CALCULATE",
                                         new_source="<fixed code>", user_confirmed=false)
   Response: Proposal with diff shown to user

5. User clicks "Accept & Deploy"

6. Eclipse calls: mcp_modify_class_method(class="ZCL_CALC", method="CALCULATE",
                                         new_source="<fixed code>", user_confirmed=true)
   Response: ✓ Deployed successfully
```

### Example 2: Large Class Refactoring

```
User: "Refactor ZCL_BIG_CLASS to use modern ABAP patterns"

1. Claude calls: mcp_get_class(class_name="ZCL_BIG_CLASS")
   Response: Full class source (10,000 characters)

2. Claude generates refactored code internally

3. Claude calls: mcp_check_syntax(object_uri="/sap/bc/.../zcl_big_class", modified_source="<refactored code>")
   Response: ✓ Syntax OK

4. Claude calculates: code.length = 10,000 >= 4096 → Use chunking

5. Claude calls mcp_store_code_chunk THREE times:
   - chunk_index=0, total_chunks=3, code_chunk="CLASS zcl_big_class DEFINITION..."
   - chunk_index=1, total_chunks=3, code_chunk="  METHOD implementation..."
   - chunk_index=2, total_chunks=3, code_chunk="ENDCLASS."

6. MCP responds: "Ready to deploy. Call DeployStoredCode with session_id='deploy_big_20250120'"

7. Claude calls: mcp_deploy_stored_code(session_id="deploy_big_20250120", user_confirmed=false)
   Response: Proposal with diff shown to user

8. User clicks "Accept & Deploy"

9. Eclipse calls: mcp_deploy_stored_code(session_id="deploy_big_20250120", user_confirmed=true)
   Response: ✓ Deployed successfully (assembled 3 chunks, 10,000 characters)
```

## Benefits

### For Users
- ✅ **Safety**: Never deploy without review
- ✅ **Transparency**: See exactly what will change
- ✅ **Control**: Accept or reject proposals
- ✅ **Warnings**: Get alerted to potential issues
- ✅ **Diff View**: Side-by-side comparison

### For Developers
- ✅ **Validation**: Syntax errors caught before deployment
- ✅ **Rollback**: Easy to reject and try again
- ✅ **Audit Trail**: All changes tracked
- ✅ **Transport Integration**: Seamless change management

### For Operations
- ✅ **Compliance**: All changes require approval
- ✅ **Traceability**: Know who approved what
- ✅ **Governance**: Enforce approval workflows
- ✅ **Quality Gates**: Syntax + structure validation mandatory

## Monitoring & Logs

**Log Locations**:
- `c:/temp/oricode_debug.log` - All tool calls and responses
- `c:/temp/mcp_debug.log` - MCP server communication

**Key Log Markers**:
- `[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk`
- `[MCP] CHUNK_PARAMS: Parsed chunk`
- `[SYNTAX_CHECK] Validation passed`
- `[DEPLOY] user_confirmed=false` (propose)
- `[DEPLOY] user_confirmed=true` (deploy)

---

**Status**: ✅ Fully Implemented and Working
**Last Updated**: January 21, 2026
**Maintainer**: Oricode AI Team
