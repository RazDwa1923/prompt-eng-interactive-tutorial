# Claude API Playground

This started as Anthropic's official prompt-engineering course and has since grown into my personal playground for learning the Claude API and Claude Code. It's no longer just "the tutorial" — it's a few loosely related learning projects living side by side in one repo.

## What's in here

- **`Anthropic 1P/`** — Where this repo started: Anthropic's official prompt-engineering course, run against the direct Anthropic API. Structured as 9 chapters (basic prompt structure → clear & direct → roles → separating data/instructions → formatting output → precognition/step-by-step → few-shot examples → avoiding hallucinations → complex prompts) plus an appendix (chaining prompts, tool use, search & retrieval). Each notebook has an "Example Playground" section for experimenting.

- **`AmazonBedrock/`** — The same course content, bundled by Anthropic as a Bedrock-flavored port (two implementations: `anthropic/` via the Anthropic SDK, `boto3/` via raw boto3, plus a CloudFormation template for a Bedrock workshop stack). Not something I've actually used — kept as unused reference material since it shipped with the original course.

- **`Claude_API_Training/`** — A separate, more advanced course I'm working through on my own, covering tool use, iterative chat, the text-edit and web-search tools, and building RAG pipelines. Independent of the two directories above.

- **`MCP App/`** — A standalone Python CLI app I added to learn the Model Context Protocol (MCP): how MCP clients and servers work together, and how that fits into the Claude API's tool-use loop. See `MCP App/README.md` for setup/run instructions and `MCP App/LESSONS.md` for a walkthrough of what it demonstrates and what to try.

## Setup

There's no shared setup across the repo — each directory manages its own dependencies and its own `.env` (all git-ignored). See the relevant subdirectory for specifics; `MCP App/README.md` has the most detail since it's the one runnable application rather than a set of notebooks.

See `CLAUDE.md` for a more detailed architecture/commands reference if you're using Claude Code in this repo.
