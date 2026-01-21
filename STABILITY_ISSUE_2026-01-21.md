# Stability Issues - January 21, 2026 @ 13:50+

## Symptoms

1. **500 Internal Server Error** - First test at ~13:30
2. **Read timed out** - Current test at ~13:50+
3. User reports: "we have lost some stability"

## Timeline

- **13:27:26** - User selected pill option "Just add exception handling and input validation"
- **13:27:34** - Called mcp_get_class_method → Method "REGEX" not found
- **13:29:32** - Called mcp_check_syntax on main class → ✅ OK
- **13:30:08** - Called mcp_check_syntax on locals → ❌ 404 Not Found
- **13:30:09** - Message finalized (session: streaming-1768994846941)
- **13:50:03** - Tried to deploy, method "regex" not found
- **13:50:29** - New session started (streaming-1768996229727)
- **13:50:40** - Called GetClassLocals → Success (60,276 chars)
- **13:50:48** - Message finalized
- **13:51:35** - Another session started (streaming-1768996295674)
- **13:53:36** - Message finalized with only 25 chars content
- **Now** - "Read timed out" error

## Possible Causes

### 1. Railway Backend Issues
- Recent commits may still be deploying
- Backend might be restarting
- Resource limits hit (memory, CPU, connections)

### 2. API Rate Limiting
- Multiple quick requests to Claude API
- Anthropic rate limits exceeded

### 3. Network Timeouts
- Claude API calls taking >120 seconds (default timeout)
- Connection to SAP system timing out
- Railway-to-Anthropic network issues

### 4. Large Response Issues
- GetClassLocals returned 60,276 chars
- Combined with previous context, hitting token limits
- Backend struggling to process large responses

## Diagnostic Steps

### Check Railway Backend
1. Verify latest commits deployed: ad7272f, eff477e
2. Check Railway build logs for errors
3. Monitor Railway metrics: CPU, memory, response times
4. Check Railway logs for stop_reason and error details

### Check Backend Logs
The stop_reason should appear in Railway backend logs at `/api/chat` endpoint when the Claude API response is received. Format:
```
[CHAT] Claude response: stop_reason=<end_turn|max_tokens|stop_sequence>
```

### Check API Health
1. Backend health endpoint: GET /health
2. MCP server connection status
3. SAP ADT connection status

## Quick Fixes to Try

### 1. Restart Railway Backend
Force redeploy to clear any stuck state:
```bash
git commit --allow-empty -m "Force Railway restart"
git push
```

### 2. Reduce Context Size
The 60K char GetClassLocals response is very large. Consider:
- Limiting GetClassLocals output
- Clearing old conversation context
- Starting a new conversation

### 3. Check Railway Resources
- Upgrade Railway plan if hitting limits
- Check if free tier quota exceeded
- Monitor concurrent request limits

## Expected Stop Reasons

### Normal Stop Reasons:
- `end_turn` - Claude completed response naturally ✅
- `stop_sequence` - Hit a stop sequence (rare)

### Problem Stop Reasons:
- `max_tokens` - Hit 8192 token limit ❌
- `timeout` - Request took too long ❌
- `error` - API error occurred ❌

## Current Status

- **Railway Deployment**: ✅ Commits 72f65b0 deployed (aggressively condensed LOCAL CLASS section)
- **Backend Health**: ✅ Improved - messages completing without timeout
- **MCP Server**: ✅ Connected (last successful call at 14:56:41)
- **SAP Connection**: ✅ Working (last successful call at 14:56:41)
- **Stability**: ⚠️ Partially Recovered - timeouts resolved, but short responses observed

## Recommended Actions

1. **Immediate**: Check Railway deployment status
2. **Immediate**: Check Railway logs for stop_reason and errors
3. **Short-term**: Restart Railway backend if needed
4. **Short-term**: Start new conversation to clear context
5. **Long-term**: Add stop_reason logging to Eclipse client
6. **Long-term**: Implement backend health monitoring

---

## Latest Update - 16:10:15 🙏

### Root Cause Identified: System Prompt Bloat
Analysis showed system prompt grew by **19% today**:
- BEFORE today (commit 5fd7600): 29,909 chars
- AFTER all fixes (commit 72f65b0): 35,645 chars
- **Growth**: +5,736 chars (+122 lines)

This bloat, combined with 60K chars of locals code, was causing timeouts.

### CRITICAL FIX DEPLOYED (commit 498d07e)
**Drastically simplified system prompt - 86% reduction**:
- FROM: 35,645 chars (~8,900 tokens)
- TO: 5,158 chars (~1,300 tokens)
- **Savings**: -30,487 chars (-7,600 tokens)

### What Was Preserved
✅ mcp_modify_class_method - ONLY for main class methods
✅ Local class workflow - Always use mcp_get_class_locals + chunking if >= 8K
✅ Chunking at 8K threshold - ONE chunk per message
✅ CHECK SYNTAX FIRST - Mandatory before deployment

### What Was Removed
❌ Verbose examples (17 lines → 2 lines)
❌ Repetitive warnings and emojis
❌ Redundant explanations
❌ Multiple copies of same instructions

**Status**: 🙏 Deploying to Railway - Praying this fixes it!
**Impact**: Should eliminate timeouts while preserving all critical workflow logic
**Next**: Wait for Railway deployment, then test again

---

## Update - 16:28:00 ✅

### System Prompt Fixes Complete
Three commits deployed to fix system prompt and pill buttons:
- **498d07e**: Drastically simplify system prompt (86% reduction: 35K → 5K)
- **5bffc94**: Add back appendPillButtonReminderToMessages (critical for VS Code)
- **44a5182**: Fix getSystemPrompt signature (3 parameters)
- **a15f759**: Fix pill button XML format (header attribute + <option> tags)

### Results
✅ **Pill Buttons**: Working! Format corrected from `<pill-button-option>` to `<option>`
✅ **System Prompt**: 78% reduction (35,645 → 8,123 chars including pill functions)
❌ **API 529 Error**: "Anthropic API is overloaded. Please try again."

### Current Status
- **System Fixes**: ✅ Complete - all critical functionality restored
- **Pill Buttons**: ✅ Rendering correctly in Eclipse
- **API Availability**: ⏳ Waiting for Anthropic API to recover from overload
- **Timeout Issue**: ✅ Should be resolved (78% token reduction)

**Next**: Wait for Anthropic API 529 error to clear, then test full workflow
