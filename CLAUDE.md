# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is **not one application** — it's four independent, self-contained learning projects living in one git repo. They don't import from each other and don't share dependencies or config. Before making a change, identify which of the four you're in; conventions differ between them.

| Directory | What it is |
|---|---|
| `Anthropic 1P/` | Anthropic's official prompt-engineering course notebooks, chapters 1–9 + appendix, using the direct Anthropic API. |
| `AmazonBedrock/` | The same course content, adapted for AWS Bedrock, in two parallel implementations (see below) plus a CloudFormation workshop stack. |
| `Claude_API_Training/` | A separate, more advanced set of numbered lessons (`Lesson 1.ipynb` … `Lesson 18.ipynb`) covering tool use, iterative chat, the text-edit and web-search tools, and building RAG pipelines. |
| `MCP App/` | The only non-notebook project: a standalone Python CLI app demonstrating an MCP (Model Context Protocol) client/server chat application. |

There is no root-level dependency manifest, build system, or test suite — each subproject manages its own environment.

## Commands

### MCP App (`MCP App/`)

The only project with an actual run/build step.

```bash
cd "MCP App"
uv venv && source .venv/bin/activate
uv pip install -e .
uv run main.py                    # start the CLI chat app
uv run main.py other_server.py    # optionally mount additional MCP servers
```

Without `uv`: `pip install anthropic python-dotenv prompt-toolkit "mcp[cli]==1.8.0"` then `python main.py`. Requires a `.env` in `MCP App/` with `ANTHROPIC_API_KEY` and `CLAUDE_MODEL` set (`USE_UV` controls how the app spawns its own `mcp_server.py` subprocess, independent of how you launch `main.py` itself).

To inspect `mcp_server.py` directly (list/call its tools, resources, prompts without going through the CLI or Claude):

```bash
npx @modelcontextprotocol/inspector uv run "MCP App/mcp_server.py"
```

There is no lint or test suite for this project.

### Notebooks (`Anthropic 1P/`, `Claude_API_Training/`, `AmazonBedrock/`)

Open and run with Jupyter or DataSpell. Each notebook directory expects its own `.env` file (git-ignored) with `ANTHROPIC_API_KEY` (`Anthropic 1P/` and `Claude_API_Training/` also read `MODEL_NAME`). `Anthropic 1P/` notebooks self-install their one dependency in-cell (`!pip install anthropic`); there is no shared requirements file for these two directories.

`AmazonBedrock/` has its own `requirements.txt` (`pip install -r AmazonBedrock/requirements.txt`) and uses AWS credentials (boto3/awscli) rather than an Anthropic API key for the `AmazonBedrock/boto3/` notebooks — the `AmazonBedrock/anthropic/` notebooks call Bedrock through Anthropic's SDK instead.

## Architecture: `MCP App/`

This is the one project where understanding the design requires reading across multiple files.

**Request flow:** `main.py` wires up a `Chat` and starts the CLI. `core/chat.py` runs the agent loop: call Claude, and if `stop_reason == "tool_use"`, execute the tool via MCP and feed the result back in — repeat until Claude returns plain text.

- `mcp_server.py` — the MCP server (a subprocess, speaking MCP over stdio). Defines the three MCP primitives, which are *not interchangeable*:
  - **Tools** (`read_doc_contents`, `edit_document`) — offered to Claude via `tools=[...]`; Claude decides when to call them.
  - **Resources** (`docs://documents`, `docs://documents/{doc_id}`) — fetched directly by the client, never shown to Claude as callable; this is what powers `@doc_id` mentions.
  - **Prompts** (`format`) — server-defined message templates; `/command doc_id` fetches one and injects its messages directly into the conversation.
- `mcp_client.py` — wraps an MCP `ClientSession` (via the `mcp` SDK's `stdio_client`) around one server subprocess. `main.py` creates one `MCPClient` per server (the built-in doc server, plus any passed as CLI args).
- `core/tools.py` (`ToolManager`) — the bridge between MCP and Claude: flattens every connected client's `list_tools()` into Anthropic's tool schema, and on a `tool_use` response, finds the right client (`_find_client_with_tool`) and routes the call to it.
- `core/cli_chat.py` (`CliChat`) — adds `@doc_id` resource injection and `/command` prompt handling on top of the base `Chat` loop.
- `core/cli.py` — terminal UX only (prompt_toolkit autocomplete for `/` and `@`); not part of the MCP integration itself.

Known gotcha: `ToolManager.execute_tool_requests` (`core/tools.py`) references `tool_output` inside its `except` block before it's necessarily assigned — a tool call that raises before `call_tool` returns will hit `UnboundLocalError` instead of surfacing the real error.

A companion walkthrough of this project (main lessons, what to try when running it) exists at `MCP App/LESSONS.md`.
