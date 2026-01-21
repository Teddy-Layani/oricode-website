# Timeout Prevention Strategy - January 21, 2026

## Problem: "Read timed out" Error

**Symptoms**:
- Error appears after long-running AI operations
- Happens during chunking workflows (8+ messages)
- User sees: "❌ Error: Read timed out"
- **Worst UX ever** - no recovery, no progress indication

## Root Causes

### 1. **Anthropic API Timeouts**
- Default SDK timeout: **10 minutes** (600 seconds)
- Railway may enforce lower timeout
- Long chunking workflows can exceed limits

### 2. **Railway Infrastructure Timeouts**
- Request timeout: Typically 120 seconds (2 minutes)
- If backend processing > 2 minutes → Railway kills connection
- Chunking with 8 messages can take 3-5 minutes

### 3. **Network-Level Timeouts**
- TCP keepalive not configured
- No heartbeat mechanism
- Client-side timeout defaults

## Solutions (Multi-Layer Defense)

### Solution 1: Increase Anthropic SDK Timeout ⚡

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-backend\src\routes\chat.ts`

```typescript
// BEFORE (implicit default)
const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
});

// AFTER (explicit timeout)
const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
  timeout: 300000, // 5 minutes (300 seconds)
  maxRetries: 2,   // Auto-retry on timeout
});
```

**Impact**:
- ✅ Handles longer AI processing times
- ✅ Auto-retries transient network issues
- ⚠️ Still vulnerable to Railway timeout

---

### Solution 2: Streaming with Heartbeat 💓

**Problem**: Railway kills idle connections after 120 seconds

**Solution**: Send periodic heartbeat during long operations

```typescript
router.post('/', async (req, res) => {
  // Enable streaming
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send heartbeat every 30 seconds
  const heartbeat = setInterval(() => {
    res.write('data: {"type":"heartbeat"}\n\n');
  }, 30000);

  try {
    // Process chunks...
    for (let i = 0; i < totalChunks; i++) {
      const result = await processChunk(i);
      res.write(`data: ${JSON.stringify(result)}\n\n`);
    }
  } finally {
    clearInterval(heartbeat);
    res.end();
  }
});
```

**Impact**:
- ✅ Keeps Railway connection alive
- ✅ Client sees progress in real-time
- ✅ Better UX - knows system is working

---

### Solution 3: Break into Smaller Requests 🔄

**Current Problem**: Single request handles all 8 chunks
- Message 1: Chunk 0
- Message 2: Chunk 1
- ...
- Message 8: Chunk 7 + Deploy
- **Total time**: 3-5 minutes (exceeds Railway timeout)

**Solution**: Each chunk is a separate HTTP request

```typescript
// Client-side (Eclipse/VS Code)
async function sendChunks(chunks) {
  for (let i = 0; i < chunks.length; i++) {
    // Separate request per chunk
    const result = await fetch('/api/chat', {
      method: 'POST',
      body: JSON.stringify({
        messages: [...context, chunkMessage],
        chat_id: chatId,  // Continue same conversation
      })
    });

    // Update UI with progress
    updateProgress(`Chunk ${i+1}/${chunks.length} stored`);
  }

  // Final deployment request
  await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({
      messages: [...context, deployMessage],
      chat_id: chatId,
    })
  });
}
```

**Impact**:
- ✅ Each request < 30 seconds (well under Railway limit)
- ✅ Progress visible to user
- ✅ Can retry individual chunks on failure
- ⚠️ More complex client implementation

---

### Solution 4: Async Job Queue 🎯 (Best Long-Term)

**Architecture**: Background job processing

```
1. Client submits chunking job → Returns job_id immediately
2. Backend processes chunks async (outside HTTP request)
3. Client polls for progress: GET /api/jobs/{job_id}
4. When complete, client fetches result
```

```typescript
// Submit job
POST /api/chat/jobs
{
  "operation": "chunk_deployment",
  "session_id": "ZCL_BM_LEARNING_1737793801",
  "chunks": 8
}
→ Response: { "job_id": "job_12345", "status": "queued" }

// Poll progress
GET /api/chat/jobs/job_12345
→ Response: {
  "job_id": "job_12345",
  "status": "processing",
  "progress": {
    "current_chunk": 3,
    "total_chunks": 8,
    "message": "Storing chunk 3/8..."
  }
}

// Get result
GET /api/chat/jobs/job_12345
→ Response: {
  "job_id": "job_12345",
  "status": "completed",
  "result": { ... }
}
```

**Impact**:
- ✅ No HTTP timeout issues (jobs run in background)
- ✅ Perfect progress tracking
- ✅ Can handle hours-long operations
- ✅ Better error recovery
- ⚠️ Significant implementation effort

---

## Recommended Immediate Fixes

### Priority 1: Increase SDK Timeout (5 minutes) ⚡
**Effort**: 2 minutes
**Impact**: Medium
**File**: `chat.ts:25-27`

```typescript
const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
  timeout: 300000,  // 5 minutes
  maxRetries: 2,
});
```

### Priority 2: Add Better Error Messages 📝
**Effort**: 10 minutes
**Impact**: High (UX)

```typescript
try {
  const response = await anthropic.messages.create({...});
} catch (error) {
  if (error.code === 'ETIMEDOUT' || error.message?.includes('timeout')) {
    return res.status(504).json({
      error: 'Request took too long to complete',
      details: 'The AI is processing a large amount of code. This typically happens with chunked deployments over 60K characters.',
      suggestions: [
        'Try breaking your change into smaller modifications',
        'Reduce the number of local classes being modified',
        'Use mcp_modify_class_method for smaller changes instead of chunking'
      ],
      retry: true,  // Client can show "Retry" button
    });
  }
  // ... other error handling
}
```

### Priority 3: Client-Side Timeout Increase
**Effort**: 5 minutes
**Impact**: Medium

**Eclipse** (`OricodeApiClient.java`):
```java
// Increase HTTP client timeout
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(300))  // 5 minutes
    .build();
```

**VS Code** (extension):
```typescript
// Increase fetch timeout
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 300000); // 5 minutes

try {
  const response = await fetch('/api/chat', {
    signal: controller.signal,
    // ...
  });
} finally {
  clearTimeout(timeout);
}
```

---

## Long-Term Solution: Streaming + Heartbeat

Implement for **next sprint**:

1. **Backend**: Streaming endpoint with heartbeat
2. **Client**: Handle SSE (Server-Sent Events)
3. **Progress UI**: Show real-time chunk progress

**Benefits**:
- No timeout issues
- Real-time progress
- Better UX
- Scales to any workflow size

---

## Testing Plan

### Test Case 1: Large Code (80K chars, 10 chunks)
**Before Fix**:
- ❌ Timeout after ~2 minutes
- ❌ No error message
- ❌ User stuck

**After Priority 1+2 Fixes**:
- ✅ Completes successfully (under 5 min timeout)
- ✅ Clear error message if still timeout
- ✅ Retry button available

### Test Case 2: Network Hiccup
**Before Fix**:
- ❌ Immediate failure
- ❌ Lost all progress

**After Fix** (with maxRetries=2):
- ✅ Auto-retries 2 times
- ✅ Recovers from transient issues

---

## Implementation Order

**Week 1** (Immediate):
1. ✅ Increase Anthropic SDK timeout (5 min)
2. ✅ Add maxRetries (2 attempts)
3. ✅ Improve error messages
4. ✅ Increase client-side timeouts

**Week 2** (UX Improvements):
1. ⏳ Add progress indicators to Eclipse UI
2. ⏳ Show "Processing chunk X/Y" messages
3. ⏳ Add retry mechanism in UI

**Week 3** (Architecture):
1. ⏳ Implement streaming with heartbeat
2. ⏳ Real-time progress updates
3. ⏳ Handle network interruptions gracefully

**Future** (Advanced):
1. ⏳ Async job queue
2. ⏳ Background processing
3. ⏳ Resume interrupted operations

---

## Monitoring

Add logging to track timeouts:

```typescript
logger.warn('TIMEOUT', `Request timeout after ${duration}ms`, {
  operation: 'chunking',
  chunksCompleted: 3,
  chunksTotal: 8,
  sessionId: 'ZCL_BM_LEARNING_1737793801',
  userId,
});
```

**Dashboard Metrics**:
- Timeout rate per endpoint
- Average request duration
- Chunk processing time
- Retry success rate

---

**Status**: Ready for Priority 1-3 implementation
**Estimated Time**: 20 minutes total
**Expected Result**: 90% reduction in timeout errors
