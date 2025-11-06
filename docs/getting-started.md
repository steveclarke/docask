---
title: Getting Started with DocAsk
description: Walkthrough for setting up and running the DocAsk CLI locally.
---

# Getting Started

Welcome to **DocAsk**, a lightweight CLI that keeps an OpenAI vector store in
sync with the docs in your repository. This quick guide walks you through the
first-time setup.

## Prerequisites

- Ruby 3.2 or newer (DocAsk is developed against Ruby 3.4)
- Bundler with the `ruby-openai` and `listen` gems
- An OpenAI API key with access to the Assistants API

## Installation

```bash
bundle add ruby-openai listen
cp docask docask.local && chmod +x docask.local
```

The second line is optional, but many developers like keeping a local wrapper
so they can add custom behaviour without touching the tracked CLI.

## Configure Your Key

DocAsk checks for `OPENAI_ACCESS_TOKEN` first, then `OPENAI_API_KEY`. Export one
of those in your shell profile:

```bash
export OPENAI_API_KEY=sk-example123
```

Reload the shell (or run `source ~/.zshrc`) so the env var is available to the
CLI.

## Bootstrap a Vector Store

```bash
./docask init
```

DocAsk prints the new vector store ID and writes it back into `docask.yml` for
later use. If you want a different name, set `DOCASK_NAME` before running the
command.

## Sync Documentation

After the vector store exists, mirror the latest docs:

```bash
./docask sync
```

DocAsk hashes each file, uploads new or changed content to the vector store, and
caches metadata in `.docask_state.json`. You can rerun the command whenever you
update docs—it is incremental and only re-uploads modified files.

## Watch for Changes

To automatically re-index while editing:

```bash
./docask watch docs
```

The watcher debounces rapid edits so the API is not spammed with uploads.

## Ask Questions

Once the vector store has content, you can query it via OpenAI Assistants:

```bash
./docask ask "How do I run the sample project?"
```

DocAsk creates a temporary assistant backed by your vector store, submits the
question, polls for completion, and then prints the response. For production
usage you could pre-create a reusable assistant via the API instead of relying
on the ephemeral workflow.

## Next Steps

- Add more Markdown or PDF files under `docs/`
- Plug DocAsk into CI to keep your vector store fresh
- Experiment with additional metadata, like tagging snippets with owners

Happy experimenting!

