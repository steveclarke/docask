# DocAsk Sample Project

Repo with a ready-to-run DocAsk CLI, sample documents, and a template config for
trying OpenAI vector stores.

## Prerequisites

- Ruby 3.2+ with Bundler
- Gems: `ruby-openai`, `listen`
- OpenAI API key exposed as `OPENAI_ACCESS_TOKEN` or `OPENAI_API_KEY`

Install dependencies:

```bash
bundle install
```

## Configure

1. Copy the template config (tracked as `docask.yml.template`):

   ```bash
   cp docask.yml.template docask.yml
   ```

   `docask.yml` stays untracked so you can store vector store ids locally.

2. Export your API key (add to `~/.zshrc` for persistence):

   ```bash
   export OPENAI_API_KEY=sk-example123
   ```

## Usage

```bash
./docask init        # creates a vector store and records the id in docask.yml
./docask sync        # uploads docs/**/*.{md,mdx,markdown,pdf} (with a repo-wide fallback)
./docask watch docs  # optional: live updates while editing
./docask ask "How do retries work?"
```

Markdown and PDF samples already live in `docs/`, so you can run these commands
immediately.

## Notes

- Uploads are incremental; `.docask_state.json` caches file hashes.
- `docask sync` warns if it has to fall back to a broad glob (e.g., when `docs/`
  is empty).
- Delete `.docask_state.json` to force a fresh upload of every file.

Happy experimenting!

