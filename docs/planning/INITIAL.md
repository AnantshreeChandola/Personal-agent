# INITIAL.md — Personal Agent (Planner + n8n, Python)

> Purpose: Generate a PRP that kicks off an MVP **personal agent** which plans like the user, previews changes, and executes via **n8n** after approval. Keep the scope to a working vertical slice with clean contracts; production hardening can come later.

---

## FEATURE

Build the **Phase‑1 MVP** of a personalized agent with:

1. **Planner Service (Python)** — model‑agnostic and read‑only at plan time

   * `FastAPI` app with `/health` and `/plan`
   * **Adapters** for planner slots: `openai`, `anthropic`, `gemini`, `oss` (interface: `plan(intent, context, tools, schema) -> SignedPlan + artifacts`)
   * **Plugin/Tool registry (planner‑time only)** — each capability is a *plugin* the planner can select (reason‑first):
     * `calendar_create_event.py` — propose event params (no write)
     * `gmail_create_draft.py` — compose email draft (never send)
     * `shopping_create_cart.py` — build cart preview (vendor allow‑list)
     * `web_search.py` — read‑only search/lookup helper
     * `gui_probe.py` — read‑only site probe (facts + optional screenshot refs)
     * `enqueue_n8n_workflow.py` — handoff to n8n (preview/execute webhooks)

   * **Plugin interface (Moveworks‑style)** — every plugin exposes:
     * `params_schema` (JSON Schema for inputs)
     * `preview(params) -> artifacts` (**no side effects**)
     * `executor_binding` (e.g., n8n webhook id / workflow name for execute phase)
     * optional `policy` (allow‑lists, budgets) and `telemetry` (tool‑level metrics keys)
   * **Signer**: attach `signature{alg,hash,sig}` to plans (HMAC/ed25519 placeholder)

2. **Personalization via RAG (as a plugin with threshold + citations)**
   * `preferences.example.json` — budgets, meeting windows, brand/vendor allow‑list, tone
   * `ingest.py` / `retrieve.py` — simple vector store (Chroma/Qdrant) + time‑decay; returns `{preferences, snippets[], exemplars[], tokenBudgetHint}`
   * **Knowledge/Search plugin behavior**:

     * Compute a relevance score; enforce `relevance_threshold` (default **0.75**)
     * If below threshold → return `none_found` and the planner emits a **needs\_info** or asks a clarifying question (do **not** hallucinate)
     * When above threshold → include **citations** (doc ids/links) in **Preview** artifacts

3. **Contracts & Schemas**

* `schemas/SignedPlan.schema.json` with required fields:
  * `plan_id`, `agent_run_id`, `intent`, `steps[]` (`tool`, `action`, `params`, optional `on_error`),
  * `risk{level: low|medium|high, reason}`, `limits{spend_usd?, max_emails?}`,
  * `dry_run` (default true), `signature{alg,hash,sig}`
* `schemas/Preview.schema.json` (optional) for planner‑produced preview artifacts

4. **Executor via n8n** (importable workflows)

   * `workflows/n8n/preview.json` — Webhook → read‑only checks → assemble preview → send to Approval (Slack/email) → return `preview_ref`
   * `workflows/n8n/execute.json` — On approval token → perform side‑effects (Calendar, Gmail send, Cart commit) → audit events + receipts
   * **Secrets** configured only in n8n credentials; planner never sees plaintext creds/OTP
   * (Optional) **Queue/outbound runner** note: executor may run in queue mode or behind an outbound‑only tunnel when touching private systems
5. **GUI Automation (spec‑first)**

* `gui-runner/specs/<site>.probe.json` and `commit.json` — contracts for future browser automation
* Planner uses `gui_probe` only; commits happen in n8n after approval

6. **Observability & Policy**

   * `observability/ids.md` — `agent_run_id`, `plan_id`, `n8n_execution_id`, `preview_ref`
   * `observability/metrics.md` — define counters and timers:

     * Preview latency (p50/p95), approval‑without‑edits, top plugins used
     * Failure taxonomy (auth, rate‑limit, data‑quality, GUI flake)
     * Spend within budget %, allow‑list violations (target 0)
   * Basic **risk policy**: money/external messaging/others’ calendars ⇒ `high`; drafts/holds ⇒ `medium`; reads ⇒ `low`
   * **Approval rules**: high ⇒ manual approval; medium ⇒ preview (may auto‑approve per flag); low ⇒ proceed to preview
7. **Tests & Dev Loop**

* `tests/test_signed_plan_schema.py` — validates schema; catches missing fields
* `tests/test_plan_endpoint.py` — `/plan` returns schema‑valid plan with `dry_run=true`
* `pyproject.toml` — ruff/black/pytest config; `Makefile`/`justfile` for `dev|fmt|lint|test`

**Non‑Goals (this phase):** prod auth/SSO, full GUI runner implementation, advanced retrieval/rerank, multitenancy.

---

## EXAMPLES

Provide **concrete files and snippets** so the PRP mirrors patterns. If these files don’t exist yet, the PRP should create them.

* `examples/fastapi_app.py` — minimal `/plan` stub (no side effects)

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PlanRequest(BaseModel):
    intent: str
    context: dict = {}

@app.post("/plan")
def plan(req: PlanRequest):
    return {"plan_id": "plan_demo", "agent_run_id": "run_demo", "intent": req.intent,
            "steps": [], "risk": {"level": "low", "reason": "stub"},
            "limits": {}, "dry_run": True,
            "signature": {"alg": "hmac-sha256", "hash": "…", "sig": "…"}}
```

* `examples/plugin_stub.py` — **plugin interface** pattern

```python
# Every plugin: params_schema, preview(), executor_binding
params_schema = {"type":"object","properties":{"title":{"type":"string"}},"required":["title"]}
executor_binding = {"workflow":"calendar_execute", "webhook":"/exec/calendar"}

def preview(params: dict) -> dict:
    # No side effects — return artifacts only
    return {"calendar_diff": ["Tue 10:30–11:15", "Thu 10:30–11:15"]}
```

* `examples/knowledge_search_example.py` — **threshold + citations**

```python
def knowledge_search(query: str, threshold: float = 0.75):
    results = search_index(query)  # returns [(doc_id, score, snippet), ...]
    top = max(results, key=lambda r: r[1], default=None)
    if not top or top[1] < threshold:
        return {"none_found": True, "threshold": threshold}
    return {"answer": summarize(results[:3]),
            "citations": [{"doc": d, "score": s} for d,s,_ in results[:3]]}
```

* `examples/signed_plan_minimal.json`

```json
{
  "plan_id": "plan_123",
  "agent_run_id": "run_abc",
  "intent": "Schedule coffee with Priya next week",
  "steps": [
    {"tool":"calendar.create_event","action":"propose","params":{"title":"Coffee with Priya","duration_min":45,"window":"next_week","attendees":["priya@example.com"]}},
    {"tool":"gmail.create_draft","action":"compose","params":{"to":["priya@example.com"],"subject":"Coffee next week?","body_template":"Hi Priya — …"}},
    {"tool":"enqueue.n8n_workflow","action":"preview","params":{"preview_artifacts":["calendar_diff","email_draft"],"dry_run":true}}
  ],
  "risk": {"level":"medium","reason":"calendar hold + external draft"},
  "limits": {"spend_usd": 0},
  "dry_run": true,
  "signature": {"alg":"hmac-sha256","hash":"…","sig":"…"}
}
```

* `examples/approval_preview_slack.md`

```
*Plan:* Coffee with Priya next week
• Calendar diff: proposes Tue 10:00–10:45 or Thu 15:00–15:45 (no conflicts)
• Draft email → priya@example.com (not sent)
Approve to execute? (This will place the event and send the email.)
```

* `examples/gui_probe_spec.json`

```json
{"url":"https://example.com/slots","selectors":[{"name":"slot","query":".slot"}],"constraints":{"no_login":true,"no_captcha":true}}
```

* `examples/preferences.example.json`

```json
{"meeting_windows":["Tue 10:00-12:00","Thu 15:00-17:00"],"brands":{"coffee":["Blue Bottle","Verve"]},"budgets":{"monthly_usd":300,"per_order_usd":60},"allow_list":{"vendors":["amazon","instacart","doordash"]},"tone":"brief-friendly"}
```

---

## DOCUMENTATION

* **LangGraph (Python)** — planner graph/tool patterns
  [https://langchain-ai.github.io/langgraph/](https://langchain-ai.github.io/langgraph/)
* **Anthropic tool use (Messages API)** — Claude planner slot
  [https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
* **OpenAI Responses API / tool calling** — OpenAI slot
  [https://platform.openai.com/docs/guides/realtime-and-responses/responses-api](https://platform.openai.com/docs/guides/realtime-and-responses/responses-api)
* **n8n Webhook node** (trigger/response)
  [https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
* **Playwright Python** (future GUI probe)
  [https://playwright.dev/python/docs/intro](https://playwright.dev/python/docs/intro)
* **Chroma or Qdrant** (vector store)
  [https://docs.trychroma.com/](https://docs.trychroma.com/)
  [https://qdrant.tech/documentation/](https://qdrant.tech/documentation/)

The PRP should reference which docs inform each output (schemas, planner endpoints, n8n flows).

---

## OTHER CONSIDERATIONS

* **Validation gates**: PRP must produce tests; iterate until tests pass (`pytest -q`).
* **No vibe coding**: validate JSON schemas and run tests before “execute”.
* **Safety defaults**: `dry_run=true`; high‑risk actions require approval; enforce vendor allow‑lists and spend caps.
* **Privacy**: no secrets in code; `.env.example` placeholders only; real creds live in n8n.
* **Plugin toggles**: allow enable/disable per plugin (e.g., disable `shopping` initially) and expose thresholds (e.g., `knowledge_search.relevance_threshold`).
* **Style**: Python 3.11+, type hints, pydantic v2, ruff/black, small functions with docstrings.
* **Dev commands** (include in README):

  ```bash
  uv run uvicorn planner.server:app --reload --port 8787
  ruff check . && black --check . && pytest -q
  ```
* **Branch etiquette**: feature branches; propose commit messages with diffs.
