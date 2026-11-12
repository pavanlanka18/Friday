# PLAN.md — Friday (personal AI assistant)

> **For the coding agent:** this file is your source of truth. Read it fully before writing any code.
> Work on **one step at a time**, in order. After each step, run its check, tick the box, add a line
> to the Progress Log at the bottom, and **stop for human review** where a step says `STOP`.

---

## 0. What we are building

Friday is a backend-only personal AI assistant (inspired by F.R.I.D.A.Y.) that:

- thinks with an **open-weight model** from one of two swappable providers:
  - **Ollama** — runs locally on the laptop (private, offline, needs a decent GPU), or
  - **NVIDIA API** (build.nvidia.com) — hosted in the cloud with a free `nvapi-` key, for anyone whose
    laptop is too weak to run a model,
- remembers conversations and facts in **Supabase (Postgres + pgvector)**,
- stores files, audio and backups in **Google Cloud Storage (GCS)**,
- can use **tools** (time, notes, memory, reminders, later web search),
- exposes a **REST API** so an Android app can talk to it and change settings later,
- eventually wakes up when Pavan **says "Friday" or claps twice**, and replies by voice.

No web frontend. Python backend only.

### Hardware it must run on
- Laptop: 24 GB RAM, 6 GB VRAM GPU, 512 GB SSD.
- 6 GB VRAM fits **one ~7–8B model at 4-bit quantisation**. Keep everything else (speech-to-text,
  embeddings if large) on **CPU** so the chat model keeps the GPU.
- **Low-spec machines** (no GPU, < 16 GB RAM): set `FRIDAY_PROVIDER=nvidia`. Ollama is then not needed
  at all; the laptop only runs the lightweight agent, database client and API.

### Provider facts to respect (NVIDIA API, checked Nov 2026 — re-check before relying on them)
- OpenAI-compatible: base URL `https://integrate.api.nvidia.com/v1`, `POST /chat/completions`
  (streaming + tool calling on supported models), `POST /embeddings`, `GET /models`.
- Free tier is **rate-limited (~40 requests/min, varies by model and load)**, not unlimited.
  The agent loop makes several calls per message when tools are used, so handle `429` with backoff.
- The free hosted endpoint is meant for **development/testing**, not production. Fine for a personal
  project; a public product would need a paid plan or self-hosting.
- Prompts go to NVIDIA's servers. Anything private sent to Friday leaves the machine in this mode.
- Model ids change. Never hard-code them — read from config and verify with `GET /models`.

---

## 1. Rules for the agent (read every session)

1. **Do not run `ollama pull`.** Pavan pulls models manually. If a model is missing, stop and tell him
   the exact command to run, e.g. `ollama pull <model>`. Never print or log the `NVIDIA_API_KEY`.
2. **One step per session/commit.** Don't start the next step until the current one passes its check.
3. **Small diffs.** Prefer adding new files over rewriting working ones. Never delete working code
   without saying why.
4. **No secrets in code or git.** Everything secret goes in `.env` (git-ignored). Keep `.env.example`
   updated with every new variable (with dummy values).
5. **Config over hard-coding.** Model names, URLs, keys, paths → `friday/config.py` reading from `.env`.
6. **Every step ships with a test or a runnable check.** Use `pytest`. Tests must not need the GPU,
   Supabase or GCS — mock them. Mark real-service tests with `@pytest.mark.integration`.
7. **Ask before adding heavy dependencies** (anything > ~50 MB or needing system packages).
8. **If something in this plan is wrong or unclear, stop and ask.** Don't silently redesign.
9. If an older Friday scaffold already exists in the repo (brain failover, SQLite memory, CLI),
   **reuse and adapt it** instead of starting over; note what you kept in the Progress Log.
10. Update the **Progress Log** at the end of every step.

---

## 2. Tech stack

| Concern | Choice | Notes |
|---|---|---|
| Language | Python 3.11+ | venv in `.venv/` |
| LLM provider | `FRIDAY_PROVIDER` = `ollama` \| `nvidia` \| `auto` | `auto` = try the primary, fall back to the other on error/timeout/429 |
| Local runtime | Ollama (`http://localhost:11434`) | Native `/api/chat` (streaming + tool calls) |
| Cloud runtime | NVIDIA API (`https://integrate.api.nvidia.com/v1`) | OpenAI-compatible; use the `openai` Python SDK with `base_url` |
| Chat model | `OLLAMA_MODEL` / `NVIDIA_MODEL` | Must support **tool calling**. Local: fits 6 GB at Q4 (7–8B). Cloud: any tool-capable model from `GET /models`. |
| Embedding model | `EMBED_PROVIDER` + `EMBED_MODEL` | **Pinned separately from chat.** Switching it changes the vector size and means re-embedding all memory. Dimension read at startup. |
| HTTP client | `httpx` | async |
| Database | Supabase Postgres + `pgvector` | via `psycopg` (v3, async). Plain SQL migrations in `migrations/`. |
| File storage | Google Cloud Storage | `google-cloud-storage`, service-account JSON path in `.env` |
| API server | FastAPI + Uvicorn | Bearer-token auth |
| Scheduler | APScheduler | reminders, morning brief |
| Speech-to-text | `faster-whisper` on **CPU (int8)** | keeps VRAM free for the LLM |
| Text-to-speech | Piper | offline, fast on CPU |
| Wake word | openWakeWord (custom "Friday" model) or Porcupine | decided at Step 9.1 |
| Clap detection | `sounddevice` + `numpy` | own simple onset detector |
| Tests | `pytest`, `pytest-asyncio`, `respx` (mock httpx) | |
| Lint/format | `ruff` | |

---

## 3. Target folder layout

```
friday/
  __init__.py
  config.py            # settings from .env
  logging.py           # one logger setup
  brain/
    base.py            # LLMProvider interface + common types (Message, ToolCall, ChatResult)
    ollama_provider.py # local: chat, stream, embeddings, health
    nvidia_provider.py # cloud: same interface via OpenAI-compatible API
    router.py          # picks provider from settings, fallback + cooldown on failure
    prompts.py         # Friday persona / system prompt
  agent/
    loop.py            # message -> LLM -> tools -> LLM -> reply
    context.py         # builds the prompt: persona + memories + recent history
  memory/
    db.py              # Postgres connection pool
    repo.py            # conversations, messages, facts, notes, reminders
    semantic.py        # embed + vector search
  tools/
    registry.py        # @tool decorator, JSON schemas for Ollama
    builtin_time.py
    builtin_memory.py
    builtin_notes.py
    builtin_reminders.py
  storage/
    gcs.py             # upload/download/list, backups
  api/
    server.py          # FastAPI app
    auth.py
  voice/
    stt.py  tts.py  wake_word.py  clap.py  listener.py
  scheduler/
    jobs.py
  cli.py               # `python -m friday.cli`
migrations/
  001_init.sql  002_pgvector.sql  ...
scripts/
  check_brain.py  migrate.py  reembed.py
tests/
.env.example
requirements.txt
README.md
PLAN.md
```

---

## 4. Environment variables (`.env.example`)

```
# Brain — which provider answers chat
FRIDAY_PROVIDER=ollama            # ollama | nvidia | auto
FRIDAY_FALLBACK_PROVIDER=nvidia   # used only when FRIDAY_PROVIDER=auto
FRIDAY_NUM_CTX=8192
FRIDAY_TEMPERATURE=0.6

# Local (Ollama)
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=your-chat-model:tag

# Cloud (NVIDIA API — get a key at build.nvidia.com, starts with nvapi-)
NVIDIA_API_KEY=nvapi-xxxxxxxx
NVIDIA_BASE_URL=https://integrate.api.nvidia.com/v1
NVIDIA_MODEL=publisher/model-id   # copy an id from GET /models that supports tool calling

# Embeddings (memory) — pick ONE and keep it; changing it means re-embedding everything
EMBED_PROVIDER=ollama             # ollama | nvidia
EMBED_MODEL=nomic-embed-text

# Database (Supabase)
DATABASE_URL=postgresql://postgres:password@db.xxxxx.supabase.co:5432/postgres

# Google Cloud Storage
GCS_BUCKET=friday-memory
GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/to/service-account.json

# API
FRIDAY_API_TOKEN=change-me-long-random-string
FRIDAY_SECRET_KEY=generate-with-cryptography-fernet   # encrypts keys saved from the phone app
FRIDAY_API_HOST=127.0.0.1
FRIDAY_API_PORT=8765

# Owner
FRIDAY_OWNER_NAME=Pavan
FRIDAY_TIMEZONE=Asia/Kolkata
```

---

## 5. Phases and steps

Legend: `[ ]` todo · `[x]` done · **Check** = how to prove the step works · `STOP` = wait for Pavan

### Phase 0 — Project setup

- [ ] **0.1 Repo skeleton.** Create the folders above (empty `__init__.py` files), `.gitignore`
  (`.venv`, `.env`, `__pycache__`, `*.json` credentials, `audio_cache/`), `requirements.txt`,
  `README.md` with setup steps.
  **Check:** `python -c "import friday"` works.
- [ ] **0.2 Config.** `friday/config.py` loads `.env` (use `pydantic-settings`), fails with a clear
  message if a required variable is missing. Add `.env.example`.
  **Check:** test that `FRIDAY_PROVIDER=nvidia` without `NVIDIA_API_KEY` (or `ollama` without `OLLAMA_MODEL`) raises a readable error.
- [ ] **0.3 Logging.** One `get_logger(name)` helper; logs to console and `logs/friday.log` (rotating).
- [ ] **0.4 Provider health check.** `scripts/check_brain.py` checks only the providers that are configured:
  - **Ollama:** `GET /api/tags`, list installed models, say whether `OLLAMA_MODEL` (and `EMBED_MODEL`
    if `EMBED_PROVIDER=ollama`) are present. If missing, print `Run: ollama pull <name>` — **do not pull**.
  - **NVIDIA:** confirm `NVIDIA_API_KEY` is set (show only `nvapi-…last4`), call `GET /models`, say
    whether `NVIDIA_MODEL` (and the embed model if `EMBED_PROVIDER=nvidia`) is in the list, then send one
    tiny chat request ("Reply with OK") and print latency.
  **Check:** run it. `STOP` — Pavan confirms the provider(s) he wants are working.

### Phase 1 — Brain (talk to the local model)

- [ ] **1.1 Provider interface.** `brain/base.py`: abstract `LLMProvider` with async
  `chat(messages, tools=None) -> ChatResult`, `chat_stream(messages) -> AsyncIterator[str]`,
  `embed(texts) -> list[list[float]]`, `health() -> bool`, and a `name`. Define shared types
  (`Message`, `ToolCall`, `ChatResult`) so the rest of Friday never sees provider-specific JSON.
- [ ] **1.2 Ollama provider.** `brain/ollama_provider.py` implementing the interface over `/api/chat`
  and `/api/embed`. Pass `num_ctx`, `temperature`, `keep_alive`. Timeouts + one retry on connection errors.
  **Check:** unit tests with `respx` (normal, streaming, tool call, error).
- [ ] **1.3 NVIDIA provider.** `brain/nvidia_provider.py` using the `openai` SDK with
  `base_url=NVIDIA_BASE_URL`, `api_key=NVIDIA_API_KEY`. Convert OpenAI-style `tool_calls`
  (arguments arrive as a JSON **string**) into the shared `ToolCall` type. On `429`, retry with
  exponential backoff (max 3) and then raise `RateLimited`. For embeddings, pass
  `input_type="passage"` when storing and `"query"` when searching if the model requires it.
  **Check:** unit tests with mocked responses (normal, streaming, tool call, 429, bad key).
  One `@pytest.mark.integration` test that hits the real API.
- [ ] **1.4 Router.** `brain/router.py`: `get_brain()` returns the provider from `FRIDAY_PROVIDER`.
  In `auto` mode: try primary → on timeout / connection error / `RateLimited`, use the fallback and
  put the primary on a cooldown (e.g. 60 s). Log which provider answered each message.
  Embeddings always use `EMBED_PROVIDER` — **never** fall back to a different embed model.
  **Check:** test that a failing primary falls back, and that cooldown expires.
- [ ] **1.5 Persona.** `brain/prompts.py`: Friday's system prompt — calm, concise, slightly witty,
  addresses the owner by name, admits when it doesn't know, never invents tool results.
  Include current date/time and timezone in the prompt at runtime.
- [ ] **1.6 CLI chat (no memory yet).** `python -m friday.cli` — REPL that streams replies, keeps
  history in RAM for the session, `/exit`, `/reset`, `/provider ollama|nvidia|auto` to switch live,
  `/status` shows the active provider and model. `--provider` flag on start.
  **Check:** 5-turn conversation on each provider; Friday remembers turn 1 at turn 5. `STOP` — Pavan tries it.

### Phase 2 — Database (Supabase Postgres)

- [ ] **2.1 Connection.** `memory/db.py`: async connection pool from `DATABASE_URL`, `ping()`.
  **Check:** integration test `ping()` returns OK against Supabase.
- [ ] **2.2 Migrations runner.** `scripts/migrate.py` applies `migrations/*.sql` in order and records
  them in a `schema_migrations` table (idempotent — running twice does nothing).
- [ ] **2.3 Core schema** (`001_init.sql`):
  - `conversations(id uuid pk, channel text, started_at timestamptz, title text)`
  - `messages(id uuid pk, conversation_id fk, role text, content text, tool_name text null,
     created_at timestamptz)`
  - `facts(id uuid pk, content text, source text, created_at, updated_at, is_active bool)`
  - `notes(id uuid pk, title text, body text, created_at, updated_at)`
  - `reminders(id uuid pk, text text, due_at timestamptz, status text, created_at)`
  - `settings(key text pk, value jsonb, updated_at)`
  **Check:** migrate runs clean on Supabase; tables visible in Supabase dashboard. `STOP`.
- [ ] **2.4 Repository layer.** `memory/repo.py` with small functions (create conversation, add
  message, last N messages, add/list/deactivate fact, CRUD notes, reminders, get/set setting).
  **Check:** integration tests for each function (use a test schema or clean up after).
- [ ] **2.5 Persist chat.** CLI now saves every message; `/history` shows last 10;
  starting the CLI resumes the latest conversation (`--new` starts fresh).
  **Check:** restart CLI, Friday still knows what was said before.

### Phase 3 — Long-term memory (semantic)

- [ ] **3.1 pgvector.** `002_pgvector.sql`: `create extension if not exists vector;` add
  `embedding vector(N)` to `facts` and `messages` — N = embedding model dimension
  (`scripts/migrate.py` asks `EMBED_PROVIDER` once and templates it). Add an HNSW or IVFFlat index.
  Store the embed provider + model name in `settings`; on startup, if `.env` no longer matches,
  refuse to write new embeddings and print how to run `scripts/reembed.py` (write that script too).
- [ ] **3.2 Embed + search.** `memory/semantic.py`: `remember(text)` stores fact + embedding;
  `recall(query, k=5)` returns top-k active facts by cosine distance; skip near-duplicates
  (similarity > 0.92 → update instead of insert).
  **Check:** unit tests with fake embeddings; integration test with real embed model.
- [ ] **3.3 Context builder.** `agent/context.py` builds each prompt as:
  persona → top-k recalled facts ("Things you know about {owner}") → last N messages → new message.
  Trim to fit `FRIDAY_NUM_CTX` (rough token estimate is fine).
  **Check:** test that context never exceeds the budget. `STOP` — Pavan tells Friday a fact,
  restarts, asks about it.

### Phase 4 — Tools (Friday can do things)

- [ ] **4.1 Tool registry.** `tools/registry.py`: `@tool` decorator that turns a typed Python function
  + docstring into an Ollama tool JSON schema; `run(name, args)` with argument validation and
  error capture (errors go back to the model as text, never crash the loop).
- [ ] **4.2 Agent loop.** `agent/loop.py`: send messages + tool schemas → if the model returns
  `tool_calls`, execute them, append results as `role: "tool"`, call the model again.
  Max 5 tool rounds per user message. Log every tool call.
  **Check:** test with a mocked model that requests a tool, then answers.
- [ ] **4.3 Built-in tools:**
  - `current_time()` — in `FRIDAY_TIMEZONE`
  - `remember_fact(text)` / `recall_facts(query)` / `forget_fact(id)`
  - `create_note(title, body)` / `search_notes(query)` / `list_notes()`
  **Check:** CLI: "remember that my favourite colour is blue" → stored; "what time is it?" uses tool.
  `STOP` — Pavan tests whether the chosen model calls tools reliably **on each provider he uses**.
  If not, try another model (he pulls it, or changes `NVIDIA_MODEL`) before going further.
  Add a rate-limit guard: if one user message would need more than ~8 cloud calls, stop and answer
  with what Friday has so far.

### Phase 5 — API server (for the Android app later)

- [ ] **5.1 FastAPI app.** `api/server.py` with Bearer-token auth (`FRIDAY_API_TOKEN`):
  - `GET /health` (no auth) — Ollama + DB status
  - `POST /chat` `{message, conversation_id?}` → `{reply, conversation_id}`
  - `POST /chat/stream` → Server-Sent Events
  - `GET /conversations`, `GET /conversations/{id}/messages`
  - `GET/PUT /settings` (model name, temperature, persona tweaks, wake options)
  - `GET/POST/DELETE /facts`, `/notes`, `/reminders`
- [ ] **5.2 Settings live-reload.** Settings in the `settings` table override `.env` defaults at
  runtime (so the phone app can switch provider `ollama/nvidia/auto`, model and temperature without
  restart). `/health` reports each provider's status.
- [ ] **5.2b API keys from the phone.** `PUT /settings/secrets/nvidia_api_key` stores the key
  **encrypted** (Fernet, master key in `.env` as `FRIDAY_SECRET_KEY`), never returns it in plain text
  (only `nvapi-…last4`). `.env` value wins if both exist.
- [ ] **5.3 Run script.** `python -m friday.api` binds to `FRIDAY_API_HOST:PORT` (localhost by default).
  **Check:** API tests with FastAPI `TestClient`; `curl` example in README. `STOP`.

### Phase 6 — Google Cloud Storage

- [ ] **6.1 GCS wrapper.** `storage/gcs.py`: `upload(local_path, dest)`, `download`, `list(prefix)`,
  `signed_url(path, minutes)`. Bucket layout: `audio/`, `files/`, `backups/`, `exports/`.
- [ ] **6.2 Memory export/backup.** Daily job (and `/backup` CLI command) exports facts, notes,
  conversations to JSONL, gzips it, uploads to `backups/YYYY-MM-DD.jsonl.gz`. Keep last 30.
- [ ] **6.3 File tool.** `save_file(name, content)` / `list_files()` tools so Friday can store and
  find documents in GCS (metadata row in Postgres, blob in GCS).
  **Check:** integration test uploads + downloads a small file. `STOP`.

### Phase 7 — Reminders and daily routine

- [ ] **7.1 Scheduler.** APScheduler started with the API server; jobs persisted via the `reminders`
  table (reload pending ones on startup).
- [ ] **7.2 Reminder tools.** `set_reminder(text, when)` — parse natural times ("in 20 minutes",
  "tomorrow 9am") with `dateparser` in the owner's timezone; `list_reminders()`, `cancel_reminder(id)`.
  When due: print in CLI, speak if voice is on, and queue for the phone (`GET /notifications`).
- [ ] **7.3 Morning brief.** Scheduled job at a configurable time: today's reminders + notes
  touched yesterday + (later) calendar/email. Stored as a message so the phone can show it.
  **Check:** set a reminder for 1 minute ahead, it fires. `STOP`.

### Phase 8 — Voice (talk and listen, push-to-talk first)

- [ ] **8.1 TTS.** `voice/tts.py` with Piper; `speak(text)` plays audio; strip markdown before speaking.
  Ask Pavan to download a Piper voice file (don't auto-download large files).
- [ ] **8.2 STT.** `voice/stt.py` with faster-whisper on CPU int8 (`small` or `base` model);
  `transcribe(audio) -> text`. Use VAD to cut silence.
- [ ] **8.3 Push-to-talk.** CLI `--voice` mode: press Enter → record until silence → transcribe →
  agent → speak reply.
  **Check:** full spoken round-trip works; measure latency and log it. `STOP`.

### Phase 9 — Wake on "Friday" or double clap

- [ ] **9.1 Choose wake-word engine.** Compare openWakeWord (free, custom model must be trained
  for "Friday") vs Porcupine (custom keyword via Picovoice console, needs an access key).
  Write a short note in the Progress Log. `STOP` — Pavan decides.
- [ ] **9.2 Wake word.** `voice/wake_word.py`: always-on mic stream, low CPU, sensitivity in settings.
- [ ] **9.3 Clap detector.** `voice/clap.py`: detect two sharp loud onsets 150–600 ms apart within
  a short window; ignore sustained noise. Threshold configurable; add a calibration command
  (`python -m friday.voice.clap --calibrate`).
- [ ] **9.4 Listener service.** `voice/listener.py`: wait for wake word OR double clap → short chime →
  record command → STT → agent → TTS → back to waiting. Cooldown to avoid double triggers.
  `/mute` and a settings flag to disable listening.
  **Check:** 10 wake attempts, count false positives/negatives, log them. `STOP`.

### Phase 10 — Phone access (Android)

- [ ] **10.1 Reach the laptop from the phone.** Recommended: Tailscale on laptop + phone, API bound
  to the Tailscale IP only. Document in README. Never expose the API to the open internet
  without HTTPS + token.
- [ ] **10.2 API contract doc.** `docs/API.md` with every endpoint, request/response examples, auth,
  SSE format — this is what the Android app will be built against.
- [ ] **10.3 Android app** — separate plan/repo (settings screen: API URL, token, model, wake options;
  chat screen; notifications). `STOP` — plan it together before building.

### Phase 11 — Later (do not start without Pavan)

- Run as a background service on boot (systemd / Windows Task Scheduler).
- Web search tool, Gmail/Calendar for the morning brief.
- Computer control (open apps, files).
- 24/7 cloud hosting of the agent (brain stays local or moves to a GPU host).
- Small on-phone model for offline basics.

---

## 6. Definition of done (every step)

- Code is formatted (`ruff format`) and lint-clean (`ruff check`).
- `pytest -m "not integration"` passes.
- New env vars are in `.env.example`; README updated if setup changed.
- Box ticked above and a Progress Log line added.

---

## 7. Progress Log

| Date | Step | What was done | Notes / problems |
|---|---|---|---|
| | | | |
