# Langfuse Chatbot Tour

A hands-on way to learn every major Langfuse feature by building **one chatbot**
and adding **one feature per stage**. Each stage is a runnable file. You feel why
a feature exists before moving to the next.

Everything runs against a **local, self-hosted Langfuse** and works with **no LLM
API key** — a deterministic mock LLM is built in. Set `OPENAI_API_KEY` if you want
real model calls instead.

> Verified against Langfuse Python SDK **v4** (the current generation). Older
> guides showing `langfuse.decorators` or `CallbackHandler`-only patterns are v2/v3
> and won't match this code.

---

## 1. Start Langfuse locally (one time)

Requires Docker Desktop running on your Mac.

```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
docker compose up        # wait ~2-3 min until "langfuse-web-1 ... Ready"
```

Open http://localhost:3000, sign up (local, any email works), create an
**organization -> project**, then copy the **public** and **secret** API keys from
Project Settings -> API Keys.

Give Docker at least ~4 GB RAM. The stack also runs Postgres, ClickHouse, Redis,
and MinIO, so the first boot is the slow one.

## 2. Set up this project

```bash
cd langfuse-chatbot-tour
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# edit .env: paste your LOCAL pk-lf-... and sk-lf-... keys, keep host as localhost:3000
```

That's it. Run the stages in order.

---

## The stages

| Stage | File | Feature it adds | Where to look in the UI |
|------:|------|-----------------|--------------------------|
| 0 | `stage_0_plain_bot.py` | none — baseline blindness | (nothing) |
| 1 | `stage_1_tracing.py` | **Tracing** (`@observe` + generation) | Tracing |
| 2 | `stage_2_sessions.py` | **Sessions + Users** | Sessions, Users |
| 3 | `stage_3_nested_spans.py` | **Nested spans** (retrieve -> generate) | a trace's tree view |
| 4 | `stage_4_prompt_mgmt.py` | **Prompt Management** (versioned, linked) | Prompts |
| 5 | `stage_5_user_feedback.py` | **Scores** (thumbs up/down) | scores on traces, Dashboards |
| 6 | *(UI only — see below)* | **LLM-as-a-Judge** | Evaluators |
| 7 | `stage_7_datasets_experiments.py` | **Datasets + Experiments** | Datasets -> Runs |
| 8 | *(UI only — see below)* | **Playground** | Playground |

Each stage is chat-interactive (type messages, `quit` to exit) except Stage 7,
which runs an experiment and exits. All of them call `langfuse.flush()` before
exiting so nothing is lost.

### Stage 0 — Plain bot
No instrumentation. You only see what you `print`. This is the problem the rest
of the tour solves.

### Stage 1 — Tracing
`@observe(name="chat-turn")` creates a **trace** per turn; inside it we open a
**generation** observation recording model, input messages, output, and token
usage. Open **Tracing** and click a trace to see latency and cost.

### Stage 2 — Sessions + Users
A chatbot is multi-turn, so isolated traces aren't enough. `propagate_attributes(
session_id=..., user_id=...)` tags every observation in the turn. **Sessions**
now replays a whole conversation; you can filter traces by user.

### Stage 3 — Nested spans
Adds a fake retrieval step as a child **span** before the generation, so the
trace becomes a tree: `chat-turn -> retrieve-context -> llm-response`. This is how
you find which step is slow or where a bad answer originated.

### Stage 4 — Prompt Management
The system prompt moves out of code into Langfuse. The bot fetches it with
`get_prompt(...)` (cached, so no added latency) and passes `prompt=` to the
generation so the trace **links to the prompt version**. Try it: run once, then
edit the prompt in **Prompts**, save a new version, run again — no code change.

### Stage 5 — Scores (user feedback)
After each reply you rate it up/down; we attach a BOOLEAN **score** to that trace
via `create_score(trace_id=...)`. Scores show on traces and aggregate in
**Dashboards** — your first quality signal.

### Stage 6 — LLM-as-a-Judge (configure in the UI)
No code. In Langfuse go to **Evaluators**, add a managed LLM-as-a-judge evaluator
(e.g. helpfulness or toxicity), point it at your project's traces, and it will
**auto-score production traces** as they arrive. Re-run any earlier stage to
generate traffic and watch scores appear automatically.
(Requires an LLM provider key configured in Langfuse settings for the judge model.)

### Stage 7 — Datasets + Experiments
The offline eval loop. Creates a **Dataset** `support-qa`, defines a `task()` that
runs one item through the bot, and two **evaluators** (expected-keyword match +
conciseness). `run_experiment(...)` runs the whole set and uploads scored results.
Change the prompt or `MODEL_NAME` and run again, then compare runs side-by-side in
**Datasets -> support-qa -> Runs**. This is how you *prove* a change helped.

### Stage 8 — Playground (use in the UI)
No code. When a trace looks bad, open it and jump into the **Playground** to tweak
the prompt/model and iterate without leaving Langfuse. Self-hosted instances
include the Playground (it's the Cloud free tier that omits it).

---

## Suggested demo order for the team

1. Run **Stage 0**, then **Stage 1** — show the before/after in one minute.
2. Run **Stage 2–3** — open **Sessions** and a nested trace.
3. Run **Stage 4**, edit the prompt live in the UI, re-run.
4. Run **Stage 5**, rate a couple replies, show the score on the trace.
5. Configure **Stage 6** evaluator, re-run Stage 5 to show auto-scoring.
6. Run **Stage 7** twice with different prompts, compare runs.
7. Open **Stage 8** Playground from a bad trace.

## Notes & gotchas

- **Mock vs real LLM:** leave `OPENAI_API_KEY` unset to use the deterministic mock
  (great for reproducible demos). Set it for real answers.
- **macOS LibreSSL warning:** harmless; silenced via `PYTHONWARNINGS=ignore` in
  `bot_core.py`.
- **401 Unauthorized:** keys are per-instance. Make sure `.env` uses your *local*
  keys and `LANGFUSE_HOST=http://localhost:3000`, not cloud.
- **Short scripts must flush:** the SDK buffers in the background; every stage
  calls `langfuse.flush()` so traces aren't lost on exit.
- **Shutdown Langfuse:** `Ctrl+C`, or `docker compose down` (add `-v` to wipe data).
