# Sluice

**Multi-entity cash positioning agent**

**Syndicate by Maximor — Track 2: Autonomous Office of the CFO**

Build window: **5 Sep 2026, 21:30 IST → 7 Sep 2026, 03:30 IST** (~30 hours).

## The workflow being automated

Every morning, a treasury analyst at a multi-entity company opens ~10 bank
portals, writes down balances, checks which subsidiaries are about to dip below
their required minimums, and decides which entity should send cash to which —
subject to loan covenants, intercompany lending limits, FX cost, and settlement
timing. It takes 1–2 hours daily, it's error-prone, and a mistake means either a
covenant breach or an overdraft.

We automate it end to end, including the cases where there is no clean answer.

## Why this isn't an LLM wrapper

Remove the LLM and there is still a system: a ledger, a constraint model, and a
min-cost-flow solver that produces a verifiable optimal transfer plan. The LLM
sits on top of real machinery, doing the parts a solver can't — diagnosing *why*
a plan is infeasible and what to trade off, writing the approval memo a CFO
signs, and absorbing human overrides as durable constraints.

The plan itself is checkable. Every run either satisfies every constraint or
explicitly escalates — there is no "the model said so."

---

## Status

| Step | State |
|---|---|
| 1. Data model + seeded company | **done** — `schema.sql`, `models.py`, `seed.py`, `fx.py` |
| 2. Projected positions | **done** — `positions.py` |
| 3. Solver | next — the spine |
| 4. Naive baseline | not started |
| 5. Infeasibility diagnosis + ranked remedies | not started — *least cuttable* |
| 6. Memo generation | not started |
| 7. Streamlit UI | not started |
| 8. Neatlogs traces + metrics panel | not started |
| 9. Learned rules loop | stretch — only if ahead by hour 20 |

**Cut from the critical path:** covenant extraction from PDF. It was the biggest
time sink for the least payoff. Demo beat 2 works just as well with a new
covenant arriving as a structured drop rather than a parsed document.

---

## Data model

Implemented in `src/sluice/schema.sql`. Money is integer minor units
everywhere below the UI layer so the solver cannot accumulate float drift.

```
entity          id, name, country, functional_currency
bank_account    id, entity_id, bank, currency, balance, purpose
covenant        entity_id, kind, threshold, currency, hardness,
                source_doc, source_quote
cash_forecast   entity_id, day, date, net_flow, note     # 14-day, daily
ic_agreement    lender_id, borrower_id, max_limit, rate_bps, permitted, reason
fx_rate         base, quote, mid, spread_bps
transfer_cost   from_bank, to_bank, fixed_fee, settlement_days
learned_rule    origin_run_id, rule_text, constraint_json, created_at
```

Two fields carry more weight than they look:

**`covenant.hardness`** (`hard` | `soft`). Ireland's term-loan floor is
contractual and may never be breached; the internal policy minimums can be
relaxed with treasurer sign-off. Without this distinction the remedy ranker in
demo beat 3 has nothing to rank on.

**`ic_agreement.permitted` + `reason`.** Prohibited lending pairs carry their
justification in prose — Ireland cannot lend upstream to the US parent (IRC
§956 deemed dividend), Singapore→Germany is suspended pending an IRAS transfer
pricing review. These are what make the naive "fund everything from HQ" answer
wrong, and the reasons are what the memo quotes.

## The seeded company

**Meridian Systems** — six entities, three currencies, ten accounts.
`python -m sluice.seed --scenario {base,covenant_shock,infeasible}`.

Verified output of `positions.summarise()`:

| | base | covenant_shock | infeasible |
|---|---|---|---|
| UK shortfall | 710k GBP | 710k GBP | 710k GBP |
| DE shortfall | 970k EUR | 970k EUR | 3,570k EUR |
| IE | lends 4.2m EUR | **needs 335k** | needs 335k |
| US free cash | 2.7m USD | 2.7m USD | 220k USD |

Under `covenant_shock`, Ireland doesn't merely stop lending — it flips from the
largest lender in the group to a borrower. That is a sharper demo moment than
"the cheap route got more expensive." Under `infeasible` the group needs ~5.1m
USD against ~1.4m available, so it fails decisively rather than marginally.

## The solver

Minimum-cost flow over an (entity × day) grid.

- **Decision variables:** transfer amount from account *i* to account *j* on day *d*
- **Objective:** minimize FX spread + wire fees + intercompany interest accrued
- **Constraints:**
  - closing balance ≥ covenant floor, per entity per day
  - cumulative intercompany exposure ≤ `max_limit` per lender/borrower pair
  - no transfers across `permitted = 0` pairs
  - settlement lag: a transfer sent on day *d* lands on day *d + settlement_days*
  - non-negativity, no overdrafts on the sending side

PuLP (CBC). Returns either an optimal plan or `INFEASIBLE` — and the infeasible
branch is the interesting one.

## Where the agent earns its keep

**Infeasibility diagnosis.** Solver says INFEASIBLE. The agent identifies the
binding constraint, then generates ranked remedies with the business cost of
each: draw on the revolver (cheap, consumes committed capacity), breach a soft
covenant (free, needs treasurer sign-off), delay a payable (free, damages a
vendor relationship), breach a hard covenant (never — escalate immediately).
Ranking these requires judgment the solver has no model for.

**The memo.** A one-page approval doc: what's moving, why, what it costs, what
was rejected and why, which constraints came close to binding.

**Learning from overrides.** The CFO rejects "delay payable to Vendor X." The
agent writes a `learned_rule` — *never delay payables to strategic vendors* — as
an actual constraint injected into the next solve. Re-run and it is gone from
the option set.

## Demo beats

1. **Clean day** (`base`). One UK shortfall, one German shortfall. Plan
   generated, cheap route through Ireland, memo written. Show cost against the
   naive baseline — that delta is the headline number.
2. **Covenant shock** (`covenant_shock`). Ireland's floor jumps to EUR 6.5m.
   The group's biggest lender becomes a borrower; the plan reroutes through the
   US and says exactly why it changed its mind.
3. **Infeasible** (`infeasible`). Solver fails. Agent diagnoses the binding
   constraint, ranks three remedies, escalates with a recommendation instead of
   silently producing garbage.
4. **Learning.** Human rejects one remedy. Re-run — it never resurfaces.

Beat 3 is the one that wins the track. Most submissions will show the happy
path; an agent that *knows when it can't solve something* and escalates
coherently is the differentiator.

## Metrics to put on screen

| Metric | How measured |
|---|---|
| Cost saved | Plan cost vs. naive fund-from-HQ baseline, per run |
| Constraint violations | Must be 0 — assert on every run, show the check |
| Autonomy rate | % of runs completed without human escalation |
| Time to plan | Wall clock vs. the 1–2 hr manual process |
| Override recurrence | Times a rejected remedy reappears — must trend to 0 |

These double as the README's "what improved across iterations" section, which
is a submission requirement.

---

## Stack

- Python 3.11 + SQLite, seeded fake data
- PuLP (CBC) for the solve
- **TensorMux** for inference — the hackathon's inference partner covers model
  calls for the build window. `src/sluice/llm.py` wraps their
  OpenAI-compatible endpoint; provider is env-configurable.
- Streamlit for the demo UI — balances, proposed plan, approve/reject
- **Neatlogs** for tracing agent runs and showing improvement over time
- Mock `BankConnector` interface, so swapping in a real bank API is visibly a
  config change and not a rewrite

### TensorMux constraints that shape the design

**GLM-4.7-Flash has a 32k context window.** Small enough to matter. The raw
forecast is 84 rows plus 30 intercompany agreements plus covenants and FX —
dumping all of it crowds the window and degrades reasoning. Diagnosis and memo
prompts get the *summarised* positions (six rows) and *only the constraints that
bind*. This is good discipline regardless: it is also what makes the memo
readable.

**50M tokens ≈ 140 agent tasks**, and agent loops resend the whole conversation
each step. Our design is mostly single-turn — diagnose, rank, write — rather
than a long tool-calling loop, which keeps us well inside budget. It is also a
reason not to reimplement the solver as an agentic loop when a solver call does
it correctly, deterministically, and for free.

Temperature is 0 by default. Treasury output that changes between identical runs
is not auditable.

---

## AO usage

AO is a build-time requirement, not a product integration: it orchestrates
parallel coding agents across the repo, spawning workers into isolated git
worktrees with a kanban board over them. The submission must explain how it was
used, and **the demo video must show the AO sessions**.

Practical consequences:

- Set AO up **before** writing more code. The requirement is explicitly "from
  the start of your build," and sessions are counted from the video —
  retrofitting will not produce that history.
- Screen-record the AO board periodically as work proceeds, rather than
  reconstructing it at hour 29.

**Build order, re-cut for parallelism.** Steps 1–4 are a serial spine — the
schema, the solver, and the baseline all depend on each other. Once the solver
lands, four tracks fan out into separate worktrees with almost no overlap:

| Worker | Owns |
|---|---|
| A | Infeasibility diagnosis + ranked remedies |
| B | Memo generation |
| C | Streamlit UI + approve/reject |
| D | Neatlogs tracing + metrics panel |

That is a far better demo-video story than a sequence of solo sessions.

## Submission requirements

- **Devpost is the only channel** — Discord showcase posts do not count
- Public GitHub repo
- Demo video showing both the product and the AO sessions used to build it
- README covering: what it does, how to run it, the track, the agent workflow
  built, what improved across iterations, and any live links

## Scope cuts to survive the timeline

- No real bank integrations — mock connector only
- 14-day horizon, daily granularity, no intraday
- Six entities, three currencies
- Static seeded FX rates, not live
- No PDF covenant extraction; covenants arrive structured
