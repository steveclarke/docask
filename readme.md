# DocAsk – Watcher + Debounce + Retry + YAML Config (Ruby)

A tiny, repo‑local CLI to ask AI about your docs, plus a file watcher that re‑indexes on change. This version adds:

- **Debounce**: batch rapid file changes before uploading
- **Retry/Backoff**: resilient uploads/deletes with jitter
- **YAML Config**: per‑repo config for globs, ignores, debounce, batch size

> Prereqs: `ruby-openai`, `listen` gems; OpenAI key in `OPENAI_ACCESS_TOKEN` (or `OPENAI_API_KEY`).

---

## 1) Install

```bash
bundle add ruby-openai listen
export OPENAI_ACCESS_TOKEN=sk-...  # or export OPENAI_API_KEY in your shell env/direnv
```

---

## 2) Optional config: `docask.yml`

Create at repo root:

```yaml
# docask.yml
name: "docs"
vector_store_id: null   # leave null initially; paste ID after `docask init`
includes:
  - "docs/**/*.{md,mdx,markdown,pdf}"
excludes:
  - "**/.git/**"
  - "**/node_modules/**"
  - "**/.docask_state.json"
  - "**/.DS_Store"
debounce_ms: 1200       # wait 1.2s of quiet before processing a batch
batch_max: 25           # max files per batch (prevent huge bursts)
retries:
  max_attempts: 5
  base_delay: 0.5       # seconds (exponential backoff)
  max_delay: 8          # cap per attempt
```

> You can omit the file—defaults (same as above) will be used.

---

## 3) Save this CLI as `docask` (make executable)

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true
require "optparse"
require "yaml"
require "json"
require "digest"
require "set"
require "openai"
require "listen"

STATE_FILE = ".docask_state.json"
DEFAULT_CFG = {
  "name" => "docs",
  "vector_store_id" => nil,
  "includes" => ["docs/**/*.{md,mdx,markdown,pdf}"],
  "excludes" => ["**/.git/**", "**/node_modules/**", "**/.docask_state.json", "**/.DS_Store"],
  "debounce_ms" => 1200,
  "batch_max" => 25,
  "retries" => { "max_attempts" => 5, "base_delay" => 0.5, "max_delay" => 8 }
}

class DocAsk
  def initialize
    token = ENV["OPENAI_ACCESS_TOKEN"] || ENV["OPENAI_API_KEY"]
    raise "Set OPENAI_ACCESS_TOKEN or OPENAI_API_KEY" unless token
    @client = OpenAI::Client.new(access_token: token)
    @cfg = DEFAULT_CFG.merge(load_yaml("docask.yml"))
    @retry_cfg = DEFAULT_CFG["retries"].merge(@cfg["retries"] || {})
  end

  def init(name: nil, expires_hours: 24)
    name ||= @cfg["name"]
    resp = with_retries { @client.vector_stores.create(parameters: {
      name: name,
      expires_after: { anchor: "last_active_at", days: (expires_hours/24.0).ceil }
    }) }
    id = resp["id"]
    puts id
    # persist into YAML if present
    persist_yaml("docask.yml") { |y| y["vector_store_id"] = id; y }
  end

  def sync(vector_store_id:, includes: nil, excludes: nil)
    includes ||= @cfg["includes"]
    excludes ||= @cfg["excludes"]
    paths = expand_globs(includes, excludes)
    if paths.empty?
      fallback_includes = ["**/*.{md,mdx,markdown,pdf}"]
      fallback_paths = expand_globs(fallback_includes, excludes)
      if fallback_paths.empty?
        raise "No files matched includes"
      else
        warn "No files matched includes #{includes.inspect}; falling back to #{fallback_includes.inspect}"
        paths = fallback_paths
      end
    end

    counts = Hash.new(0)
    paths.each_slice(@cfg["batch_max"]) do |batch|
      batch.each do |p|
        res = upsert_file(vector_store_id: vector_store_id, path: p)
        counts[res] += 1 if res
        $stdout.puts format("%-8s %s", (res || :skipped), p) unless res == :skipped
      end
    end
    $stdout.puts "done: #{counts[:updated]} updated, #{counts[:skipped]} skipped"
  end

  def watch(vector_store_id:, dir: ".")
    excludes = @cfg["excludes"]
    debounce_ms = Integer(@cfg["debounce_ms"])
    batch_max = Integer(@cfg["batch_max"])

    queue = Set.new
    mutex = Mutex.new
    timer = nil

    trigger = lambda do
      to_process = []
      mutex.synchronize do
        to_process = queue.to_a.first(batch_max)
        to_process.each { |p| queue.delete(p) }
      end
      process_batch(vector_store_id, to_process) unless to_process.empty?
    end

    schedule = lambda do
      # cancel old timer
      if timer&.alive?
        Thread.kill(timer) rescue nil
      end
      timer = Thread.new do
        sleep(debounce_ms / 1000.0)
        trigger.call
      end
    end

    listener = Listen.to(dir, ignore: ignores_to_regex(excludes)) do |modified, added, removed|
      changed = (modified + added).select { |p| indexable?(p) }
      deleted = removed.select { |p| indexable?(p) }
      mutex.synchronize do
        changed.each { |p| queue << [ :upsert, p ] }
        deleted.each { |p| queue << [ :remove, p ] }
      end
      schedule.call unless queue.empty?
    end

    puts "Watching #{dir} (debounce #{debounce_ms}ms, batch #{batch_max}) … Ctrl-C to stop"
    listener.start
    sleep
  end

  def ask(vector_store_id:, question:)
    asst = with_retries { @client.assistants.create(parameters: {
      model: "gpt-4.1-mini",
      instructions: "Answer strictly using the attached files; cite filenames.",
      tools: [{ type: "file_search" }],
      tool_resources: { file_search: { vector_store_ids: [vector_store_id] } }
    }) }["id"]

    thread = with_retries { @client.threads.create(parameters: {
      messages: [{ role: "user", content: question }]
    }) }["id"]

    run = with_retries { @client.runs.create(thread_id: thread, parameters: { assistant_id: asst }) }["id"]

    loop do
      st = with_retries { @client.runs.retrieve(id: run, thread_id: thread) }["status"]
      break if st == "completed"
      sleep 0.3
    end

    msgs = with_retries { @client.messages.list(thread_id: thread) }["data"]
    puts msgs.dig(0, "content", 0, "text", "value")
  end

  # ============ internals ============
  def process_batch(vector_store_id, ops)
    return if ops.empty?
    updated = removed = 0
    ops.each do |(op, path)|
      case op
      when :upsert
        res = upsert_file(vector_store_id: vector_store_id, path: path)
        updated += 1 if res == :updated
        puts format("%-8s %s", (res || :skipped), path) unless res == :skipped
      when :remove
        res = remove_file(vector_store_id: vector_store_id, path: path)
        removed += 1 if res == :removed
        puts format("%-8s %s", res, path)
      end
    end
    puts "batch: #{updated} updated, #{removed} removed"
  end

  def upsert_file(vector_store_id:, path:)
    return :skipped unless File.file?(path)
    state = load_state
    prev  = state["files"][path] || {}
    sum   = sha256(path)
    return :skipped if prev["sha256"] == sum

    file_id = with_retries { @client.files.upload(parameters: { file: path, purpose: "assistants" }) }["id"]
    vsf = with_retries { @client.vector_store_files.create(vector_store_id: vector_store_id, parameters: { file_id: file_id }) }
    state["files"][path] = { "sha256" => sum, "file_id" => file_id, "vector_store_file_id" => vsf["id"] }
    save_state(state)
    :updated
  rescue => e
    warn "upsert failed for #{path}: #{e.class}: #{e.message}"
    :error
  end

  def remove_file(vector_store_id:, path:)
    state = load_state
    rec = state["files"].delete(path)
    save_state(state)
    return :missing unless rec
    begin
      with_retries { @client.vector_store_files.delete(vector_store_id: vector_store_id, file_id: rec["file_id"]) }
    rescue => _
      # remote already gone—ignore
    end
    :removed
  end

  def with_retries(max: nil, base: nil, cap: nil)
    max ||= @retry_cfg["max_attempts"]
    base ||= @retry_cfg["base_delay"]
    cap ||= @retry_cfg["max_delay"]
    attempt = 0
    begin
      attempt += 1
      return yield
    rescue => e
      raise if attempt >= max
      sleep_time = [cap, base * (2 ** (attempt - 1))].min
      # full jitter (0..sleep_time)
      sleep(rand * sleep_time)
      retry
    end
  end

  def load_yaml(path)
    File.exist?(path) ? (YAML.load_file(path) || {}) : {}
  end

  def persist_yaml(path)
    base = load_yaml(path)
    merged = yield(base.dup)
    File.write(path, merged.to_yaml)
  rescue => _
    # best-effort; skip if no permission
  end

  def load_state
    File.exist?(STATE_FILE) ? JSON.parse(File.read(STATE_FILE)) : { "files" => {} }
  end

  def save_state(state)
    File.write(STATE_FILE, JSON.pretty_generate(state))
  end

  def sha256(path)
    Digest::SHA256.file(path).hexdigest
  end

  def indexable?(path)
    path =~ /\.(md|mdx|markdown|pdf)\z/i
  end

  def expand_globs(includes, excludes)
    inc_paths = includes.flat_map { |g| Dir[g] }.uniq
    reject = ->(p) { excludes.any? { |g| File.fnmatch?(g, p, File::FNM_PATHNAME | File::FNM_DOTMATCH | File::FNM_EXTGLOB) } }
    inc_paths.reject { |p| reject.call(p) }
  end

  def ignores_to_regex(globs)
    # Convert exclude globs to a single regex for Listen
    parts = globs.map do |g|
      Regexp.escape(g).gsub("\\*\\*", ".*").gsub("\\*", '[^/]*?')
    end
    parts.empty? ? nil : Regexp.new(parts.join("|"))
  end
end

# ---------------- CLI ----------------
cmd = ARGV.shift
cli = DocAsk.new

case cmd
when "init"
  name = ENV["DOCASK_NAME"] || nil
  hours = (ENV["DOCASK_EXPIRES_HOURS"] || "24").to_i
  cli.init(name: name, expires_hours: hours)
when "sync"
  vs = ENV["VECTOR_STORE_ID"] || cli.send(:load_yaml, "docask.yml")["vector_store_id"] || abort("Set VECTOR_STORE_ID or docask.yml vector_store_id")
  cli.sync(vector_store_id: vs)
when "watch"
  vs = ENV["VECTOR_STORE_ID"] || cli.send(:load_yaml, "docask.yml")["vector_store_id"] || abort("Set VECTOR_STORE_ID or docask.yml vector_store_id")
  dir = ARGV[0] || "."
  cli.watch(vector_store_id: vs, dir: dir)
when "ask"
  vs = ENV["VECTOR_STORE_ID"] || cli.send(:load_yaml, "docask.yml")["vector_store_id"] || abort("Set VECTOR_STORE_ID or docask.yml vector_store_id")
  q  = ARGV.join(" ")
  abort "Usage: docask ask \"your question\"" if q.strip.empty?
  cli.ask(vector_store_id: vs, question: q)
else
  puts <<~USAGE
    Usage:
      docask init                          # create vector store; prints id & writes to docask.yml
      docask sync                          # upsert all included files once
      docask watch [dir]                   # watch dir (debounced batches)
      docask ask "How do I run the app?"   # query your docs

    ENV or docask.yml may provide VECTOR_STORE_ID
  USAGE
end
```

Make it executable:

```bash
chmod +x docask
```

---

## 4) First run

```bash
./docask init                     # prints a vector store id and writes it into docask.yml
./docask sync                     # one-time bulk upsert
./docask watch docs               # live updates: add/modify/delete
./docask ask "How do I run the project locally?"
```

---

## 5) Notes

- **Costs**: You pay only when files are processed (initial + changes) and when you ask questions. The watcher uploads **only changed** files.
- **PDF vs Markdown**: Both work. For huge PDFs, consider splitting—this script will still upsert, but smaller files re-index faster.
- **CI Option**: Run `docask sync` in CI after docs changes merge to ensure prod stores stay fresh.

---

## 6) Troubleshooting

- `Set VECTOR_STORE_ID or docask.yml vector_store_id`: Run `docask init` first, or paste the id.
- Flaky network/API limits: handled by retries with jitter; if it still fails, rerun `docask sync`.
- To reset state cache: delete `.docask_state.json` and re-run `sync`.

