# File-Based Chunking Test - Using Real Data
## January 22, 2026

---

## Test Data from Successful Session

**Session**: `ZCL_REGEX_SECURE_1737493000`
**Date**: January 21, 2026 @ 22:37-22:46
**Total code**: 76,488 characters
**Chunks used**: 15 chunks (with 3K strategy)

### Actual Chunk Sizes:
```
Chunk 1:  5,898 chars
Chunk 2:  8,540 chars
Chunk 3:  5,251 chars
Chunk 4:  4,880 chars
Chunk 5:  4,760 chars
Chunk 6:  4,137 chars
Chunk 7:  4,195 chars
Chunk 8:  5,448 chars
Chunk 9:  5,569 chars
Chunk 10: 5,973 chars
Chunk 11: 6,358 chars
Chunk 12: 7,618 chars
Chunk 13: 5,484 chars
Chunk 14: 2,376 chars
Chunk 15: 1 char (just newline)
────────────────────
Total: 76,488 chars
```

**Problem with chunk 15**: Hit token limit, sent empty chunk (CHUNK_MISSING_CODE error), then retry sent only 1 char.

---

## Test Plan: File-Based Chunking with 8K Strategy

### Step 1: Calculate Chunks with New 8K Strategy

```
Total code: 76,488 chars
Chunk size: 8,000 chars
Chunks needed: ceil(76,488 / 8,000) = 10 chunks (instead of 15!)
```

### Step 2: Theoretical Chunk Distribution (8K each)

```
Chunk 1:  8,000 chars
Chunk 2:  8,000 chars
Chunk 3:  8,000 chars
Chunk 4:  8,000 chars
Chunk 5:  8,000 chars
Chunk 6:  8,000 chars
Chunk 7:  8,000 chars
Chunk 8:  8,000 chars
Chunk 9:  8,000 chars
Chunk 10: 4,488 chars (remaining)
──────────────────────
Total: 76,488 chars
```

**Benefits**:
- ✅ 10 chunks instead of 15 (33% fewer!)
- ✅ Each chunk ~2,000 tokens (safe margin)
- ✅ No token limit issues

---

## Simulated File-Based Workflow

### Using Windows C:\temp

```batch
REM Session ID
SET SESSION=ZCL_BM_LEARNING_S4D401_1737493000

REM Create session directory
mkdir "C:\temp\oricode\%SESSION%"

REM AI writes chunks using write_file tool
REM (10 file writes instead of 15)
write_file(path: "C:\temp\oricode\%SESSION%\chunk_001.abap", content: chunk1_8000_chars)
write_file(path: "C:\temp\oricode\%SESSION%\chunk_002.abap", content: chunk2_8000_chars)
...
write_file(path: "C:\temp\oricode\%SESSION%\chunk_010.abap", content: chunk10_4488_chars)

REM Merge all chunks into complete file
type "C:\temp\oricode\%SESSION%\chunk_*.abap" > "C:\temp\oricode\%SESSION%\complete.abap"

REM Read complete file
read_file(path: "C:\temp\oricode\%SESSION%\complete.abap")

REM Check syntax
mcp_check_syntax(object_type: "class", object_name: "ZCL_BM_LEARNING_S4D401", source_code: complete_code, include_type: "implementations")

REM If OK, deploy
mcp_deploy_stored_code(session_id: "%SESSION%", object_type: "class", object_name: "ZCL_BM_LEARNING_S4D401", code: complete_code, include_type: "implementations", user_confirmed: false)

REM Cleanup
rmdir /s /q "C:\temp\oricode\%SESSION%"
```

---

## Comparison: Old vs New Strategy

| Metric | Old (Backend Memory, 3K) | New (File-Based, 8K) |
|--------|--------------------------|----------------------|
| **Chunks** | 15 chunks | 10 chunks |
| **File writes** | 15 mcp_store_code_chunk calls | 10 write_file calls |
| **Token risk** | ⚠️ Hit limit on chunk 15 | ✅ Safe (2K tokens/chunk) |
| **Error** | CHUNK_MISSING_CODE (chunk 15) | None expected |
| **Storage** | Backend memory (ephemeral) | C:\temp (persistent) |
| **User visibility** | None | Full (can open files) |
| **Recovery** | Lost on timeout | Persists |
| **Time** | ~9 minutes | ~6 minutes (estimate) |

---

## Expected Results with File-Based Chunking

### No CHUNK_MISSING_CODE Error:
- ✅ Each chunk ~2,000 tokens (well under 8,192 limit)
- ✅ Even if AI is verbose (1,000 tokens reasoning), still 5,000 tokens left
- ✅ Files have no token constraints

### Faster Execution:
- ✅ 10 writes instead of 15 (33% fewer)
- ✅ No backend round-trips during chunking
- ✅ Local file operations are instant

### Better User Experience:
- ✅ User can inspect files in C:\temp\oricode\{session}\
- ✅ Can see progress (chunk_001.abap, chunk_002.abap, ...)
- ✅ Can manually fix if needed
- ✅ Files persist through timeouts

---

## Validation Test

After Railway deploys the new changes, run this test:

### Test Command:
```
"Can you help me improve the regex method in class zcl_bm_learning_s4d401"
```

### Expected Flow:
1. AI reads locals (76K chars)
2. AI calculates: ceil(76488 / 8000) = 10 chunks
3. AI writes chunk_001.abap through chunk_010.abap to C:\temp\oricode\{session}\
4. AI merges: type chunk_*.abap > complete.abap
5. AI reads complete.abap
6. AI checks syntax: mcp_check_syntax
7. AI deploys: mcp_deploy_stored_code
8. AI cleans up: rmdir /s /q C:\temp\oricode\{session}

### Success Criteria:
- [  ] All 10 chunks written successfully
- [  ] No CHUNK_MISSING_CODE errors
- [  ] complete.abap is 76,488 chars
- [  ] Last 200 chars end with ENDCLASS
- [  ] Syntax check passes
- [  ] Deployment succeeds
- [  ] Pill button appears
- [  ] Cleanup removes files

---

## Edge Cases to Test

### Test 1: Code < 30K (no chunking needed)
- Code: 25,000 chars
- Expected: Direct deployment, no files written

### Test 2: Code = 32K (just over threshold)
- Code: 32,000 chars
- Chunks: ceil(32000 / 8000) = 4 chunks
- Expected: 4 files, merge, deploy

### Test 3: Code = 100K (very large)
- Code: 100,000 chars
- Chunks: ceil(100000 / 8000) = 13 chunks
- Expected: 13 files, merge, deploy

---

## Rollback Plan

If file-based chunking fails:

1. Check C:\temp\oricode\ for files
2. Verify chunks were written
3. Check if merge worked (complete.abap exists)
4. Check syntax error logs
5. If persistent failure, revert commits:
   ```bash
   git revert 8538388  # Revert chunk size change
   git revert 35f2c9b  # Revert threshold change
   git revert 5e7d1b1  # Revert file-based chunking
   git push
   ```

---

**Status**: Ready for testing when Railway deploys
**Expected deployment**: ~3-5 minutes from last push
**Test ready**: January 22, 2026 @ 01:40

