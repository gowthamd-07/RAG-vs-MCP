# Mini Project: Personal Note-Taking Agent via Slack MCP

> **Concept:** A conversational AI agent that detects when you mention something to remember — todos, reminders, meeting notes, ideas — and automatically posts it to your private Slack channel, acting as a persistent personal notepad you never have to open.

---

## Problem

You're in the middle of a coding session or a conversation with your AI assistant, and you think: *"I need to meet with Karthick Rai Sake tomorrow"* or *"Remind me to review the PR for auth-service."* Today, you either:

1. Context-switch to Slack/Notion/Todoist to write it down (and lose flow)
2. Tell yourself you'll remember (and forget)

There's no way to just *say it* in your current workflow and have it reliably captured somewhere you'll see it later.

---

## Solution

An MCP-powered agent that:

1. **Detects intent** — Recognizes phrases like "Todo:", "Remind me:", "Note:", "I need to...", "Don't forget..." in your natural conversation
2. **Extracts the note** — Parses out the actionable content
3. **Posts to your private Slack channel** — Sends a well-formatted message to a dedicated `#my-notes` (or similar) channel only you are in
4. **Confirms back** — Responds with a brief confirmation so you know it's captured

---

## Architecture

```
┌──────────────────────┐
│   You (in Cursor)    │
│                      │
│  "Todo: meet with    │
│   karthick-rai-sake" │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   AI Agent (LLM)     │
│                      │
│  1. Detect intent    │
│  2. Extract note     │
│  3. Categorize       │
│  4. Call Slack MCP   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Slack MCP Server   │
│                      │
│  Tools:              │
│  - send_message()    │
│  - list_channels()   │
│  - add_reaction()    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Slack: #my-notes    │
│                      │
│  ✅ Todo: Meet with  │
│  karthick-rai-sake   │
│  📅 2026-03-12       │
└──────────────────────┘
```

---

## Implementation

### Step 1: Set Up a Private Slack Channel

Create a private channel in your Slack workspace for personal notes:

- **Channel name:** `#raghu-notes` (or any name you prefer)
- **Purpose:** Personal note-taking channel managed by AI agent
- **Members:** Only you (and the Slack bot)

### Step 2: Create a Slack App & Bot Token

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Name it something like `NoteKeeper Bot`
3. Under **OAuth & Permissions**, add these **Bot Token Scopes**:
   - `chat:write` — post messages to channels
   - `channels:read` — list channels (if public)
   - `groups:read` — list private channels
   - `reactions:write` — add emoji reactions for visual feedback
   - `chat:write.customize` — customize bot name/icon per message (optional)
4. **Install to Workspace** → copy the **Bot User OAuth Token** (`xoxb-...`)
5. **Invite the bot** to your `#raghu-notes` channel: `/invite @NoteKeeper Bot`

### Step 3: Set Up the Slack MCP Server

Use the official or community Slack MCP server. Below is a custom lightweight implementation:

#### `slack-mcp-server/index.ts`

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { WebClient } from "@slack/web-api";

const slack = new WebClient(process.env.SLACK_BOT_TOKEN);
const NOTES_CHANNEL = process.env.SLACK_NOTES_CHANNEL_ID!;

const server = new Server(
  { name: "slack-notes", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "post_note",
      description:
        "Post a personal note/todo/reminder to the user's private Slack notes channel. " +
        "Use this whenever the user mentions something they want to remember, " +
        "a todo item, a reminder, a meeting to schedule, or any actionable note.",
      inputSchema: {
        type: "object",
        properties: {
          note: {
            type: "string",
            description: "The note content to post",
          },
          category: {
            type: "string",
            enum: ["todo", "reminder", "meeting", "idea", "note"],
            description: "Category of the note for emoji tagging",
          },
          priority: {
            type: "string",
            enum: ["high", "normal", "low"],
            description: "Priority level. Default: normal",
          },
        },
        required: ["note", "category"],
      },
    },
    {
      name: "list_recent_notes",
      description:
        "Retrieve recent notes from the Slack notes channel to review what's been captured.",
      inputSchema: {
        type: "object",
        properties: {
          limit: {
            type: "number",
            description: "Number of recent notes to fetch (default: 10)",
          },
        },
      },
    },
  ],
}));

const CATEGORY_EMOJI: Record<string, string> = {
  todo: "✅",
  reminder: "🔔",
  meeting: "📅",
  idea: "💡",
  note: "📝",
};

const PRIORITY_EMOJI: Record<string, string> = {
  high: "🔴",
  normal: "",
  low: "🔵",
};

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "post_note") {
    const { note, category, priority = "normal" } = args as {
      note: string;
      category: string;
      priority?: string;
    };

    const emoji = CATEGORY_EMOJI[category] || "📝";
    const priorityTag = PRIORITY_EMOJI[priority] || "";
    const timestamp = new Date().toLocaleString("en-IN", {
      timeZone: "Asia/Kolkata",
      dateStyle: "medium",
      timeStyle: "short",
    });

    const message = [
      `${emoji} *${category.charAt(0).toUpperCase() + category.slice(1)}* ${priorityTag}`.trim(),
      `> ${note}`,
      `_Captured: ${timestamp}_`,
    ].join("\n");

    const result = await slack.chat.postMessage({
      channel: NOTES_CHANNEL,
      text: message,
      unfurl_links: false,
    });

    return {
      content: [
        {
          type: "text",
          text: `Note posted to Slack. Message timestamp: ${result.ts}`,
        },
      ],
    };
  }

  if (name === "list_recent_notes") {
    const limit = (args as { limit?: number }).limit || 10;

    const result = await slack.conversations.history({
      channel: NOTES_CHANNEL,
      limit,
    });

    const notes = (result.messages || [])
      .map((msg) => msg.text)
      .join("\n---\n");

    return {
      content: [{ type: "text", text: notes || "No notes found." }],
    };
  }

  return { content: [{ type: "text", text: `Unknown tool: ${name}` }] };
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main();
```

#### `slack-mcp-server/package.json`

```json
{
  "name": "slack-notes-mcp",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "@slack/web-api": "^7.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/node": "^20.0.0"
  }
}
```

#### `slack-mcp-server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["index.ts"]
}
```

### Step 4: Configure MCP in Cursor

Add the Slack MCP server to your Cursor MCP configuration:

#### `.cursor/mcp.json`

```json
{
  "mcpServers": {
    "slack-notes": {
      "command": "node",
      "args": ["<path-to>/slack-mcp-server/dist/index.js"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-your-bot-token-here",
        "SLACK_NOTES_CHANNEL_ID": "C0XXXXXXX"
      }
    }
  }
}
```

> **Finding your channel ID:** Open Slack in a browser, navigate to your `#raghu-notes` channel. The URL will be `.../archives/C0XXXXXXX` — that `C0XXXXXXX` is your channel ID.

### Step 5: Create a Cursor Rule for Intent Detection

Create a rule that teaches the agent to recognize note-taking intent and call the MCP tool.

#### `.cursor/rules/note-keeper.mdc`

```markdown
---
description: Detect personal notes, todos, and reminders and post them to Slack
globs:
alwaysApply: true
---

## Note-Keeping Agent Behavior

When the user says something that sounds like a personal note, todo, reminder,
meeting plan, or idea they want to remember, ALWAYS use the `post_note` MCP tool
to save it to their Slack notes channel.

### Trigger phrases (non-exhaustive — use judgment)

- "Todo: ..." / "TODO: ..."
- "Remind me to ..."
- "I need to ..." / "I have to ..."
- "Don't forget ..."
- "Note to self: ..."
- "Meeting with ..."
- "I should ..."
- "Remember to ..."
- "Idea: ..."
- "Follow up on ..."
- "Schedule ..." / "Set up a meeting ..."
- "I have something to tell" / "Keep a note of ..."

### Category mapping

| User says                                      | Category    |
| ---------------------------------------------- | ----------- |
| "Todo: ...", "I need to ...", "I should ..."   | `todo`      |
| "Remind me ...", "Don't forget ..."            | `reminder`  |
| "Meet with ...", "Schedule a call ..."         | `meeting`   |
| "Idea: ...", "What if we ..."                  | `idea`      |
| Everything else                                | `note`      |

### Priority mapping

| Signal                                          | Priority |
| ----------------------------------------------- | -------- |
| "urgent", "ASAP", "critical", "important", "!" | `high`   |
| No signal                                       | `normal` |
| "when I get time", "low priority", "eventually" | `low`    |

### Behavior rules

1. Extract the **core note** — strip conversational fluff, keep the actionable content
2. Pick the **category** based on intent
3. Pick the **priority** based on urgency signals (default: `normal`)
4. Call `post_note` with the extracted note, category, and priority
5. Confirm to the user briefly: "Got it, posted to your notes channel."
6. Then continue the conversation naturally — don't dwell on the note

### Examples

**User:** "Todo: meet with karthick-rai-sake about the API redesign"
**Agent action:** `post_note(note: "Meet with karthick-rai-sake about the API redesign", category: "todo")`
**Agent says:** "Noted — posted to your Slack notes channel. Now, what were we working on?"

**User:** "Oh remind me to review the PR for payments-service, it's urgent"
**Agent action:** `post_note(note: "Review the PR for payments-service", category: "reminder", priority: "high")`
**Agent says:** "Done, captured as a high-priority reminder."

**User:** "I have something to tell — I need to follow up with the platform team on the infra migration"
**Agent action:** `post_note(note: "Follow up with the platform team on the infra migration", category: "todo")`
**Agent says:** "Got it, saved to your notes."
```

---

## How It Works End-to-End

Here's the complete flow when you say **"Todo: meet with karthick-rai-sake"**:

```
1. You type in Cursor:
   "Todo: meet with karthick-rai-sake"

2. The Cursor Rule (note-keeper.mdc) triggers — agent recognizes intent

3. Agent calls the MCP tool:
   post_note(
     note: "Meet with karthick-rai-sake",
     category: "meeting",
     priority: "normal"
   )

4. Slack MCP Server receives the call and posts to #raghu-notes:
   ┌──────────────────────────────────────────┐
   │ 📅 *Meeting*                             │
   │ > Meet with karthick-rai-sake            │
   │ _Captured: 12 Mar 2026, 3:45 pm_        │
   └──────────────────────────────────────────┘

5. Agent confirms:
   "Got it, posted to your notes channel."

6. You continue working — zero context switch.
```

---

## Setup Checklist

- [ ] Create private Slack channel (`#raghu-notes` or similar)
- [ ] Create Slack App with bot token and required scopes
- [ ] Invite bot to the private channel
- [ ] Clone/create the `slack-mcp-server` directory
- [ ] `npm install && npm run build` in the server directory
- [ ] Add `SLACK_BOT_TOKEN` and `SLACK_NOTES_CHANNEL_ID` to `.cursor/mcp.json`
- [ ] Add `.cursor/rules/note-keeper.mdc` for intent detection
- [ ] Test: say "Todo: test note" and verify it appears in Slack

---

## What It Looks Like in Slack

```
#raghu-notes

📅 *Meeting*
> Meet with karthick-rai-sake
_Captured: 12 Mar 2026, 3:45 pm_

✅ *Todo*
> Review the PR for payments-service auth changes
_Captured: 12 Mar 2026, 4:10 pm_

💡 *Idea* 🔵
> What if we use WebSockets instead of polling for the dashboard?
_Captured: 12 Mar 2026, 5:30 pm_

🔔 *Reminder* 🔴
> Submit expense report before EOD Friday
_Captured: 13 Mar 2026, 10:00 am_
```

---

## Optional Enhancements

| Enhancement | Description |
|---|---|
| **Thread grouping** | Group notes by date using Slack threads — one parent message per day, notes as thread replies |
| **Slash command** | Add `/notes today` to review today's notes directly in Slack |
| **Daily digest** | Scheduled job that posts a summary of yesterday's unresolved todos each morning |
| **Mark complete** | React with ✅ on a todo in Slack to mark it done; agent can query for open todos |
| **Search notes** | `list_recent_notes` tool already supports this; add keyword filtering |
| **Calendar integration** | For meeting notes, auto-create a Google Calendar event via a Calendar MCP server |

---

## Cost & Complexity

| Dimension | Detail |
|---|---|
| **Slack API** | Free tier covers all needed endpoints |
| **LLM cost** | ~$0.001 per note (just intent detection + extraction) |
| **MCP servers** | 1 (Slack) |
| **Build time** | ~2-3 hours for full setup |
| **Complexity** | Low-Medium — single MCP server, rule-based intent detection |

---

## Q&A — Design Decisions & Deep Dives

---

### Q: Did we add our own MCP server for Slack?

Yes — we wrote a **custom Slack MCP server** (`slack-mcp-server/index.ts`) rather than using a pre-built community one. It is purpose-built and exposes exactly two tools:

- `post_note` — posts a categorized, emoji-tagged, IST-timestamped note to your private channel
- `list_recent_notes` — fetches recent messages from that channel

Built using `@modelcontextprotocol/sdk` (official MCP SDK) and `@slack/web-api` (Slack's official Node client). The advantage over a generic Slack MCP server is that the message formatting (category emoji, priority tag, timestamp) is baked in, and configuration is minimal.

---

### Q: How does Cursor use `note-keeper.mdc` as a rule?

Cursor looks for rule files in `.cursor/rules/` at the workspace root. Any `.mdc` file placed there is automatically picked up.

**The frontmatter controls when the rule activates:**

```markdown
---
description: Detect personal notes, todos, and reminders and post them to Slack
globs:
alwaysApply: true
---
```

| Mode | Behavior |
|---|---|
| `alwaysApply: true` | Active in every conversation — what we use |
| `globs: ["**/*.ts"]` | Only activates when a matching file is open |
| Description only | Cursor's AI decides when to apply it |

Since `alwaysApply: true` is set, the agent watches for todo/reminder phrases regardless of which file you have open. Cursor injects the rule content as part of the agent's system prompt, so the trigger phrases, category/priority mappings, and behavior rules are always loaded.

**To create the rule in Cursor:**
- Option A: `Cmd+Shift+P` → "New Cursor Rule" → paste rule content
- Option B: Manually create `.cursor/rules/note-keeper.mdc`

---

### Q: Where does the server file live, and how does it run?

**File location — anywhere on your machine, e.g.:**

```
~/mcp-servers/slack-notes/
├── index.ts
├── package.json
├── tsconfig.json
└── dist/
    └── index.js   ← compiled output
```

It lives outside any project — at the system level since Cursor uses it globally.

**How it runs — Cursor spawns it automatically as a subprocess:**

Cursor reads `.cursor/mcp.json` and runs `node dist/index.js` as a background child process on startup. You never start it manually.

Communication happens over **stdin/stdout** (no HTTP port). Cursor sends JSON in via stdin, the server responds via stdout — this is the `StdioServerTransport` in the MCP SDK.

| Question | Answer |
|---|---|
| Where does it live? | Anywhere, e.g. `~/mcp-servers/slack-notes/` |
| Started manually? | No — Cursor spawns it on startup |
| Port? | None — communicates via stdin/stdout |
| Stops when? | When Cursor closes |
| After code changes? | Rebuild (`npm run build`) and restart Cursor |

---

### Q: Full end-to-end flow including the Cursor Rule

```
Cursor starts
     │
     ▼
Reads .cursor/mcp.json
Spawns: node dist/index.js   ← MCP server is now running
     │
     ▼
Reads .cursor/rules/note-keeper.mdc
Injects rule content into agent's system prompt   ← agent now knows trigger phrases,
     │                                               category/priority mappings, behavior
     ▼
You say "Todo: meet with karthick-rai-sake"
     │
     ▼
Agent matches phrase against rule triggers
("Todo: ..." → category: "meeting", priority: "normal")
     │
     ▼
Agent decides to call the MCP tool
     │
     ▼
Cursor sends JSON to server's stdin:
  { "tool": "post_note", "args": { "note": "Meet with karthick-rai-sake", "category": "meeting" } }
     │
     ▼
Server calls Slack API → posts to #raghu-notes
     │
     ▼
Server writes JSON to stdout:
  { "content": [{ "type": "text", "text": "Note posted." }] }
     │
     ▼
Cursor reads the response
     │
     ▼
Agent says "Got it, posted to your notes channel."
```

The rule is what bridges natural language to the MCP tool call. Without it, the agent would just respond conversationally and never call `post_note`.

---

### Q: What if I want to build this as a hosted chatbot web app instead of using Cursor?

Instead of Cursor being the chat interface, you build your own web chatbot. The architecture changes — you own everything Cursor was handling automatically.

**Architecture:**

```
┌─────────────────────────┐
│   Chat UI (browser)     │
└────────────┬────────────┘
             │ HTTP
             ▼
┌─────────────────────────┐
│   Python Web Server     │
│   (FastAPI/Flask)       │
│  1. Receive user msg    │
│  2. Call LLM API        │
│  3. LLM decides tool    │
│  4. Call MCP server     │
│  5. Return response     │
└────────────┬────────────┘
             │
     ┌───────┴────────┐
     ▼                ▼
┌─────────┐    ┌─────────────┐
│  OpenAI │    │  MCP Server │
│  API    │    │  (Slack)    │
└─────────┘    └──────┬──────┘
                      ▼
               ┌─────────────┐
               │  Slack API  │
               │  #my-notes  │
               └─────────────┘
```

| Responsibility | In Cursor | In Your Web App |
|---|---|---|
| Chat UI | Cursor's chat panel | You build it (React/HTML) |
| System prompt / rules | `.cursor/rules/*.mdc` | Hardcoded string passed to LLM API |
| Detecting note intent | Cursor injects rule into LLM | Your system prompt does this |
| Calling MCP tools | Cursor handles tool dispatch | Your server handles tool dispatch |
| Running MCP server | Cursor spawns it as subprocess | You run it as a process on your server |

**Hosting options:** Railway, Render, EC2, DigitalOcean. For a hosted web app, prefer running the MCP server as an **HTTP/SSE service** on a separate port rather than stdio, so both processes can communicate over the network.

---

### Q: How does OpenAI actually make calls to the MCP server?

**OpenAI does NOT call your MCP server directly.** OpenAI just returns a tool call (JSON) saying "I want to call this tool with these arguments." Your Python backend intercepts that and calls the MCP server itself.

**The tool calling loop:**

```
User message
     │
     ▼
Python Backend → sends to OpenAI with tool definitions
     │
     ▼
OpenAI returns tool_call (NOT a text reply):
  { "tool": "post_note", "args": { "note": "...", "category": "meeting" } }
     │
     ▼
Python Backend sees tool_call → calls MCP server
     │
     ▼
MCP Server posts to Slack → returns result
     │
     ▼
Python Backend sends tool result BACK to OpenAI
     │
     ▼
OpenAI returns final text reply: "Got it, saved to your notes."
     │
     ▼
Python Backend → Chat UI
```

**Python code pattern:**

```python
tools = [{
    "type": "function",
    "function": {
        "name": "post_note",
        "description": "Post a todo/reminder/meeting note to Slack. "
                       "Call when user says Todo:, Remind me:, Meet with:, etc.",
        "parameters": {
            "type": "object",
            "properties": {
                "note":     { "type": "string" },
                "category": { "type": "string", "enum": ["todo","reminder","meeting","idea","note"] },
                "priority": { "type": "string", "enum": ["high","normal","low"] }
            },
            "required": ["note", "category"]
        }
    }
}]

response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[
        { "role": "system", "content": system_prompt },
        { "role": "user",   "content": user_message }
    ],
    tools=tools
)

if response.choices[0].finish_reason == "tool_calls":
    tool_call = response.choices[0].message.tool_calls[0]
    args = json.loads(tool_call.function.arguments)

    # YOUR code calls the MCP server — OpenAI does not do this
    mcp_result = call_mcp_server(tool_call.function.name, args)

    # Send result back to OpenAI for final natural language reply
    final_response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            { "role": "system", "content": system_prompt },
            { "role": "user",   "content": user_message },
            response.choices[0].message,
            { "role": "tool", "tool_call_id": tool_call.id, "content": mcp_result }
        ],
        tools=tools
    )
    return final_response.choices[0].message.content
```

**In one line:** OpenAI is the **brain** (decides what to call). Your Python backend is the **hands** (actually calls the MCP server).

---

### Q: How does RAG work, and how to set it up here?

RAG (Retrieval-Augmented Generation) means: before sending the user's message to OpenAI, fetch relevant context from a knowledge store and inject it into the prompt. This lets the LLM answer questions about your past notes.

**Use cases in this project:**
- *"What did I note about karthick last week?"* → fetch past notes, LLM summarizes
- *"Do I have any pending meetings?"* → retrieve meeting-category notes
- *"What did I say about the auth-service PR?"* → semantic search over notes

**How RAG works:**

```
User asks: "What meetings do I have pending?"
     │
     ▼
Embed the query → [0.23, -0.41, 0.87, ...]
     │
     ▼
Search Vector DB (semantic similarity)
     │
     ▼
Top 3 matching notes retrieved:
  - "Meet with karthick-rai-sake"
  - "Sync with platform team on infra migration"
  - "Call with design team about dashboard"
     │
     ▼
Inject into LLM prompt:
  "Context: [retrieved notes]. Now answer: What meetings do I have pending?"
     │
     ▼
LLM answers: "You have 3 pending meetings: ..."
```

**When saving a note — also embed + store:**

```python
def save_note_to_vector_db(note: str, category: str, note_id: str):
    embedding = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=note
    ).data[0].embedding

    collection.add(
        ids=[note_id],
        embeddings=[embedding],
        documents=[note],
        metadatas=[{ "category": category, "timestamp": datetime.now().isoformat() }]
    )
```

**When user asks a question — retrieve + inject:**

```python
def retrieve_relevant_notes(query: str, top_k: int = 3):
    query_embedding = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    ).data[0].embedding

    results = collection.query(query_embeddings=[query_embedding], n_results=top_k)
    return results["documents"][0]

def build_prompt_with_context(user_message: str):
    notes = retrieve_relevant_notes(user_message)
    context = "\n".join(f"- {note}" for note in notes)
    return f"""You are a personal assistant with access to the user's past notes.

Relevant notes:
{context}

Use these to answer the user's question. If the user is adding a new note, call the post_note tool."""
```

**Vector DB options:**

| Option | Best for | Hosting |
|---|---|---|
| **ChromaDB** | Local dev, simple setup | In-process, no separate server |
| **Pinecone** | Production | Managed cloud |
| **pgvector** | Already using Postgres | Add extension to existing DB |

**Full picture with RAG:**

```
"Todo: meet with K"  →  MCP → Slack #my-notes
                     →  Embed → store in Vector DB

"What meetings?"     →  Embed query → search Vector DB → top-K notes
                     →  Inject into prompt → OpenAI → answer
```

**Key insight:** RAG is not a separate service — it's two extra steps in your Python backend. Embed on write, retrieve on read, inject into prompt. The LLM does the rest.
