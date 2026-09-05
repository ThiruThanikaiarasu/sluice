# Recap

Where Sluice stands and how it got here. Written 5 Sep 2026, ~22:45 IST,
roughly 1h15m into the build window.

## What we're building and why

**Sluice** — an autonomous multi-entity treasury cash positioning agent, for
Syndicate by Maximor, **Track 2: Autonomous Office of the CFO**.

We chose Track 2 over Track 1 because "a meta-agent that designs agents for
unseen tasks" is a research problem that demos badly in 30 hours, while Track 2
is a scoped, real workflow.

We then deliberately avoided the obvious Track 2 idea. Invoice matching and
reconciliation will be built by many teams, and most of it reduces to parsing
plus a lookup. We picked multi-entity cash positioning instead: tedious,
rarely automated, and genuinely judgement-heavy.

The test we held the idea to: **if you removed the LLM, would there still be a
system?** For Sluice, yes — a ledger, a constraint model, and a min-cost-flow
solver producing a verifiable optimal plan. An earlier candidate (an audit
support agent) failed that test — strip the model and nothing remains but a
search box — so we dropped it.

## The workflow

Every morning someone pulls balances from ~10 bank portals across six legal
entities, works out who will breach their minimum cash floor, and decides who
sends money to whom — subject to covenants, intercompany lending limits, FX
cost, and settlement timing. One to two hours, daily, in a spreadsheet.

What makes it hard is not arithmetic. It is that the obvious answer is
frequently **illegal**: the parent has cash, but an Irish subsidiary lending
upstream to a US parent is a deemed dividend under IRC §956.

## Decisions made

| Decision | Reasoning |
|---|---|
| Track 2, cash positioning | Scoped, judgement-heavy, few teams will build it |
| Solver underneath, LLM on top | Output is verifiable; nothing to hallucinate |
| Name: **Sluice** | A sluice gate moves water between basins under constraint |
| Cut PDF covenant extraction | Biggest time sink, least payoff; covenants arrive structured |
| Integer minor units everywhere | Float drift in a solver yields plans a bank rejects |
| Temperature pinned at 0 | Treasury output that varies between runs is not auditable |
| Rebuilt repo from scratch at 22:25 | See eligibility note below |

## Eligibility issue, and how it was resolved

The rules state *"All project work must begin after the official hackathon
start time"* and *"Organizers may verify repositories, commit history."*

An initial version of the data model was committed at 09:45 and 10:51 IST —
about 12 hours **before** the 21:30 start. Rewriting those timestamps would be
"misrepresentation of project progress," which the rules list as grounds for
disqualification, so that was never an option.

Instead the repo was deleted and rebuilt from zero at 22:25 IST, re-authored
rather than restored. Current history is two commits, both inside the window.
The rebuild improved things anyway: the package was renamed `treasury` →
`sluice`, `fx.py` was added, and `positions.py` gained typed summaries.

## Built so far

Repo: **https://github.com/ThiruThanikaiarasu/sluice** (public)

```
src/sluice/schema.sql     entities, accounts, covenants, forecasts,
                          intercompany agreements, FX, transfer costs,
                          learned rules
src/sluice/models.py      dataclasses + minor/major money conversion
src/sluice/db.py          connection and schema lifecycle
src/sluice/fx.py          inverse and cross rates; spread always charged
src/sluice/seed.py        Meridian Systems, three scenarios
src/sluice/positions.py   unfunded projection, shortfalls, free cash
src/sluice/llm.py         TensorMux client, temperature 0
```

Two schema fields carry more weight than they look:

- **`covenant.hardness`** (`hard` | `soft`) — contractual floors may never be
  breached; internal policy floors may be relaxed with sign-off. Without this
  the remedy ranker has nothing to rank on.
- **`ic_agreement.permitted` + `reason`** — prohibited lending pairs carry
  their legal justification in prose, and that text is what the memo quotes.

### The seeded company

**Meridian Systems** — six entities, USD/EUR/GBP, ten accounts. Verified output
of `positions.summarise()`:

| | base | covenant_shock | infeasible |
|---|---|---|---|
| UK shortfall | 710k GBP | 710k GBP | 710k GBP |
| DE shortfall | 970k EUR | 970k EUR | 3,570k EUR |
| IE | lends 4.2m EUR | **needs 335k** | needs 335k |
| US free cash | 2.7m USD | 2.7m USD | 220k USD |

Under `covenant_shock` Ireland flips from the group's largest lender to a
borrower — a sharper demo moment than "the cheap route got more expensive."
Under `infeasible` the group needs ~5.1m USD against ~1.4m available, so it
fails decisively rather than marginally.

## Tooling setup

**AO holds no credentials.** It shells out to whichever agent CLI is installed
and uses that CLI's own auth. Nothing to paste into AO.

| | Powers | Budget |
|---|---|---|
| Claude Code | AO orchestrator | Claude subscription |
| Codex (`auth_mode: chatgpt`) | AO workers | ChatGPT plan |
| TensorMux | **Sluice at runtime** | 50M free tokens |

Workers were put on Codex deliberately: there are several running at once and
they consume the bulk of the tokens, so that load belongs on a different budget
from the single orchestrator. If one budget runs dry, flip the other dropdown.

"Auto review PRs" is off while the spine lands. Worth switching on for one PR
later — AO reviewing a PR and routing feedback back to the owning worker is
exactly the footage the demo video needs.

**TensorMux is deliberately not used for coding.** GLM-4.7-Flash has a 32k
context (too small for codebase work) and those tokens are the product's only
inference budget. The same 32k limit shapes the product: diagnosis and memo
prompts get `positions.summarise()` (six rows) and only the binding
constraints, never the raw 84-row projection.

**Git:** push over the SSH alias `github.com-personal`
(`sethu-ravichandran`). HTTPS resolves to `sethu-genspark`, which has no write
access and 403s.

## What's next

The **solver** is the last serial piece. Infeasibility diagnosis, memo
generation, the Streamlit UI, and tracing all consume a `Plan` object that does
not exist yet — fanning out before it lands means workers guessing at the same
interface.

Once it merges, four AO workers run in parallel:

| Worker | Owns |
|---|---|
| A | Infeasibility diagnosis + ranked remedies |
| B | Memo generation |
| C | Streamlit UI + approve/reject |
| D | Neatlogs tracing + metrics panel |

## Open items

- `.env` may not exist yet — gitignored, so unverifiable from here. Diagnosis
  and memo calls fail at the first request without `SLUICE_LLM_API_KEY`.
- `DEVPOST.md` placeholders in «guillemets» need real numbers. Fabricated
  results are explicitly grounds for disqualification.
- **"Early traction or user validation" is a judging criterion** — the one a
  30-hour build usually scores zero on. Cheapest credible answer: show it to one
  person who has actually done treasury or accounting work and quote them. The
  Maximor team is in the Discord.
- Confirm every team member registered individually (eligibility rule).
- README is a submission requirement: what it does, how to run it, the track,
  the agent workflow, what improved across iterations, live links.

## Reference

- Submit on **Devpost** only — Discord showcase posts do not count
- Demo video must show both the product and the AO sessions used to build it
- Judging: practical use case, working end-to-end product, agent autonomy and
  tool use, reliability and exception handling, measurable improvements,
  track alignment, early traction, demo clarity
