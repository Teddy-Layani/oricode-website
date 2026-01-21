# Check and Propose Workflow Investigation

## Investigation Date
January 21, 2026

## Problem Statement
User reported: "but it has not been used in last test"

The check-and-propose workflow exists and is documented in [CHECK_AND_PROPOSE_WORKFLOW.md](CHECK_AND_PROPOSE_WORKFLOW.md), but recent Eclipse tests show it's not being triggered in practice.

## Root Cause Analysis

### What We Found in the Logs

Analyzed logs from `c:/temp/oricode_debug.log` for the most recent test session on **January 21, 2026 at 08:09-08:13**.

**Timeline of Events:**
```
08:09:28 - MCP server connected, CheckSyntax tool available
08:12:19 - mcp_store_code_chunk (chunk 0/3) - NO mcp_check_syntax before this!
08:12:54 - mcp_store_code_chunk (chunk 1/3)
08:13:09 - mcp_store_code_chunk (chunk 2/3)
08:13:14 - Chunks assembled: 3 chunks, 12834 characters
08:13:25 - mcp_deploy_stored (user_confirmed=false)
08:13:46 - User clicked "Accept & Deploy"
08:13:46 - mcp_deploy_stored (user_confirmed=true)
08:13:46 - ❌ DEPLOYMENT FAILED: Missing CLASS DEFINITION section
```

**Critical Finding:**
❌ **NO `mcp_check_syntax` call before chunking workflow started**

The last `mcp_check_syntax` in the logs was on **January 20 at 22:40**, but the test on **January 21 at 08:13** shows zero syntax checks.

### Why This Breaks the Workflow

The check-and-propose workflow has 5 phases:
1. READ - Get current code
2. **CHECK** - Validate syntax ← **THIS WAS SKIPPED**
3. PROPOSE - Show code for review
4. USER REVIEW - Accept or reject
5. DEPLOY - Deploy approved code

When Phase 2 (CHECK) is skipped:
- Code is never validated before proposal
- Structure errors aren't caught early
- User sees a proposal that will fail on deployment
- Deployment fails with validation errors that should have been caught in CHECK phase

### Evidence from Recent Test

**What Happened:**
1. ✅ Chunking workflow executed correctly (3 chunks, 12834 chars)
2. ✅ user_confirmed=false proposal shown to user
3. ✅ User clicked "Accept & Deploy"
4. ❌ Deployment failed: "Missing CLASS DEFINITION section"

**What Should Have Happened:**
1. 🚨 **mcp_check_syntax called FIRST with full improved code**
2. 🚨 **Structure validation fails in CHECK phase**
3. 🚨 **Claude fixes the structure error**
4. 🚨 **mcp_check_syntax called again**
5. ✅ Syntax OK, proceed to chunking
6. ✅ Chunking workflow executes
7. ✅ Proposal shown to user
8. ✅ Deployment succeeds

## System Prompt Analysis

### Current State (Before Fix)
**File**: `oricode-backend/src/utils/systemPromptComposable.ts`

**Line 142-146**: Mentions CHECK SYNTAX as step 3, says "MANDATORY STEP", but:
- Instructions are buried in middle of large workflow section
- No visual emphasis to make it stand out
- No explicit examples showing syntax check BEFORE chunking
- Claude can overlook it when rushing to start chunking

### The Fix Applied

**Added at line 120**: New section with maximum visibility

```markdown
## 🚨🚨🚨 ABSOLUTE REQUIREMENT: CHECK SYNTAX FIRST 🚨🚨🚨

**STOP! READ THIS BEFORE ANY CODE DEPLOYMENT:**

Before calling ANY deployment tool (mcp_modify_class_method, mcp_store_code_chunk), you MUST:

1️⃣ Call mcp_check_syntax FIRST with your improved code
2️⃣ Wait for the response
3️⃣ If errors: Fix them and call mcp_check_syntax again (max 3 attempts)
4️⃣ Only after ✓ NO ERRORS: Proceed to deployment

**YOU ARE NOT ALLOWED TO:**
❌ Skip mcp_check_syntax and go straight to deployment
❌ Call mcp_store_code_chunk before calling mcp_check_syntax
❌ Call mcp_modify_class_method before calling mcp_check_syntax
❌ Assume code is correct without validation

**Example of CORRECT workflow:**
- Step 1: mcp_get_class_method to read current code
- Step 2: Generate improved code internally
- Step 3: mcp_check_syntax with object_uri and modified_source MANDATORY
- Step 4: If syntax OK, count characters (e.g., 12834 chars)
- Step 5: 12834 >= 4096 so use chunking
- Step 6: mcp_store_code_chunk for chunk 0 of 3
- Step 7: mcp_store_code_chunk for chunk 1 of 3
- Step 8: mcp_store_code_chunk for chunk 2 of 3
- Step 9: mcp_deploy_stored_code with user_confirmed=false

**Example of WRONG workflow (DO NOT DO THIS):**
- Step 1: mcp_get_class_method to read current code
- Step 2: Generate improved code internally
- Step 3: Count characters 12834
- Step 4: mcp_store_code_chunk chunk 0 ← ❌ WRONG! No syntax check first!
```

## Why This Fix Should Work

### 1. Visual Prominence
- 🚨🚨🚨 emoji alerts at section start
- "STOP! READ THIS" command
- "ABSOLUTE REQUIREMENT" in title
- Placed BEFORE the detailed workflow section

### 2. Explicit Prohibitions
- Four specific ❌ "NOT ALLOWED" rules
- Each one directly addresses the skip behavior
- Clear consequences stated

### 3. Side-by-Side Examples
- CORRECT workflow shows exact sequence
- WRONG workflow shows what NOT to do
- Contrasting examples make the difference obvious

### 4. Numbered Steps with Emojis
- 1️⃣ 2️⃣ 3️⃣ 4️⃣ makes sequence unmissable
- Visual anchors for each step
- Impossible to skim past

## Expected Outcome

After this fix, Claude should:
1. ✅ Always call `mcp_check_syntax` BEFORE any deployment tool
2. ✅ Catch structure errors in CHECK phase (not DEPLOY phase)
3. ✅ Fix errors and retry syntax check before proposing
4. ✅ Only propose code that has passed validation
5. ✅ Reduce deployment failures caused by structure errors

## Testing Plan

1. **Test Small Code (< 4096 chars)**
   - Verify mcp_check_syntax called before mcp_modify_class_method
   - Check logs for sequence: CHECK → PROPOSE → DEPLOY

2. **Test Large Code (>= 4096 chars)**
   - Verify mcp_check_syntax called before mcp_store_code_chunk
   - Check logs for sequence: CHECK → CHUNK → CHUNK → CHUNK → PROPOSE → DEPLOY

3. **Test Code with Structure Errors**
   - Verify structure errors caught in CHECK phase
   - Verify Claude fixes and retries
   - Verify proposal only shown after fix

4. **Test Code with Syntax Errors**
   - Verify syntax errors caught in CHECK phase
   - Verify up to 3 retry attempts
   - Verify clear explanation if still failing

## Log Markers to Watch

After fix, logs should show:
```
[MCP] TOOL_ROUTED: mcp_check_syntax          ← Should appear FIRST
[MCP] Syntax check: ✓ OK
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk ← Only after syntax OK
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk
[MCP] CHUNK_TOOL_ROUTED: mcp_store_code_chunk
[MCP] Chunks assembled: 3 chunks, 12834 characters
[MCP] TOOL_ROUTED: mcp_deploy_stored
[MCP] user_confirmed=false (propose)
[User clicks Accept]
[MCP] user_confirmed=true (deploy)
[MCP] ✓ Deployment successful
```

## Related Files

- [CHECK_AND_PROPOSE_WORKFLOW.md](CHECK_AND_PROPOSE_WORKFLOW.md) - Complete workflow documentation
- [systemPromptComposable.ts:120-156](../oricode-backend/src/utils/systemPromptComposable.ts) - Fix location
- Logs: `c:/temp/oricode_debug.log`

## Conclusion

The check-and-propose workflow was being bypassed because Claude was skipping the `mcp_check_syntax` step. By adding a highly visible, explicit requirement section at the top of the deployment instructions with clear examples and prohibitions, we ensure that syntax validation happens BEFORE any deployment attempt.

This fix transforms the workflow from:
- ❌ CHUNK → PROPOSE → DEPLOY (fails) → User sees error

To:
- ✅ CHECK → Fix if needed → CHUNK → PROPOSE → DEPLOY (succeeds)

---

**Status**: Fix Applied
**Commit**: Pending
**Next**: Test in Eclipse with large ABAP code deployment
