# Verification Checklist - Post-Deployment Testing

## Railway Backend Verification

### 1. Check Deployment Status
```bash
# Visit Railway dashboard
# Verify latest commit: d011620
# Check deployment logs for errors
```

**Expected**: ✅ Deployment successful, no errors

---

### 2. Test Backend Health Endpoint
```bash
curl https://your-railway-url.railway.app/health
```

**Expected**: `{"status": "ok"}`

---

## Functional Testing

### Test 1: Small Method Modification (< 8K chars)

**Steps**:
1. Open Eclipse
2. Request modification to small method (e.g., "Add logging to calculate_discount method")
3. Observe workflow

**Expected Behavior**:
- ✅ AI calls `mcp_get_class_method`
- ✅ AI calls `mcp_check_syntax` BEFORE deployment
- ✅ AI calls `mcp_modify_class_method(user_confirmed=false)`
- ✅ Pill button appears: "Accept" / "Reject"
- ✅ No timeout errors
- ✅ Response time < 30 seconds

---

### Test 2: Local Class Modification (60K+ chars, requires chunking)

**Steps**:
1. Open Eclipse
2. Request modification to large local class (e.g., ZCL_BM_LEARNING)
3. Watch chunking workflow

**Expected Behavior**:
- ✅ AI calls `mcp_get_class_locals`
- ✅ AI detects code >= 8K chars
- ✅ AI generates unique session_id: `ZCL_NAME_1737XXXXXX`
- ✅ AI calculates chunks: e.g., 8 chunks for 60K chars
- ✅ **Smart Interleaved Workflow**:
  - MESSAGE 1: `mcp_store_code_chunk(chunk 0)` + `mcp_check_syntax(partial)`
  - MESSAGE 2: `mcp_store_code_chunk(chunk 1)` + `mcp_check_syntax(partial)`
  - MESSAGE 3-7: Continue pattern
  - MESSAGE 8: `mcp_store_code_chunk(chunk 7)` + `mcp_check_syntax(full)` + `mcp_deploy_stored_code`
- ✅ Last chunk includes `ENDCLASS.`
- ✅ No timeout errors
- ✅ No "continue" prompt needed
- ✅ Total time < 5 minutes
- ✅ Pill button appears for final approval

---

### Test 3: Syntax Error Handling

**Steps**:
1. Request modification that will introduce syntax error
2. Observe AI behavior

**Expected Behavior**:
- ✅ Syntax error detected after chunk N
- ✅ AI analyzes error message
- ✅ AI fixes the problematic chunk
- ✅ AI re-sends corrected chunk
- ✅ Continues with remaining chunks
- ✅ Deployment succeeds after fix

---

### Test 4: Network Timeout Resilience

**Steps**:
1. Request large code modification
2. Simulate network hiccup (optional)
3. Observe behavior

**Expected Behavior**:
- ✅ Request completes within 5-minute timeout
- ✅ If transient failure occurs, auto-retries (maxRetries=2)
- ✅ Session preserved on network errors
- ✅ User can retry if needed
- ✅ <3% timeout rate overall

---

### Test 5: Pill Button Rendering

**Steps**:
1. Request any modification
2. Wait for deployment proposal
3. Check pill button UI

**Expected Behavior**:
- ✅ Pill buttons render correctly in Eclipse chat
- ✅ "Accept" and "Reject" options visible
- ✅ Clicking "Accept" deploys code
- ✅ Clicking "Reject" cancels deployment
- ✅ **After Eclipse rebuild**: No old orange "CHUNKED DEPLOYMENT PROPOSAL" box

---

## Log Verification

### Railway Logs to Check

**Look for these patterns**:

1. **System Prompt Size** (should be ~2,163 tokens):
```
[API] getSystemPrompt called: clientType=eclipse, size=8651 chars
```

2. **Anthropic SDK Timeout Config**:
```
Anthropic client initialized: timeout=300000ms, maxRetries=2
```

3. **Smart Interleaved Workflow**:
```
Tool: mcp_store_code_chunk (chunk_index=0)
Tool: mcp_check_syntax (partial)
Tool: mcp_store_code_chunk (chunk_index=1)
Tool: mcp_check_syntax (partial)
...
Tool: mcp_deploy_stored_code (user_confirmed=false)
```

4. **No Timeout Errors**:
```
❌ Should NOT see: "Read timed out"
❌ Should NOT see: "ETIMEDOUT"
✅ Should see: "Request completed successfully"
```

5. **Session Management**:
```
Session created: ZCL_BM_LEARNING_1737XXXXXX
Total chunks: 8
Last chunk includes ENDCLASS: ✅
```

---

## Performance Metrics

### Token Usage

**Before Optimizations**:
- System Prompt: ~8,900 tokens
- User Code (60K chars): ~15,000 tokens
- **Total Input**: ~23,900 tokens

**After Optimizations**:
- System Prompt: ~2,163 tokens (76% reduction)
- User Code (60K chars): ~15,000 tokens
- **Total Input**: ~17,163 tokens
- **Savings**: 6,737 tokens per request

### Response Times

**Target Metrics**:
| Operation | Target | Current |
|-----------|--------|---------|
| Small method (< 8K) | < 30 sec | ⏳ Test |
| Large locals (60K, 8 chunks) | < 3 min | ⏳ Test |
| Timeout rate | < 3% | ⏳ Measure |

---

## Rollback Plan (If Issues Found)

### If Timeouts Still Occur

**Option 1**: Increase timeout further
```typescript
timeout: 420000,  // 7 minutes (instead of 5)
```

**Option 2**: Implement streaming with heartbeat (from STREAMING_HEARTBEAT_IMPLEMENTATION.md)

### If System Prompt Too Aggressive

**Revert commits**:
```bash
cd ../oricode-backend
git revert 498d07e  # Revert system prompt simplification
git revert 72f65b0  # Revert LOCAL CLASS condensing
git push
```

### If Pill Buttons Not Working

**Check XML format** in Eclipse logs:
- Should see: `<pill-buttons header="...">`
- Should see: `<option value="1">...</option>`
- Should NOT see: `<pill-button-header>` or `<pill-button-option>`

**If still broken**, rebuild Eclipse plugin with latest ToolRouter.java

---

## Success Criteria

### ✅ Ready for Production

All of these must pass:

- [ ] Railway deployment successful (commit d011620)
- [ ] System prompt size = 8,651 chars
- [ ] Small method modification works (< 30 sec)
- [ ] Large code chunking works (< 3 min, no timeout)
- [ ] Smart interleaved workflow executes (store+check repeatedly)
- [ ] Last chunk includes ENDCLASS
- [ ] No "continue" prompts needed
- [ ] Pill buttons render correctly
- [ ] Syntax errors caught and fixed automatically
- [ ] Timeout rate < 3%

### 🎯 Next Phase (Streaming)

After production validation:
- [ ] Implement streaming with heartbeat
- [ ] Add progress indicators ("Processing chunk X/Y")
- [ ] Real-time status updates
- [ ] 100% timeout elimination

---

## Contact & Support

**If issues occur**:
1. Check Railway logs first
2. Check Eclipse console for error messages
3. Review [STABILITY_ISSUE_2026-01-21.md](STABILITY_ISSUE_2026-01-21.md)
4. Review [DEPLOYMENT_STATUS_2026-01-21.md](DEPLOYMENT_STATUS_2026-01-21.md)

**Emergency rollback**: Revert commits 498d07e and 72f65b0

---

**Last Updated**: January 21, 2026 @ 16:50
**Status**: ✅ Ready for verification testing
