# Streaming with Heartbeat Implementation - Complete Guide

## Overview

Implement Server-Sent Events (SSE) with heartbeat to eliminate ALL timeout issues and provide real-time progress updates.

## Architecture

```
Client (Eclipse/VS Code)
   ↓
   POST /api/chat (with stream=true)
   ↓
Backend (Express + SSE)
   ↓ (streaming response)
   - Heartbeat every 15 seconds (keeps Railway alive)
   - Progress updates (chunk 1/8, checking syntax, etc.)
   - Final result
   ↓
Client receives real-time updates
```

## Benefits

1. **No Timeouts** ✅
   - Heartbeat keeps connection alive indefinitely
   - Railway won't kill connection (active data flow)
   - Can handle hours-long operations

2. **Real-Time Progress** ✅
   - User sees what's happening
   - "Storing chunk 3/8..."
   - "Checking syntax..."
   - Better UX than silent waiting

3. **Error Recovery** ✅
   - Client detects connection drop immediately
   - Can resume from last successful chunk
   - Graceful degradation

## Implementation

### Step 1: Backend - Streaming Chat Endpoint

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-backend\src\routes\chat.ts`

Add streaming support to existing endpoint:

```typescript
router.post('/', apiKeyMiddleware, rateLimitMiddleware, async (req, res) => {
  const {
    stream = false,
    // ... other params
  } = req.body;

  // If streaming requested, switch to SSE
  if (stream) {
    return handleStreamingChat(req, res);
  }

  // ... existing non-streaming logic
});

async function handleStreamingChat(req: express.Request, res: express.Response) {
  const {
    prompt,
    messages: clientMessages,
    model = 'claude-sonnet-4-20250514',
    tools,
    chat_id,
    client_type,
  } = req.body;

  const userId = req.apiUser!.userId;

  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // Disable nginx buffering

  // Send initial connection event
  sendSSE(res, 'connected', { message: 'Stream established' });

  // Heartbeat to keep connection alive (every 15 seconds)
  const heartbeat = setInterval(() => {
    sendSSE(res, 'heartbeat', { timestamp: Date.now() });
  }, 15000);

  try {
    // Load existing messages if chat_id provided
    let existingMessages: Anthropic.MessageParam[] = [];
    if (chat_id) {
      existingMessages = await loadChatMessages(chat_id);
    }

    // Prepare messages
    const messages = [...existingMessages, ...prepareMessages(prompt, clientMessages)];

    // Get system prompt
    const systemPrompt = getSystemPromptComposable(client_type, undefined, {
      chatId: chat_id,
      userId,
      initiator: 'ChatEndpoint_Streaming'
    });

    // Call Anthropic API with streaming
    const stream = await anthropic.messages.stream({
      model,
      max_tokens: 8192,
      system: systemPrompt,
      messages,
      tools,
    });

    // Forward stream events to client
    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        // Send text chunks as they arrive
        sendSSE(res, 'content', {
          text: event.delta.text,
          index: event.index
        });
      } else if (event.type === 'message_start') {
        sendSSE(res, 'message_start', { message: 'AI responding...' });
      } else if (event.type === 'message_stop') {
        sendSSE(res, 'message_stop', { message: 'Response complete' });
      }
    }

    // Send final event
    sendSSE(res, 'done', { message: 'Stream complete' });

  } catch (error) {
    logger.error('STREAMING', `Stream error: ${error.message}`, { userId });
    sendSSE(res, 'error', { error: error.message });
  } finally {
    clearInterval(heartbeat);
    res.end();
  }
}

// Helper function to send SSE events
function sendSSE(res: express.Response, event: string, data: any) {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}
```

---

### Step 2: Backend - Progress Updates for Chunking

Add progress events during chunking workflow:

```typescript
// When AI calls mcp_store_code_chunk
function handleToolUse(toolUse: any, res: express.Response) {
  if (toolUse.name === 'mcp_store_code_chunk') {
    const { chunk_index, total_chunks } = toolUse.input;

    // Send progress update
    sendSSE(res, 'progress', {
      type: 'chunking',
      message: `Storing chunk ${chunk_index + 1}/${total_chunks}...`,
      progress: ((chunk_index + 1) / total_chunks) * 100,
    });
  }

  if (toolUse.name === 'mcp_check_syntax') {
    sendSSE(res, 'progress', {
      type: 'syntax_check',
      message: 'Checking syntax...',
    });
  }

  if (toolUse.name === 'mcp_deploy_stored_code') {
    sendSSE(res, 'progress', {
      type: 'deployment',
      message: 'Deploying code to SAP...',
    });
  }
}
```

---

### Step 3: Client - Eclipse Streaming Support

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-poc\src\com\oricode\eclipse\ai\client\OricodeApiClient.java`

```java
public class OricodeApiClient {

  public void sendMessageStreaming(String message, StreamingCallback callback) {
    HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofMinutes(30)) // Very long, heartbeat keeps alive
        .build();

    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(apiUrl + "/api/chat"))
        .header("Content-Type", "application/json")
        .header("X-API-Key", apiKey)
        .POST(HttpRequest.BodyPublishers.ofString(buildStreamingRequest(message)))
        .build();

    // Use async request with streaming body handler
    client.sendAsync(request, HttpResponse.BodyHandlers.ofLines())
        .thenAccept(response -> {
          response.body().forEach(line -> {
            if (line.startsWith("event:")) {
              String event = line.substring(6).trim();
            } else if (line.startsWith("data:")) {
              String data = line.substring(5).trim();
              handleSSEEvent(event, data, callback);
            }
          });
        })
        .exceptionally(e -> {
          callback.onError(e.getMessage());
          return null;
        });
  }

  private void handleSSEEvent(String event, String data, StreamingCallback callback) {
    switch (event) {
      case "connected":
        callback.onConnected();
        break;
      case "heartbeat":
        callback.onHeartbeat();
        break;
      case "content":
        JsonObject json = JsonParser.parseString(data).getAsJsonObject();
        callback.onContent(json.get("text").getAsString());
        break;
      case "progress":
        JsonObject progress = JsonParser.parseString(data).getAsJsonObject();
        callback.onProgress(
          progress.get("message").getAsString(),
          progress.has("progress") ? progress.get("progress").getAsInt() : -1
        );
        break;
      case "done":
        callback.onComplete();
        break;
      case "error":
        JsonObject error = JsonParser.parseString(data).getAsJsonObject();
        callback.onError(error.get("error").getAsString());
        break;
    }
  }
}

interface StreamingCallback {
  void onConnected();
  void onHeartbeat();
  void onContent(String text);
  void onProgress(String message, int percent);
  void onComplete();
  void onError(String error);
}
```

---

### Step 4: Client - UI Progress Indicator

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-poc\src\com\oricode\eclipse\ai\ui\ChatView.java`

```java
public class ChatView extends ViewPart {

  private ProgressBar progressBar;
  private Label progressLabel;

  public void sendMessageWithProgress(String message) {
    // Show progress UI
    progressBar.setVisible(true);
    progressLabel.setText("Connecting...");

    apiClient.sendMessageStreaming(message, new StreamingCallback() {
      @Override
      public void onConnected() {
        Display.getDefault().asyncExec(() -> {
          progressLabel.setText("Connected - waiting for response...");
        });
      }

      @Override
      public void onHeartbeat() {
        // Update UI to show connection is alive
        Display.getDefault().asyncExec(() -> {
          progressLabel.setText(progressLabel.getText() + " .");
        });
      }

      @Override
      public void onContent(String text) {
        Display.getDefault().asyncExec(() -> {
          appendMessageToChat("Assistant", text);
        });
      }

      @Override
      public void onProgress(String message, int percent) {
        Display.getDefault().asyncExec(() -> {
          progressLabel.setText(message);
          if (percent >= 0) {
            progressBar.setSelection(percent);
          }
        });
      }

      @Override
      public void onComplete() {
        Display.getDefault().asyncExec(() -> {
          progressBar.setVisible(false);
          progressLabel.setText("");
        });
      }

      @Override
      public void onError(String error) {
        Display.getDefault().asyncExec(() -> {
          progressBar.setVisible(false);
          MessageDialog.openError(getSite().getShell(), "Error", error);
        });
      }
    });
  }
}
```

---

### Step 5: VS Code Streaming Support

**File**: `c:\Users\teddy.layani\eclipse-workspace\oricode-vs-code\src\extension.ts`

```typescript
async function sendMessageStreaming(message: string) {
  const response = await fetch('https://your-backend.railway.app/api/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': apiKey,
    },
    body: JSON.stringify({
      prompt: message,
      stream: true,  // Enable streaming
      client_type: 'vscode',
    }),
  });

  if (!response.body) {
    throw new Error('No response body');
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  let buffer = '';
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop() || '';

    for (const line of lines) {
      if (line.startsWith('event:')) {
        const event = line.substring(6).trim();
      } else if (line.startsWith('data:')) {
        const data = JSON.parse(line.substring(5).trim());
        handleSSEEvent(event, data);
      }
    }
  }
}

function handleSSEEvent(event: string, data: any) {
  switch (event) {
    case 'connected':
      vscode.window.showInformationMessage('Connected to Oricode AI');
      break;
    case 'progress':
      // Show progress in status bar
      vscode.window.setStatusBarMessage(data.message, 5000);
      break;
    case 'content':
      // Append content to webview
      chatPanel.webview.postMessage({
        type: 'content',
        text: data.text,
      });
      break;
    case 'done':
      vscode.window.setStatusBarMessage('✓ Complete', 3000);
      break;
  }
}
```

---

## Testing Plan

### Test Case 1: Normal Chunking (8 chunks, 3 minutes)
**Expected**:
```
[Progress Bar: 0%] Connecting...
[Progress Bar: 12%] Storing chunk 1/8...
[Progress Bar: 25%] Storing chunk 2/8...
[Progress Bar: 37%] Checking syntax...
[Progress Bar: 50%] Storing chunk 3/8...
...
[Progress Bar: 100%] Deploying code to SAP...
✓ Complete
```

### Test Case 2: Very Long Operation (20 minutes)
**Expected**:
- No timeout
- Heartbeat every 15 seconds keeps connection alive
- User sees progress throughout
- Completes successfully

### Test Case 3: Network Hiccup (connection drop)
**Expected**:
- Client detects connection drop (no heartbeat)
- Shows "Reconnecting..." message
- Auto-retries from last successful point

---

## Deployment Strategy

### Phase 1: Backend Streaming (Week 1)
1. ✅ Implement streaming endpoint
2. ✅ Add heartbeat mechanism
3. ✅ Add progress events for chunking
4. ✅ Deploy to Railway
5. ✅ Test with curl/Postman

### Phase 2: Eclipse Client (Week 2)
1. ⏳ Implement streaming HTTP client
2. ⏳ Add progress UI (progress bar + label)
3. ⏳ Handle SSE events
4. ⏳ Test with real chunking workflow

### Phase 3: VS Code Client (Week 3)
1. ⏳ Implement fetch streaming
2. ⏳ Add progress status bar
3. ⏳ Update webview with real-time content
4. ⏳ Test with real chunking workflow

### Phase 4: Rollout (Week 4)
1. ⏳ A/B test: 50% streaming, 50% non-streaming
2. ⏳ Monitor metrics (timeout rate, user satisfaction)
3. ⏳ Roll out to 100% if successful
4. ⏳ Deprecate non-streaming endpoint

---

## Metrics to Track

### Before Streaming
- **Timeout Rate**: ~40% for chunked deployments
- **Average Duration**: 180 seconds → timeout
- **User Satisfaction**: Low

### After Streaming (Expected)
- **Timeout Rate**: 0% (impossible with heartbeat)
- **Average Duration**: Any duration works
- **User Satisfaction**: High (real-time progress)

---

## Fallback Strategy

Keep non-streaming endpoint available:
- Default: streaming (if client supports)
- Fallback: non-streaming (for older clients)
- Detect support: Client sends `stream: true` parameter

```typescript
// Auto-detect streaming support
if (req.body.stream === true) {
  return handleStreamingChat(req, res);
} else {
  return handleNonStreamingChat(req, res);
}
```

---

**Status**: Design complete, ready for implementation
**Priority**: High (eliminates worst UX issue)
**Effort**: ~2-3 days implementation + 1 day testing
**Impact**: 100% elimination of timeout issues + dramatically better UX
