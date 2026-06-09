# `fm` CLI — Complete Reference & Serve API

All content here is **[OBSERVED]** — captured by running the tool on macOS 27, ANSI stripped.

```
% fm <command> [options]
```

## Commands

| Command | Purpose |
|---------|---------|
| `respond` | Generate a response to a prompt (one-shot) |
| `chat` | Interactive chat session (persisted to `~/.fm/sessions/`) |
| `token-count` | Count tokens with the on-device tokenizer |
| `schema` | Generate a JSON generation schema (for guided generation) |
| `serve` | Start an OpenAI Chat-Completions API server |
| `available` | Check model availability (`system` / `pcc`) |
| `quota-usage` | Check PCC quota usage |

## Models (execution tiers)

| Model | Meaning | Status on this host |
|-------|---------|---------------------|
| `system` | On-device Apple Foundation Model (default) | ✅ available |
| `pcc` | Apple Foundation Model on Private Cloud Compute | ❌ "not available in this context" |

---

## `fm respond`

```
fm respond [options] '<prompt>'
echo '<prompt>' | fm respond            # reads stdin
```

| Option | Notes |
|--------|-------|
| `-m, --model <system\|pcc>` | Execution tier |
| `-i, --instructions <text>` | System instructions |
| `--schema <file>` | JSON schema file → **constrained/guided generation** |
| `--text <text>` | Add a text segment to the prompt |
| `--image <path>` | Add an image (multimodal / vision) |
| `--load-transcript <file>` | Seed conversation from a saved transcript |
| `--save-transcript <name>` | Persist the transcript after responding |
| `--[no-]stream` | Stream tokens (default **on**) |
| `-g, --greedy` | Greedy (deterministic) sampling |
| `-v, --verbose` | Verbose — prints "Creating session with …" (no model IDs/latency exposed) |
| **System-model only:** | |
| `--use-case <general\|content-tagging>` | Selects a use-case adapter. `content-tagging` returns comma-separated tags. |
| `--guardrails <level>` | `default` or `permissive-content-transformations` (a third level, `permissive`, exists in an internal error string but is not advertised in help) |

**Examples that work:**
```bash
fm respond 'What is Swift?'
fm respond --greedy --no-stream 'Say hello in three words.'
fm respond --use-case content-tagging 'Apple shipped a CLI for on-device AI.'
#   → technology, ai, macOS, tools, cli
fm respond --schema movie.json 'Give me a classic sci-fi film.'
#   → {"title":"Blade Runner","year":1982,"rewatchable":true}
fm respond --image photo.jpg --text 'What is in this image?'
```

---

## `fm chat`

Interactive REPL. Sessions saved to `~/.fm/sessions/`.

| Option | Notes |
|--------|-------|
| `-m, --model <system\|pcc>` | Tier for this session |
| `--set-default-model <system\|pcc>` | Persist default tier for future `fm chat` |
| `-r, --resume <name>` | Resume a saved session |
| `--continue` | Continue the most recent session |
| `-i, --instructions <text>` | System instructions (mutually exclusive with `--resume`/`--load-transcript`) |

**In-REPL slash commands** (recovered from the binary — `/help` lists them live):
`/help`, `/model <system|pcc>`, `/instructions [text]`, `/appearance <dark|light|auto>`,
`/history` (list sessions), `/load <name>`, `/save`, `/clear`, `/increase-limit` (PCC quota flow — PCC only), `exit`.

---

## `fm token-count`

On-device tokenizer only. Output is `Token count: N` in a TTY, a bare integer when piped (`-q/--quiet` forces bare).

```bash
fm token-count 'The quick brown fox jumps over the lazy dog.'   # → 19
fm token-count -i 'You are concise' --image a.jpg --text 'Describe' 'Follow up'
```
Supports `--instructions`, repeatable `--text` / `--image`, and `--load-transcript`.

---

## `fm schema object` — guided-generation schema builder

Emits a JSON Schema (with an `x-order` extension to preserve field order) for use with `respond --schema`.

| Option | Notes |
|--------|-------|
| `--name <Name>` | Root object type name (required) |
| `--string/--integer/--double/--boolean <name>` | Declare a typed property |
| `--object <name> --schema <json>` | Nested object (compose with `$(fm schema object …)`) |
| `--anyOf --schema … --schema …` | Union type |
| `--array` / `--optional` / `--description <text>` | Modifiers on the **preceding** property |
| dot notation | `address.street` auto-creates nested objects |

```bash
fm schema object --name Movie --string title --integer year --boolean rewatchable
fm schema object --name Restaurant --string title \
  --object address --schema "$(fm schema object --name Address --string zip)"
```

---

## `fm available` / `fm quota-usage`

```bash
fm available                 # checks all tiers
#   System model available
#   Error: PCC inference is not available in this context.
fm quota-usage
#   System: Not applicable (quota only applies to PCC)
#   PCC: unavailable (PCC inference is not available in this context.)
```
`--model <system|pcc>` narrows to one tier.

---

## `fm serve` — OpenAI Chat-Completions API server

```
fm serve                              # default TCP
fm serve --host 0.0.0.0 --port 1976   # TCP, custom bind
fm serve --socket /tmp/fm.sock        # Unix-domain socket (recommended for local bindings)
```
`--socket` is mutually exclusive with `--host`/`--port`.

### Endpoints (all verified live)

| Method | Path | Behavior |
|--------|------|----------|
| `GET`  | `/health` | `{"status":"fm serve is running","models":[{name,available,reason?}]}` |
| `GET`  | `/v1/models` | OpenAI model list; `id` ∈ {`system`,`pcc`}, `owned_by:"Apple"` |
| `POST` | `/v1/chat/completions` | Chat completions, streaming **and** non-streaming |

### Verified request/response

**Non-streaming** — full OpenAI shape, real `usage` accounting:
```bash
curl -s localhost:PORT/v1/chat/completions -H 'Content-Type: application/json' -d '{
  "model":"system",
  "messages":[{"role":"system","content":"You are terse."},
              {"role":"user","content":"Name 2 colors."}],
  "temperature":0.0,"max_tokens":20,"stream":false}'
```
```json
{
  "id":"chatcmpl-…","object":"chat.completion","model":"system","created":1781042656,
  "choices":[{"index":0,"finish_reason":"stop",
              "message":{"role":"assistant","content":"Yes. Red and blue."}}],
  "usage":{"prompt_tokens":61,"completion_tokens":13,"total_tokens":74}
}
```
Accepted/honored params seen: `model`, `messages` (system/user/assistant roles), `temperature`, `max_tokens`, `stream`.

**Streaming** — standard SSE `data:` frames with OpenAI delta objects:
```
data: {"id":"chatcmpl-…","choices":[{"delta":{"role":"assistant"}}],"model":"system"}
data: {"id":"chatcmpl-…","model":"system","choices":[{"delta":{"content":"Yes,"}}]}
data: {"id":"chatcmpl-…","model":"system","choices":[{"delta":{"content":" this"}}]}
…
```

**Error envelope** — OpenAI-style, e.g. requesting the gated tier:
```json
{"error":{"type":"service_unavailable","code":"503",
          "message":"Model 'pcc' is unavailable: PCC inference is not available in this context."}}
```

### Wiring it into agent frameworks

Because it speaks the Chat Completions API, point any OpenAI-compatible client at it:
```bash
fm serve --port 1976 &
export OPENAI_BASE_URL="http://127.0.0.1:1976/v1"
export OPENAI_API_KEY="not-checked"      # no auth enforced on localhost
# model name = "system" (or "pcc" where available)
```
For local Python, prefer the Unix socket (`--socket /tmp/fm.sock`) per Apple's own help note.
**Note:** no API-key check is performed — bind to `127.0.0.1`, not `0.0.0.0`, unless you intend to expose it.
