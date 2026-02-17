# Table Creation Bug: Fields Parameter Missing

## 🔍 Quick Reference - Where Are Tool Definitions?

| Component | File Path | Search Keywords |
|-----------|-----------|-----------------|
| **MCP Server (SOURCE OF TRUTH)** | `mcp-abap-adt-crud/src/index.ts` | `CreateTable`, `DDIC CREATE TOOLS` |
| **Eclipse - DDIC Tools** | `oricode-poc/src/com/oricode/eclipse/ai/tools/EclipseTools.java` | `mcp_create_table`, `getToolDefinitionsJson` |
| **Eclipse - MCP Tools** | `oricode-poc/src/com/oricode/eclipse/ai/mcp/McpAbapClient.java` | `mcp_search`, `getToolDefinitionsJson` |
| **Eclipse - Tool Input Forwarding** | `oricode-poc/src/com/oricode/eclipse/ai/api/OricodeApiClient.java` | `DDIC TOOLS`, `mcp_create_table`, `parseFieldsArray` |
| **VS Code** | `oricode-vs-code/src/tools/ToolRouter.ts` | `getVsCodeToolDefinitions`, `TOOL_NAME_MAP` |

## 🏗️ Architectural Problem

**THIS IS NOT A BUG - IT'S A DESIGN FLAW!**

Tool definitions are **hardcoded in multiple places** instead of being centralized:

1. ✅ **MCP Server** (TypeScript) - Source of truth, defines complete schemas
2. ❌ **Eclipse EclipseTools.java** - DDIC tools (mcp_create_table, mcp_create_domain)
3. ❌ **Eclipse McpAbapClient.java** - MCP tools (mcp_search, mcp_get_class, etc.)
4. ❌ **VS Code ToolRouter.ts** - Also has hardcoded definitions

**Problem:** Every time we update a tool (add parameter, change description), we must manually sync 3+ files. This causes:
- Incomplete tool definitions (like missing `fields` property)
- Synchronization nightmares
- Maintenance hell
- Constant bugs like this one

## Problem Summary

When Eclipse AI calls `mcp_create_table`, the **`fields` array parameter is missing** from the tool input, causing the error:
```
Error: fields array is required and must not be empty
```

## Root Cause

The Eclipse plugin (**McpAbapClient.java**) is **NOT forwarding the complete tool definition** from the MCP server to Claude AI.

### Evidence

1. **MCP Server Definition** ([index.ts:909-936](c:\Users\teddy.layani\eclipse-workspace\mcp-abap-adt-crud\src\index.ts#L909-L936)):
   ```typescript
   {
     name: 'CreateTable',
     description: 'Create a new database table (DDIC) with DDL source code.',
     inputSchema: {
       type: 'object',
       properties: {
         table_name: { type: 'string', description: '...' },
         description: { type: 'string', description: '...' },
         package: { type: 'string', description: '...' },
         fields: {  // ✅ DEFINED HERE
           type: 'array',
           description: 'Array of field definitions',
           items: { ... }
         },
         transport: { type: 'string', description: '...' }
       },
       required: ['table_name', 'description', 'package', 'fields']  // ✅ MARKED REQUIRED
     }
   }
   ```

2. **What Eclipse Sends to Claude** ([c:\temp\oricode_request.json:878-901](c:\temp\oricode_request.json#L878-L901)):
   ```json
   {
     "name": "mcp_create_table",
     "description": "Create a new database table (DDIC transparent table).",
     "input_schema": {
       "type": "object",
       "properties": {
         "table_name": { "type": "string", "description": "..." },
         "description": { "type": "string", "description": "..." },
         "package": { "type": "string", "description": "..." },
         "transport": { "type": "string", "description": "..." }
         // ❌ NO "fields" PROPERTY!
       },
       "required": ["table_name", "description", "package"]  // ❌ "fields" NOT REQUIRED
     }
   }
   ```

3. **Debug Log Confirms** ([c:/temp/oricode_debug.log](c:/temp/oricode_debug.log)):
   ```
   [MCP] args?.fields type: undefined, value: undefined
   [MCP] rawFields after extraction: undefined
   [MCP] Fields validation failed. rawFields: undefined, length: undefined
   ```

## ✅ Proper Centralized Architecture

### Option A: Backend as Single Source of Truth (RECOMMENDED)

```
┌──────────────┐        ┌──────────────┐
│   Eclipse    │        │   VS Code    │
└──────┬───────┘        └──────┬───────┘
       │ GET /api/tools/definitions  │
       │                             │
       └─────────────┬───────────────┘
                     ▼
            ┌─────────────────┐
            │  Oricode Backend │ ◄─── SINGLE SOURCE OF TRUTH
            │  Tool Registry   │      (REST API)
            └────────┬─────────┘
                     │ Fetches from MCP servers
                     ▼
            ┌─────────────────┐
            │   MCP Servers   │
            │  (ABAP, ...)    │
            └─────────────────┘
```

**Benefits:**
- ✅ ONE place to update tool definitions
- ✅ Eclipse and VS Code fetch from same source
- ✅ Can aggregate multiple MCP servers
- ✅ Simple REST API (no MCP protocol complexity in clients)
- ✅ Can add versioning, caching, validation

**API Design:**
```typescript
// Backend endpoint
GET /api/tools/definitions?client=eclipse&mcp_servers=abap,common

// Response
{
  "tools": [
    {
      "name": "mcp_create_table",
      "source": "abap-mcp-server",
      "description": "...",
      "input_schema": { /* complete schema */ }
    },
    // ...
  ],
  "version": "1.0.0",
  "last_updated": "2026-02-02T09:00:00Z"
}
```

### Option B: MCP Server as Source (Current Design)

```
┌──────────────┐        ┌──────────────┐
│   Eclipse    │        │   VS Code    │
└──────┬───────┘        └──────┬───────┘
       │ MCP ListTools          │ MCP ListTools
       │                        │
       └────────────┬───────────┘
                    ▼
           ┌─────────────────┐
           │   MCP Server    │ ◄─── Source of truth
           │   (TypeScript)  │
           └─────────────────┘
```

**Current Problems:**
- ❌ Eclipse hardcodes instead of calling MCP ListTools
- ❌ VS Code also has hardcoded tool definitions
- ❌ Both clients must implement MCP protocol correctly
- ❌ No aggregation of multiple MCP servers

### Current Broken Architecture

```
┌─────────────────┐
│   Claude API    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│ Eclipse McpAbapClient.java      │
│ getToolDefinitionsJson()        │ ◄─── HARDCODED DEFINITIONS! 😱
│ ❌ Incomplete schemas           │
│ ❌ Missing fields property      │
│ ❌ Must manually sync with MCP  │
└─────────────────────────────────┘

┌─────────────────┐
│   MCP Server    │ ◄─── Definitions exist but NOT USED! 🤦
│   (TypeScript)  │
└─────────────────┘
```

## Where to Fix

**File:** `**/McpAbapClient.java` (Eclipse plugin)

**Glob Pattern:** `**/McpAbapClient.java`

**Current Problem:** `getToolDefinitionsJson()` method hardcodes tool definitions

**Proper Fix:** Replace hardcoded definitions with dynamic MCP ListTools call

### Investigation Steps

1. **Check if DDIC tools are hardcoded:**
   ```bash
   grep -A 30 "mcp_create_table" **/McpAbapClient.java
   ```

2. **Find where tool definitions are fetched from MCP server:**
   ```bash
   grep -n "listTools\|ListTools\|getTools" **/McpAbapClient.java
   ```

3. **Check for JSON schema filtering or simplification:**
   ```bash
   grep -n "inputSchema\|input_schema\|properties" **/McpAbapClient.java
   ```

## 🎯 Implementation Plan: Centralize in Backend

### Phase 1: Create Backend Tool Registry

**File:** `oricode-backend/src/api/toolRegistry.ts` (NEW)

```typescript
import { McpClient } from '@/mcp/McpClient';

export class ToolRegistry {
  private toolCache: Map<string, ToolDefinition[]> = new Map();
  private mcpClients: Map<string, McpClient> = new Map();

  /**
   * Fetch tool definitions from all configured MCP servers
   */
  async getToolDefinitions(mcpServers: string[] = ['abap']): Promise<ToolDefinition[]> {
    const allTools: ToolDefinition[] = [];

    for (const serverName of mcpServers) {
      const client = this.mcpClients.get(serverName);
      if (!client) continue;

      // Call MCP ListTools
      const tools = await client.listTools();

      // Add source attribution
      tools.forEach(tool => {
        tool.source = serverName;
        allTools.push(tool);
      });
    }

    return allTools;
  }
}

// REST API endpoint
router.get('/api/tools/definitions', async (req, res) => {
  const mcpServers = req.query.mcp_servers?.split(',') || ['abap'];
  const tools = await toolRegistry.getToolDefinitions(mcpServers);
  res.json({ tools, version: '1.0.0', last_updated: new Date().toISOString() });
});
```

### Phase 2: Update Eclipse Plugin

**File:** `**/McpAbapClient.java`

**REMOVE:** `getToolDefinitionsJson()` method with hardcoded tools

**ADD:** Dynamic tool fetching from backend:

```java
public static String getToolDefinitionsJson() {
    try {
        // Call backend API instead of hardcoding
        String backendUrl = System.getenv("ORICODE_BACKEND_URL");
        if (backendUrl == null) {
            backendUrl = "https://api.oricode.ai";
        }

        URL url = new URL(backendUrl + "/api/tools/definitions?mcp_servers=abap");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("GET");

        BufferedReader in = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        StringBuilder response = new StringBuilder();
        String line;
        while ((line = in.readLine()) != null) {
            response.append(line);
        }
        in.close();

        // Parse and return tools array
        JSONObject json = new JSONObject(response.toString());
        return json.getJSONArray("tools").toString();
    } catch (Exception e) {
        OricodeLogger.getInstance().error("Failed to fetch tool definitions from backend", e);
        return getFallbackToolDefinitions(); // Emergency fallback
    }
}
```

### Phase 3: Update VS Code Extension

**File:** `oricode-vs-code/src/tools/ToolRouter.ts`

**REMOVE:** Hardcoded tool definitions in `getVsCodeToolDefinitions()`

**ADD:** Fetch from backend:

```typescript
private async fetchToolDefinitionsFromBackend(): Promise<ToolDefinition[]> {
  try {
    const response = await fetch(`${this.backendUrl}/api/tools/definitions?mcp_servers=abap,common`);
    const data = await response.json();
    return data.tools;
  } catch (error) {
    this.logger.error('ROUTER', 'Failed to fetch tool definitions from backend', error);
    return this.getFallbackToolDefinitions(); // Emergency fallback
  }
}
```

## Expected Fix (Short-term Workaround)

**Until backend centralization is complete**, Eclipse plugin should:

1. **Fetch complete tool definitions** from MCP server via `ListTools` JSON-RPC call
2. **Forward ALL properties** including nested `fields` array definition
3. **Preserve the `required` array** exactly as defined by MCP server
4. **NOT filter or simplify** complex nested schemas

## Workaround (Temporary)

Hardcode the complete `mcp_create_table` tool definition in `McpAbapClient.java`:

```java
{
  "name": "mcp_create_table",
  "description": "Create a new database table (DDIC) with DDL source code.",
  "input_schema": {
    "type": "object",
    "properties": {
      "table_name": { "type": "string", "description": "Name of the table (e.g., ZTMY_TABLE)" },
      "description": { "type": "string", "description": "Short description" },
      "package": { "type": "string", "description": "Package name" },
      "fields": {
        "type": "array",
        "description": "Array of field definitions",
        "items": {
          "type": "object",
          "properties": {
            "name": { "type": "string", "description": "Field name" },
            "type": { "type": "string", "description": "Field type (e.g., abap.clnt, abap.char, CHAR, domain name)" },
            "length": { "type": "number", "description": "Length for CHAR/NUMC types (optional)" },
            "decimals": { "type": "number", "description": "Decimals for DEC type (optional)" },
            "key": { "type": "boolean", "description": "Is this a key field? (optional, default false)" },
            "notNull": { "type": "boolean", "description": "Is this field NOT NULL? (optional, default false)" }
          },
          "required": ["name", "type"]
        }
      },
      "transport": { "type": "string", "description": "Transport request (required for non-$TMP)" }
    },
    "required": ["table_name", "description", "package", "fields"]
  }
}
```

## Testing

After fix, verify:

1. **Tool definition includes fields:**
   ```bash
   cat c:\temp\oricode_request.json | jq '.tools[] | select(.name=="mcp_create_table") | .input_schema.properties.fields'
   ```
   Should output the fields array definition, NOT null.

2. **Claude includes fields in tool calls:**
   ```bash
   cat c:\temp\oricode_request.json | jq '.messages[-1].content[] | select(.type=="tool_use") | .input.fields'
   ```
   Should output the fields array with field objects.

3. **MCP server receives fields:**
   ```bash
   tail -f c:/temp/oricode_debug.log | grep "args?.fields"
   ```
   Should show `type: object` or `type: array`, NOT `type: undefined`.

## Related Files

- **MCP Server:** [c:\Users\teddy.layani\eclipse-workspace\mcp-abap-adt-crud\src\index.ts](c:\Users\teddy.layani\eclipse-workspace\mcp-abap-adt-crud\src\index.ts)
- **Eclipse Plugin:** `**/McpAbapClient.java` (need to glob)
- **Request Log:** [c:\temp\oricode_request.json](c:\temp\oricode_request.json)
- **Debug Log:** [c:/temp/oricode_debug.log](c:/temp/oricode_debug.log)

## Status

- ✅ **MCP Server:** Tool definition is correct with `fields` array
- ✅ **Parameter Mapper:** Regex-based normalization for `key`/`keyField` properties implemented
- ❌ **Eclipse Plugin:** Stripping `fields` property from tool definition before sending to Claude
- ⏳ **Needs Fix:** Update Eclipse plugin to forward complete tool definitions

## 🎓 Key Lessons

### Why This Keeps Happening

Every time we add/update a tool, we must manually sync:
1. ✅ MCP server tool definition (source of truth)
2. ❌ Eclipse `McpAbapClient.java` hardcoded definitions
3. ❌ VS Code `ToolRouter.ts` hardcoded definitions
4. ❌ Documentation

**Result:** Things get out of sync, parameters go missing, bugs appear.

### The Real Solution

**STOP HARDCODING TOOL DEFINITIONS!**

```
❌ WRONG: Hardcode in 3 places
✅ RIGHT: Backend API as single source
```

Tool definitions should:
- 📍 **Live in ONE place** (backend or MCP server)
- 🔄 **Be fetched dynamically** by all clients
- 🚫 **NEVER be hardcoded** in Eclipse/VS Code
- 📝 **Be auto-documented** from the single source

### Migration Priority

1. **Immediate (this bug):** Fix Eclipse to fetch from MCP or backend
2. **Short-term:** Create backend `/api/tools/definitions` endpoint
3. **Medium-term:** Migrate Eclipse and VS Code to use backend API
4. **Long-term:** Auto-generate documentation from backend tool registry

## Next Steps

### Immediate Fix (Stop the Bleeding)
1. Update Eclipse `McpAbapClient.java` to call MCP ListTools
2. OR hardcode the complete `mcp_create_table` definition with `fields` property
3. Test table creation

### Long-term Fix (Prevent Future Issues)
1. Implement backend tool registry (`/api/tools/definitions`)
2. Update Eclipse to fetch from backend
3. Update VS Code to fetch from backend
4. Remove ALL hardcoded tool definitions
5. Add tests to ensure tool definitions stay in sync

## 🤖 AI Assistant Quick Commands

When debugging tool definition issues, use these grep commands:

```bash
# Find ALL tool definitions across codebase
grep -rn "mcp_create_table\|CreateTable" --include="*.ts" --include="*.java"

# MCP Server - Source of truth
grep -n "DDIC CREATE TOOLS" mcp-abap-adt-crud/src/index.ts

# Eclipse - DDIC tools (mcp_create_table, mcp_create_domain)
grep -n "mcp_create_table" oricode-poc/src/com/oricode/eclipse/ai/tools/EclipseTools.java

# Eclipse - MCP tools (mcp_search, mcp_get_class)
grep -n "getToolDefinitionsJson" oricode-poc/src/com/oricode/eclipse/ai/mcp/McpAbapClient.java

# VS Code - Tool definitions
grep -n "TOOL DEFINITIONS" oricode-vs-code/src/tools/ToolRouter.ts

# Find missing 'fields' property in tool definitions
grep -B5 -A20 "mcp_create_table" --include="*.java" --include="*.ts" | grep -A15 "input_schema"
```

**Key Files to Check:**
- `mcp-abap-adt-crud/src/index.ts` - MCP Server (SOURCE OF TRUTH)
- `oricode-poc/.../tools/EclipseTools.java` - Eclipse DDIC tool definitions
- `oricode-poc/.../mcp/McpAbapClient.java` - Eclipse MCP tool definitions
- `oricode-poc/.../api/OricodeApiClient.java` - Eclipse tool INPUT FORWARDING (extracts params from Claude and sends to MCP)
- `oricode-vs-code/src/tools/ToolRouter.ts` - VS Code tools

**CRITICAL:** OricodeApiClient.java is where tool inputs are FORWARDED to MCP. If a parameter is defined in EclipseTools.java but not extracted in OricodeApiClient.java, it will NOT reach the MCP server!
