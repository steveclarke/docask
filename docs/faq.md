# DocAsk FAQ

This document answers common questions about the sample DocAsk setup. Use it as
a template for your own knowledge base content.

## What does DocAsk store in the vector database?

DocAsk uploads the full text of each Markdown or PDF file that matches your
`includes` globs. The OpenAI Assistants API then chunks and embeds that text.

## How does DocAsk avoid re-uploading unchanged files?

The script keeps a `.docask_state.json` file that maps pathnames to the last
uploaded SHA-256 hash. Before uploading, DocAsk recomputes the hash and skips
the file if it matches the cached value.

## What happens if a file is deleted locally?

When the watcher sees a deletion, DocAsk removes the corresponding file from the
vector store (if it exists in the cache). If the file was never uploaded, the
deletion is ignored.

## Can I change the default globs?

Yes. Edit `docask.yml` and tweak the `includes` and `excludes` arrays. On the
next run, DocAsk merges the YAML with built-in defaults. If the custom includes
produce no files, DocAsk falls back to a broad `**/*.md`-style glob and warns in
the console so you know defaults were used.

## Does DocAsk support other file types?

Out of the box it targets Markdown-like extensions and PDFs. You can modify the
`indexable?` helper in the CLI to add more extensions, or preprocess files into
a supported format before syncing.

## How do retries work?

Any API call is wrapped in `with_retries`, which uses exponential backoff with
full jitter. The defaults allow five attempts, doubling the delay until it hits
an 8-second cap. These values can be overridden in `docask.yml`.

## Is DocAsk production ready?

This sample script is meant for experimentation. For production use, consider
adding structured logging, error reporting, and stricter validation of YAML
configuration files.

