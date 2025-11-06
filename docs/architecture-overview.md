# DocAsk Architecture Overview

This short write-up explains the moving parts inside DocAsk so you know what to
customize when you adapt the script to a real project.

## High-Level Flow

1. **Configuration** – `docask.yml` is merged with `DEFAULT_CFG` constants.
2. **Client Setup** – `OpenAI::Client` is initialized with your API key.
3. **File Discovery** – `expand_globs` resolves include patterns and filters
   with exclude patterns.
4. **Processing** – Each file goes through `upsert_file`, which hashes,
   conditionally uploads, and updates local state.
5. **Watch Mode** – `Listen` monitors a directory, debounces bursts of changes,
   and batches upserts/removals.

## Key Components

### Configuration

- `DEFAULT_CFG` keeps safe defaults for a simple docs repository.
- `load_yaml` and `persist_yaml` provide best-effort persistence for the local
  YAML file without crashing if permissions are missing.

### Retry Strategy

- `with_retries` wraps API calls in exponential backoff with jitter.
- The method reads retry settings from the merged configuration so you can tune
  the behaviour per repository.

### State Tracking

- `.docask_state.json` stores a dictionary of file paths, their SHA-256 hashes,
  and OpenAI file metadata.
- Because uploads are idempotent, the cache is primarily a shortcut to skip
  redundant work.

### Vector Store Integration

- DocAsk uploads files via the Assistants `files.upload` endpoint, then attaches
  them to the current vector store using `vector_store_files.create`.
- Removing files calls the complementary delete endpoint, ignoring errors if
  the remote copy is already gone.

### Listener Lifecycle

- `Listener.to` watches for modifications, additions, and removals.
- Events are queued with an operation type (`:upsert` or `:remove`).
- A timer resets on each change. When the queue settles for `debounce_ms`, a
  batch job pops up to `batch_max` entries and processes them.

## Extensibility Ideas

- Swap in a different embedding backend by replacing the OpenAI calls.
- Support image or code snippets by adding pre-processing hooks before upload.
- Emit stats to Prometheus or another observability stack after each batch.

Keep iterating on this sample as you explore embeddings workflows.

