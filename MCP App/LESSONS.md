# MCP App — Guide to the Main Lessons

This app is a fully working reference implementation (the README's "TODOs" are
already done in the code). Treat it as a worked example of how the Model
Context Protocol (MCP) connects to the Claude API, not a template you need to
finish.

## 1. The architecture, file by file

| File | Role |
|---|---|
| `mcp_server.py` | The MCP **server**. Defines tools, resources, and prompts over a fake "document store" (`docs` dict). Runs as a separate process, speaking MCP over stdio. |
| `mcp_client.py` | The MCP **client**. Launches the server as a subprocess and wraps the MCP session (`list_tools`, `call_tool`, `list_prompts`, `get_prompt`, `read_resource`). |
| `core/claude.py` | Thin wrapper around the **Anthropic API** (`client.messages.create`). Knows nothing about MCP. |
| `core/tools.py` | The bridge: converts MCP `Tool` objects into Claude's `tools=[...]` schema, and routes Claude's `tool_use` blocks back to the right MCP client. |
| `core/chat.py` | The **agent loop**: calls Claude, and if it asks for a tool, executes it via MCP and feeds the result back — repeats until Claude gives a final text answer. |
| `core/cli_chat.py` | Adds document-aware features on top of the loop: `@doc_id` mentions (resources) and `/command` (prompts). |
| `core/cli.py` | Terminal UX only — autocomplete for `/` and `@`. Not part of the MCP lesson. |
| `main.py` | Wires everything together and starts the CLI. |

**Data flow for a normal message:**
`main.py` → `CliChat._process_query` → `Chat.run` loop → `Claude.chat` →
(if `tool_use`) `ToolManager.execute_tool_requests` → `MCPClient.call_tool` →
back into the message history → loop again until plain text.

## 2. The three MCP primitives — where each one lives

- **Tools** (`mcp_server.py:27-56`) — `read_doc_contents`, `edit_document`.
  These are the only things *Claude itself* can decide to call. Claude sees
  them via `ToolManager.get_all_tools`, which just flattens `inputSchema`
  from every connected client into Anthropic's tool format.
- **Resources** (`mcp_server.py:59-68`) — `docs://documents` (list of ids) and
  `docs://documents/{doc_id}` (contents). These are *not* offered to Claude
  as tools; the CLI fetches them directly (`CliChat._extract_resources`) and
  stuffs the content into the prompt when you type `@doc_id`.
- **Prompts** (`mcp_server.py:71-90`) — `format`. A prompt is a
  server-defined message template with arguments. `/format deposition.md`
  calls `get_prompt` on the server and injects the returned messages
  directly into the conversation, rather than sending a fresh user message.

The key distinction to internalize: **tools are for the model, resources and
prompts are for the client/user**. That split is the main design lesson of
MCP.

## 3. The agent loop (`core/chat.py`)

```python
while True:
    response = claude.chat(messages, tools=...)
    if response.stop_reason == "tool_use":
        # execute tool(s) via MCP, append result, loop again
    else:
        return final text
```

This is the canonical "agentic tool-use loop" pattern for the Claude API —
worth understanding independent of MCP, since it's the same shape you'd use
with any tool source (not just MCP servers).

## 4. What to actually try when running the app

Run it with `uv run main.py` (or `python main.py`), then:

1. **Plain chat** — ask something with no `@` or `/`. Confirms the basic
   request/response path with no tools involved.
2. **`@deposition.md tell me about this doc`** — watch that no tool call
   happens; the content is injected as context (resource, not tool).
3. **`/format deposition.md`** — watch the model call `edit_document` (a real
   tool_use round-trip) and then return the reformatted text. This is the
   only path that exercises the full agent loop with a real tool call.
4. **Ask a question that requires reading a doc without `@`**, e.g. "what
   does spec.txt say?" — Claude should choose to call `read_doc_contents`
   itself. Compare this to step 2 to see tool-choice vs. injected-context.
5. **Tab-completion** — type `/` then Tab, and `@` then Tab, to see prompts
   and resources being listed live from the server (`CliApp.refresh_*`).
6. **Multiple servers** — `main.py` accepts extra server scripts as CLI args
   (`uv run main.py other_server.py`) and mounts each as its own
   `MCPClient` in the `clients` dict. `ToolManager._find_client_with_tool`
   is what lets Claude use tools from *any* connected server transparently.

## 5. Gotchas / things that will bite you

- `.env` needs both `ANTHROPIC_API_KEY` and `CLAUDE_MODEL` (currently set to
  `claude-sonnet-4-5`) — `main.py` asserts on both.
- `USE_UV` in `.env` controls how the **doc_client** subprocess is spawned
  (`uv run mcp_server.py` vs `python mcp_server.py`); it does not affect how
  *you* launch `main.py`.
- Tool errors: `ToolManager.execute_tool_requests` has a latent bug — in the
  `except` branch it references `tool_output` before it's guaranteed to be
  assigned, so a tool call that throws *before* `call_tool` returns will
  raise `UnboundLocalError` instead of reporting the real error. Worth
  noticing if you extend `mcp_server.py` with tools that can fail.
- `docs` in `mcp_server.py` is an in-memory dict — edits via `/format` don't
  persist across restarts.

## 6. Good extension exercises

- Add a new doc to `docs` and confirm it shows up in `@`/`/` autocomplete
  without any other code changes (proves resources are dynamic).
- Add a `summarize` prompt (the README mentions it) mirroring `format`.
- Add a second tool, e.g. `delete_document`, and watch it appear in Claude's
  tool list automatically — nothing in `core/tools.py` needs to change.
- Write a second, independent MCP server (e.g. a calculator) and pass it as
  an extra `sys.argv` entry to see multi-server tool routing in action.
