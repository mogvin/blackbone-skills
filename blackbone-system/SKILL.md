---
name: blackbone-system
description: Consult this whenever you (Hermes) need to start, stop, check the status of, or retrieve data from any Blackbone application (BB Terminal, OpenStock, OpenAlice, AI Hedge Fund, Quant Mind, Trading Agent, Vibe Trading, Backtesting.py, Agent Reach, OpenErii, Osiris, TradingView MCP, Blackbone Retriever), or need the exact Orchestrator/Gateway API calls, message formats, and ID rules required to talk to the Blackbone system. Also consult this whenever the user says something like "let's trade" / "lets trade" / "time to trade" -- that phrase triggers the full twice-daily trade research workflow defined in this skill (section "The 'Let's Trade' Workflow"). Use this any time a request involves trading data, market data, app control, system status, or "get me data from X" within Blackbone, even if the request doesn't name a specific application.
version: 3.0.0
author: Blackbone Orchestrator Team
---

# Overview

Blackbone is a personal, single-machine trading/research system made of many
independent applications (terminals, agents, backtesters, data retrievers)
that are too heavy to all run at once on the host machine. A central
**Orchestrator** starts each on-demand application only when its data is
actually needed, waits for it to become ready, pulls a result out of it,
forwards that result to the **Gateway**, and then shuts the application
back down. **Gateway itself is different: it is always-on.** The
Orchestrator starts it immediately on its own boot (before serving any
job) and a watchdog keeps it running for as long as the Orchestrator runs.

This version corrects several factual errors found in the previous draft
of this document by checking it directly against the real source
(`orchestrator.py`, `applications.json`, `gateway_sender.py`,
`session_executor.py`). Where something is still genuinely unverified
(rather than just fixed), it says so explicitly instead of guessing.

# System Map

```
                    Hermes
                       |
                       v
              Blackbone Gateway  (Node/Express, HTTPS-only, port 64799)
                       |
                       v
          BLACKBONE ORCHESTRATOR (Python, HTTP, port 51800)
                       |
          +------------+------------+------------+
          v            v            v            v
         P1           P2           P3           P4
   (lifecycle:    (retrieval:   (delivery:   (session apps:
   start/stop      pulls data    pushes        OpenErii,
   via PSI)        out of the    result to     OpenAlice,
                    app)          Gateway       CIEN)
                                  over HTTPS)
          |
          v
   applications.json  ---defines---> which apps exist, their
          |                          startup/stop commands,
          v                          dependencies, and which
         PSI                        P2 module handles their data
   (process supervisor)
          |
          v
   The actual Blackbone application
```

**Two separate network endpoints matter here:**

| Component | Protocol | Host:Port | Purpose |
|---|---|---|---|
| Orchestrator | plain HTTP | `127.0.0.1:51800` | Submit start/stop jobs, poll job status, check system health |
| Gateway | **HTTPS** (self-signed local cert) | `127.0.0.1:64799` | Receives delivered results from the Orchestrator (P3) and is the intended point of contact for Hermes |

Do not confuse the two. The Orchestrator's own API is plain HTTP; the
Gateway is HTTPS-only. **This used to be broken on the Orchestrator side**
(P3 had no HTTPS support at all and defaulted to the wrong port) — that has
been fixed; P3 now delivers to `https://127.0.0.1:64799` with certificate
verification relaxed for this local self-signed cert by default.

# Requirements & Setup

- **Orchestrator must be running** before any job can be submitted:
  ```
  cd C:\Users\MOGVIN\Blackbone\BLACKBONE_ORCHESTRATOR
  python .\orchestrator.py
  ```
  Confirm it's up with:
  ```
  GET http://127.0.0.1:51800/status
  ```
  A healthy response looks like:
  ```json
  {
    "running": true,
    "draining": false,
    "started_at": "...",
    "components": ["P1", "P3", "P4"],
    "gateway_ready": true,
    "jobs": 0
  }
  ```
  `gateway_ready` is the field to watch — it's true once the Orchestrator
  has successfully started Gateway itself. Don't submit jobs expecting
  delivery to work while this is false.

- **Gateway no longer needs to be started separately.** The Orchestrator
  starts it automatically the moment it comes up, before serving any job,
  and a watchdog restarts it if it ever goes down. You can still start it
  manually if needed:
  ```
  cd C:\Users\MOGVIN\Blackbone\Gateway
  npm start
  ```

- **TLS note:** Gateway's certificate is local/self-signed (configured via
  `HTTPS_CERT` / `HTTPS_KEY` / `HTTPS_CA` env vars read by
  `Gateway/config.js`). P3 (`gateway_sender.py`) now has real
  certificate-verification handling: by default it trusts nothing and
  skips verification for this local connection
  (`BLACKBONE_GATEWAY_VERIFY_TLS=false`, the default); set
  `BLACKBONE_GATEWAY_CA_FILE` to a real CA bundle if you ever put a proper
  certificate in front of Gateway, and verification will be enforced
  instead. **Unverified, worth checking directly:** Gateway's own
  `package.json` has an `npm run health` script that literally curls
  `http://` (not `https://`) the same port — that's an inconsistency in
  Gateway's own source, not something resolved here. Run `npm run health`
  yourself inside `Gateway\` and see whether it actually succeeds; that
  tells you definitively whether Gateway also tolerates plain HTTP on
  that one route.

- **ID format requirements** — Gateway rejects any message that doesn't
  follow these prefixes exactly (enforced by `Gateway/protocol.js` — this
  specific validation table itself has not been independently re-verified
  against that file's source by re-reading it directly, it's carried over
  from the prior version of this document; treat it as likely correct,
  not confirmed):

  | ID field | Required prefix | Example |
  |---|---|---|
  | `request_id` | `req_` | `req_a1b2c3d4e5f6...` |
  | `correlation_id` | `corr_` | `corr_a1b2c3d4e5f6...` |
  | `message id` | `msg_` | `msg_a1b2c3d4e5f6...` |
  | `session_id` | `sess_` | `sess_a1b2c3d4e5f6...` |
  | `idempotency_key` | `idem_` | `idem_a1b2c3d4e5f6...` |

  **Fixed:** the Orchestrator now generates a correctly-prefixed
  `request_id`, `correlation_id`, and `session_id` automatically for any
  job that doesn't already have one, before it ever reaches P3 — you no
  longer need to supply these yourself for a basic request, though you
  can (e.g. pass `context.idempotency_key` with an `idem_` prefix if you
  want duplicate-request protection).

# Instructions & Core Workflow

## 1. The application registry (`applications.json`)

Every application the Orchestrator can control is listed in
`BLACKBONE_ORCHESTRATOR\config\applications.json`. Each entry defines its
`name`, whether it's `permanent` (always-on) or on-demand, its
**`path`** (working directory — not `working_directory`, that was a
naming error in the previous version of this doc), `startup_command`,
`shutdown_command`, optional `health_url`/`health_command`/
`expected_port`, its `dependencies`, and (critically for retrieving data)
a `metadata.p2_module` pointing to the Python file in `P2/` that knows
how to pull data out of that specific app.

Currently configured applications (exact `name` to use in requests):

| Application | Always-on? | Has a working P2 data adapter? |
|---|---|---|
| Hermes | Yes | — (not a data-retrieval target, not started/stopped by the Orchestrator) |
| **Gateway** | Yes | No (infrastructure, not a data source) |
| TradingView MCP | Yes | **Yes** (`tradingview_mcp`) — fixed; previously had none |
| BB Terminal | No | **Yes** (`bb_terminal`) |
| OpenErii | No | **Yes** (`open_erii`) — session-managed, see section 5 |
| Osiris | No | **Yes** (`osiris`) |
| Trading Agent | No | **Yes** (`trading_agent`) |
| OpenAlice | No | **Yes** (`openalice`) — session-managed, see section 5 |
| **Backtesting.py** | No | **Yes** (`backtesting`) |
| Agent Reach | No | **Yes** (`agent_reach`) |
| AI Hedge Fund | No | **Yes** (`ai_hedge_fund`) |
| Quant Mind | No | **Yes** (`quant_mind`) |
| Blackbone Retriever | No | No (see Constraints & Pitfalls) |
| Vibe Trading | No | **Yes** (`vibe_trading`) |
| OpenStock | No | **Yes** (`openstock`) |

Corrections from the previous version of this document: the exact name is
`Gateway`, not `Blackbone Gateway` — sending the latter will fail to
resolve. There is no `Blackbone Orchestrator` entry in `applications.json`
at all; it isn't a P1-managed application, it's what's running the whole
pipeline, so that row has been removed. The Backtesting app's exact name
is `Backtesting.py` (with the suffix), not `Backtesting`.

An application can only be asked to **retrieve and deliver data** if it
has a working P2 adapter (right-hand column above). Any app without one
can still be started/stopped, but a data-retrieval job against it will
fail with `No P2 adapter is registered for application: <name>`.

## 2. Starting/stopping an application and getting its result

```json
POST http://127.0.0.1:51800/requests
{
  "operation": "start",
  "application": "BB Terminal"
}
```
Then poll:
```
GET http://127.0.0.1:51800/jobs/{job_id}
```
**Correction:** the terminal states are `"completed"` and `"failed"` —
**never `"succeeded"`**. The previous version of this document said to
poll for `"succeeded"`, which the Orchestrator will never actually
produce; a Hermes flow written against that value would hang forever
waiting for a state that doesn't exist. The full set of states a job can
be in: `queued`, `starting`, `retrieving`, `sending`, `cleaning_up`,
`completed`, `failed`, `cancelled`.

A job response also now includes `error_code` (a short machine-readable
reason when `state` is `failed` — one of `validation_error`,
`unknown_application`, `activation_error`, `retrieval_error`,
`delivery_error`, `session_error`, `timeout`, `cancelled`,
`internal_error`) and a `history` array showing every state transition
with a timestamp, useful for diagnosing exactly where a job got stuck.

## 3. What actually happens underneath

1. **P1 (lifecycle)** — starts the target application (and any
   dependencies `applications.json` declares for it) via PSI, and waits
   for it to report ready.
2. **P2 (retrieval)** — the Orchestrator loads that application's adapter
   from `P2/` and calls its `handle()` (or `retrieve()`/`execute()` as a
   fallback) to pull the actual data out of the running application.
3. **P3 (delivery)** — the retrieved data is wrapped into an envelope
   (with correctly-prefixed `request_id`/`correlation_id`) and POSTed
   over HTTPS to the Gateway. The current default delivery path is
   `https://127.0.0.1:64799/v1/orchestrator/events` (this is what the
   code actually uses — the previous version of this document said
   `/orchestrator/event`, which does not match; neither value has been
   independently confirmed against Gateway's own route table in
   `server.js`, since that file hasn't been read directly. If delivery
   consistently 404s, this path is the first thing to check against
   Gateway's real routes).
4. **The Gateway** validates the envelope and, if `destination` is
   `"hermes"`, attempts to hand the payload to whatever communication
   handler is registered for Hermes. **This last hop only succeeds if
   Hermes itself is online and registered as a destination with the
   Gateway** — see "Connecting Hermes to Gateway" below.
5. **P1 cleanup** — once the job finishes, P1 stops the on-demand
   application (and any dependencies it started) unless they're marked
   `permanent`. Gateway itself is never touched by this step — it's
   always-on and outside any per-job lifecycle.

## 4. The Gateway's own routes (for reference, from `Gateway/package.json`)

| Route | Notes |
|---|---|
| `GET /health` | Basic liveness check |
| `GET /status` | |
| `GET /capabilities` | What the Gateway supports |
| `GET /peers` | Registered communication peers (e.g. Hermes) — check this to confirm Hermes is actually connected |
| `GET /queue/status` | |
| `GET /transport/status` | |

The confirmed, currently-used delivery route is whatever
`DEFAULT_GATEWAY_ENDPOINT` resolves to in `gateway_sender.py` (see
section 3 above) — set `BLACKBONE_GATEWAY_ENDPOINT` to override it if
it turns out to be wrong.

**Honesty note carried over from the previous version:** the
Hermes→Gateway→Orchestrator trigger path (Hermes asking Gateway to
start a new Orchestrator job, rather than Hermes calling the
Orchestrator's own API directly) has not been traced or tested. Until
verified, trigger jobs against `127.0.0.1:51800` directly, as shown
throughout this document.

## 5. Session-managed applications (OpenErii, OpenAlice, CIEN)

These three route through a fourth component, **P4**, instead of the
plain P1→P2→P1 cycle every other app uses. The request shape is
identical — you still just POST to `/requests` with `application` set to
`OpenErii`/`OpenAlice`/`CIEN` — but underneath:

- P4 creates and starts a session for that application (which itself
  starts the application via P1).
- **The session is not closed at the end of the job.** It stays open
  until you explicitly close it:
  ```json
  POST http://127.0.0.1:51800/sessions/close
  { "session_id": "sess_xxxxxxxxxxxx" }
  ```
  Use the `session_id` field from the job response (or from
  `GET /jobs/{job_id}`) — it's the Gateway-facing id, prefixed `sess_`.
  The Orchestrator internally maps this to P4's own session id (a
  different id space, prefixed `session_`) automatically; you never need
  to know or handle P4's internal id yourself.

# Getting Data From Specific Things

## Getting data from BB Terminal

```json
POST http://127.0.0.1:51800/requests
{ "operation": "start", "application": "BB Terminal" }
```
Then poll `GET /jobs/{job_id}` until `state` is `completed` or `failed`.

## Getting data from OpenStock

Same shape, same pipeline:
```json
{ "operation": "start", "application": "OpenStock" }
```
It launches via `npm run dev` (a dev server), so its first startup after
being idle may take longer than BB Terminal's to become ready — that's
normal cold-start compilation, not a failure.

## Getting a specific file

There is no separate "give me file X" endpoint yet. File-oriented
retrieval is expected to go through **Blackbone Retriever**, but it has
**no `metadata.p2_module` configured**, so a data-retrieval job against
it will fail with `No P2 adapter is registered for application: Blackbone
Retriever` until one is written. Treat this as a known gap.

## Getting data from any other application

1. Confirm the application is listed in `applications.json` under its
   exact `name` (see the table in section 1).
2. Confirm that entry has a real `startup_command` (not a placeholder).
3. Confirm that entry has `metadata.p2_module` pointing to a real file in
   `P2/`.
4. If all three are true, the same request shape works:
   ```json
   { "operation": "start", "application": "<Application Name>" }
   ```
5. If any of the three is missing, that's the specific thing to fix — not
   a sign the whole system is broken.

# Other useful Orchestrator endpoints

These weren't in the previous version of this document:

| Endpoint | Purpose |
|---|---|
| `GET /jobs?state=&application=&limit=` | List/filter recent jobs instead of polling one at a time |
| `GET /metrics` | Job counts by state and application, component load errors, `gateway_ready` |
| `POST /jobs/{job_id}/cancel` | Cooperatively cancel a job still in flight |
| `POST /config/reload` | Reload `applications.json` without restarting the Orchestrator |
| `POST /components/reload` | Hot-reload P1/P3/P4 and clear the cached P2 adapters |

# Examples / Usage

**Example 1 — start BB Terminal and read back the result**
```
POST http://127.0.0.1:51800/requests
Content-Type: application/json

{ "operation": "start", "application": "BB Terminal" }
```
```
GET http://127.0.0.1:51800/jobs/job_xxxxxxxxxxxx
```

**Example 2 — stop an application that's currently running**
```json
{ "operation": "stop", "application": "BB Terminal" }
```

**Example 3 — check whether the Orchestrator (and Gateway) are healthy
before submitting anything**
```
GET http://127.0.0.1:51800/status
```
Look at both `"running"` and `"gateway_ready"`.

**Example 4 — start a session app and close it later**
```json
{ "operation": "start", "application": "OpenErii" }
```
```
GET /jobs/{job_id}   -> note the "session_id" field, e.g. "sess_abc..."
```
```json
POST /sessions/close
{ "session_id": "sess_abc..." }
```

# The "Let's Trade" Workflow (Twice-Daily Trade Research)

**Trigger:** the user says something like *"let's trade"*, *"lets trade"*,
or *"time to trade"*. This runs the full research-and-recommendation
procedure below and ends with exactly one structured output (section 4).

**What this is, and what it is not:** the Orchestrator and its P2 adapters
only retrieve raw data and run backtests — they do not contain any
trading strategy or decision logic. **The research, comparison, and
buy/sell decision in step 3 are your (Hermes') own reasoning, done over
whatever data steps 1–2 actually returned** — not something the
Orchestrator computes for you. This workflow never places a live order.
It produces a recommendation for the user to review and execute manually.
Historical backtest performance does not guarantee future results — say
so in the output, don't just imply it.

## 0. Session gate — exactly two sessions per day

- Sessions are **Morning** and **Evening**, one "let's trade" run each,
  no more. Use your own judgement for what counts as morning vs. evening
  for this user unless they've told you specific windows; if genuinely
  ambiguous, ask once and remember the answer.
- Before starting, check whether the relevant session has already run
  today. Log every completed run (date, session, the 3 symbols and
  calls given) to your memory (Obsidian vault, per your own
  `config.yaml`) so this persists across restarts.
- If the applicable session already ran today: don't re-run the research.
  Show the user the result you already gave them for that session, and
  say when the next session opens.

## 1. Live research pass

Pull fresh data through the Orchestrator (`POST /requests`, then poll
`GET /jobs/{job_id}` until `completed`/`failed` — see "Instructions &
Core Workflow" above for the exact request shape) from each application
below, roughly in this order. Treat every one of these as
**best-effort**: if an application fails to start or returns nothing
useful, note it as unavailable in the final output and continue with the
rest — one app being down should never block the whole workflow.

| # | Application | What to ask it for |
|---|---|---|
| 1 | TradingView MCP | `"operation": "status"` — only operation confirmed working right now (live-chart/CDP connectivity check). If/when more TradingView MCP operations exist, use them here for live price/indicator reads. |
| 2 | BB Terminal | `"operation": "start"`, then whatever it returns |
| 3 | OpenStock | `"operation": "start"` |
| 4 | Osiris | `"operation": "start"` |
| 5 | Trading Agent | `"operation": "start"` |
| 6 | Vibe Trading | `"operation": "start"` |
| 7 | AI Hedge Fund | `"operation": "start"` |
| 8 | Quant Mind | Try `"operation": "news"` and `"operation": "mind"` for research/sentiment input |

**Honesty note:** beyond `start`/`status`, the exact operation vocabulary
for BB Terminal, OpenStock, Osiris, Trading Agent, Vibe Trading, and AI
Hedge Fund hasn't been catalogued in this document yet. Most of these
adapters expose a `"capabilities"` and/or `"command_info"` operation —
call `{"operation": "capabilities", "application": "<name>"}` against
each one first to see its real menu, rather than guessing an operation
name that might not exist. Once you've mapped these for real, this table
should be updated with the actual operation names instead of `start`.

## 2. Backtest pass

For every candidate symbol/setup the research pass surfaced, validate it
against history before it's allowed into the final 3:
```json
{ "operation": "start", "application": "Backtesting.py" }
```
followed by whatever specific backtest/validate operation that adapter
exposes for the setup in question (check its `"capabilities"` the same
way). Don't include a symbol in the final output if it hasn't at least
been run through this step.

## 3. Synthesis — your own reasoning over steps 1–2

- **Selection rule:** exactly 3 symbols, always. Exactly **2 must be
  forex pairs**. The 3rd is whichever of **metal, crypto, or bond**
  looks strongest that session based on what the research pass actually
  returned — don't default to the same one every time out of habit.
- **Decision rule:** for each of the 3, decide **BUY** or **SELL** based
  on what the combined research + backtest data actually supports. If the
  data is mixed or thin for a candidate, say so plainly in the rationale
  rather than forcing a confident call.
- **Timing rule:** for each symbol, decide **Immediate** or a **specific
  scheduled time** later in that session's window (state the time and
  timezone). Only symbols marked Immediate get a stop loss / take profit
  in the output (see below) — a scheduled entry doesn't need one until
  it's actually about to trigger.

## 4. Required output structure

Give exactly this shape at the end of every "let's trade" run — don't
freestyle the format:

```
BLACKBONE TRADE SESSION — <Morning/Evening> — <date>
Sources used: <list of apps that actually returned usable data>
Sources unavailable: <list, or "none">

1) <SYMBOL> — <Forex/Metal/Crypto/Bond> — <BUY/SELL>
   Entry: Immediate | Scheduled <HH:MM TZ>
   Stop Loss: <n> pips (<price level>)      [only if Immediate]
   Take Profit: <n> pips (<price level>)    [only if Immediate]
   Rationale: <2-3 sentences, name which sources/backtest result support this>

2) <SYMBOL> — ...same structure...

3) <SYMBOL> — ...same structure...

Next session: <Morning/Evening>, opens <when>
Reminder: this is research output for manual review, not an executed trade.
```

## 5. Pips vs. points — don't misapply forex math

"Pips" is a forex-specific unit (0.0001 for most pairs, 0.01 for
JPY-quoted pairs). Metals (e.g. XAU/USD), crypto, and bonds are not
normally quoted in pips — use **points** or plain price levels for those
instead, and say so explicitly in the output rather than forcing a "pips"
number onto an instrument where it doesn't apply. Always show both the
pip/point count and the actual price level (e.g. "30 pips (1.0850 →
1.0880)") so the number is checkable at a glance.



This is the one hop this document still can't fully confirm end-to-end,
because it lives entirely in Hermes' own configuration, which hasn't been
directly inspected here. What's known:

- Gateway pre-registers a peer named `hermes` with permissions `request`,
  `response`, `event`, `status`.
- Two Hermes skills already exist for this — `blackbone_gateway` and
  `blackbone_retriever` (hub-installed, source `url`). Run
  `hermes skills inspect blackbone_gateway` to see exactly what they
  expect; that's a more authoritative source than this document for the
  Hermes-side half of the connection.
- Until Hermes registers itself, deliveries will fail with
  `GATEWAY_ERROR: No communication handler registered for destination
  'hermes'.` — this is expected, not a bug, and resolves itself once
  Hermes connects.

See the separate "Connecting Hermes to Gateway — steps" instructions for
the concrete sequence.

# Constraints & Pitfalls

- **Hermes must be online for the final hop to succeed.** Everything up
  to and including Gateway acceptance can work with Hermes off; the one
  expected failure with Hermes off is the `GATEWAY_ERROR` above.
- **Never invent a `startup_command`.** Every command in
  `applications.json` should come from a real, previously-verified launch
  script — a guessed command produces a new, confusing failure.
- **Three always-on apps have no P2 adapter**: Gateway, and (still, as of
  this version) nothing else in that category needs one — TradingView MCP
  now has one (`tradingview_mcp`), so it no longer belongs in this list.
  Don't add a fake `metadata.p2_module` for Gateway; it's infrastructure,
  not a data source.
- **Blackbone Retriever has no P2 adapter yet.** Don't rely on it for
  data retrieval until one is written and wired up.
- **ID prefixes are enforced by Gateway** (`req_`, `corr_`, `msg_`,
  `sess_`, `idem_`) — but the Orchestrator now generates the first three
  for you automatically if you don't supply them, so this mostly matters
  if you're constructing a message by hand rather than going through
  `/requests`.
- **Gateway is HTTPS-only** with a local/self-signed certificate on port
  `64799`. Don't attempt a plain HTTP connection to it (except possibly
  `/health` — see the unresolved note in Requirements & Setup).
- **Respect Gateway's `retryable` flag** on a failed delivery response.
  `"retryable": false` means don't retry that same request.
- **On-demand applications auto-stop after each job.** Session apps
  (OpenErii, OpenAlice, CIEN) are the exception — they stay open until an
  explicit `close_session` request.
- **The Hermes→Gateway→Orchestrator trigger path is unverified.** Trigger
  jobs against the Orchestrator's own API (`127.0.0.1:51800`) directly.
- **"Let's trade" is capped at two runs per day** (Morning, Evening) —
  don't re-run the research a third time same day; show the existing
  result instead. See "The 'Let's Trade' Workflow" above.
- **Never place a live order from the "let's trade" workflow.** It only
  produces a recommendation (symbols, direction, timing, stop loss/take
  profit) for the user to act on manually.
