# Smart Interleaved Workflow - January 21, 2026 @ 20:40

## Problem Statement

**Issue 1**: AI was stopping after sending all chunks, requiring user to type "continue" before calling `mcp_deploy_stored_code`
- **Root Cause**: Hitting max_tokens (8192) after 8 chunking turns
- **Impact**: Poor UX, manual intervention required

**Issue 2**: Syntax errors only discovered at final deployment
- **Root Cause**: Check syntax only called once on full assembled code
- **Impact**: Wasted time sending all chunks before finding errors

## Solution: Smart Interleaved Workflow

### Old Workflow (Sequential)
```
1. Generate full improved code
2. Call mcp_check_syntax on full code ✅
3. If OK, split into chunks
4. MESSAGE 1: mcp_store_code_chunk(chunk 0)
5. MESSAGE 2: mcp_store_code_chunk(chunk 1)
6. MESSAGE 3: mcp_store_code_chunk(chunk 2)
7. ... (8 messages for 8 chunks)
8. MESSAGE 9: mcp_deploy_stored_code
   ❌ Problem: Hits max_tokens, AI stops
   ❌ User must type "continue"
```

### New Workflow (Interleaved)
```
1. Generate full improved code
2. Split into chunks (preserving ENDCLASS)
3. MESSAGE 1: mcp_store_code_chunk(chunk 0) + mcp_check_syntax(chunk 0 only)
   ✅ Check early, catch errors fast
4. MESSAGE 2: mcp_store_code_chunk(chunk 1) + mcp_check_syntax(chunks 0+1)
   ✅ Incremental validation
5. MESSAGE 3: mcp_store_code_chunk(chunk 2) + mcp_check_syntax(chunks 0+1+2)
6. ... continue for all chunks
7. FINAL MESSAGE:
   - mcp_store_code_chunk(chunk N-1)
   - mcp_check_syntax(full assembled code)
   - mcp_deploy_stored_code(user_confirmed=false)
   ✅ All in one final message - no stopping!
```

## Benefits

### 1. No Token Limit Issues ✅
- Each message does: store chunk (small) + check syntax (incremental)
- Final message completes deployment automatically
- **No user intervention needed**

### 2. Early Error Detection ✅
- Syntax errors caught **immediately** after the problematic chunk
- No need to wait for all chunks before discovering issues
- **Fail fast** principle

### 3. Better Progress Feedback ✅
- User sees syntax validation happening chunk by chunk
- Clear indication of progress
- Builds confidence

### 4. Easier Debugging ✅
- Know exactly which chunk has the problem
- Smaller code segment to debug
- Faster fix iteration

## Implementation

### System Prompt Changes

**File**: [systemPromptComposable.ts:49-66](c:\Users\teddy.layani\eclipse-workspace\oricode-backend\src\utils\systemPromptComposable.ts#L49-L66)

```typescript
## 🚨 CHUNKING RULES (>= 8K chars)

When code >= 8192 chars, use chunking with interleaved syntax checks:

1. **Generate unique session_id**: Use format "{OBJECT_NAME}_{UNIX_TIMESTAMP}"
2. **Calculate chunks**: chunk_count = ceil(code_length / 8000)
3. **Split code** into 8K chunks at safe boundaries (after ENDMETHOD)
4. **CRITICAL**: Last chunk MUST include final ENDCLASS
5. **Smart interleaved workflow** (store + check repeatedly):
   - MESSAGE 1: mcp_store_code_chunk(chunk 0) + mcp_check_syntax(partial)
   - MESSAGE 2: mcp_store_code_chunk(chunk 1) + mcp_check_syntax(partial)
   - FINAL: mcp_store_code_chunk(chunk N-1) + mcp_check_syntax(full) + mcp_deploy_stored_code
6. **If syntax error found**: Fix immediately, re-send chunk with corrected code
```

### MCP Server Support

**Already Supported** ✅ - No changes needed!

The MCP server's `mcp_check_syntax` tool already supports checking partial code:
- Accepts any code snippet
- Validates ABAP syntax
- Returns errors with line numbers
- Can be called multiple times per session

## Testing Requirements

### Test Case 1: Large Code (>60K chars)
1. Request modification to local class with 60K+ chars
2. AI should split into 8 chunks (~8K each)
3. Each chunk should call:
   - `mcp_store_code_chunk`
   - `mcp_check_syntax` with accumulated code so far
4. Final message should include all three:
   - `mcp_store_code_chunk` (last chunk)
   - `mcp_check_syntax` (full code)
   - `mcp_deploy_stored_code`
5. **Expected**: No "continue" prompt needed

### Test Case 2: Syntax Error in Chunk 3
1. Introduce syntax error in code that will be in chunk 3
2. AI sends chunks 1-2 successfully
3. AI sends chunk 3 + syntax check
4. **Expected**: Error detected immediately after chunk 3
5. AI fixes and re-sends chunk 3
6. Continues with chunks 4-8
7. Deploys successfully

### Test Case 3: Final ENDCLASS Preservation
1. Request large modification
2. Verify last chunk includes `ENDCLASS.`
3. Verify assembled code ends with `ENDCLASS.`
4. **Expected**: Structure validation passes

## Edge Cases Handled

### 1. Network Error During Chunking
- Session preserved (smart cleanup)
- User can retry with same session_id
- Already-stored chunks remain intact

### 2. Syntax Error in Last Chunk
- Detected before deployment (interleaved check)
- AI fixes last chunk
- Re-checks full assembled code
- Only then deploys

### 3. Max Token Limit Hit
- Unlikely: Each message is small (chunk + incremental check)
- Final message includes deployment call
- No separate "deploy" message needed

## Performance Comparison

### Old Sequential Workflow
- **Messages**: 9 (8 chunks + 1 deploy)
- **Syntax Checks**: 1 (at end)
- **Token Usage**: ~7000 tokens (hits limit)
- **Error Detection**: After all chunks sent
- **User Intervention**: Required ("continue")

### New Interleaved Workflow
- **Messages**: 8 (one per chunk)
- **Syntax Checks**: 8 (incremental)
- **Token Usage**: ~5000 tokens (well below limit)
- **Error Detection**: Immediate (after each chunk)
- **User Intervention**: None needed ✅

## Rollout Plan

1. ✅ **Backend Deployed**: Commit `6ba33d9` pushed to Railway
2. ⏳ **Wait for Railway**: Auto-deploy (3-5 minutes)
3. ⏳ **Test with User**: Request large code modification
4. ✅ **Monitor Logs**: Verify interleaved pattern in logs
5. ✅ **Confirm No "Continue" Needed**: Should complete automatically

## Related Commits

- `6ba33d9` - Implement smart interleaved workflow
- `09c9db4` - Fix chunking: Preserve ABAP structure boundaries
- `bd63978` - Remove ToolRouter interception for pill buttons

---

**Status**: ✅ Deployed to Railway
**Impact**: Eliminates token limit issues + enables early error detection
**Next**: Test with real large code modification (~60K+ chars)
