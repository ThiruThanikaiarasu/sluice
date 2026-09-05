# Devpost submission draft

Paste the "About the project" section below into Devpost. Anything in
«guillemets» is a placeholder that needs a real number or observation before
submitting — do not ship them as-is.

---

## Inspiration

Most companies of any size are not one company. They are six or ten legal
entities, in different countries, holding cash in different currencies, each
with its own bank and its own minimum balance it is contractually forbidden to
drop below.

Someone has to sit down every morning, pull the balances, work out which
subsidiary is about to run dry, and decide who should send money to whom. It
takes one to two hours. It is done in a spreadsheet. Getting it wrong means an
overdraft or a covenant breach, and covenant breaches are the kind of mistake
that ends up in a board pack.

What makes it genuinely hard is not the arithmetic — it is that the obvious
answer is frequently illegal. The parent company has plenty of cash, but an
Irish subsidiary lending upstream to a US parent is a deemed dividend under IRC
§956. Singapore could cover Germany, except intercompany lending between them
is suspended pending a transfer pricing review. The constraints are written in
loan agreements, in prose, and they change.

We wanted to automate the whole judgement, not just the arithmetic.

## What it does

Sluice takes over the daily multi-entity cash positioning run end to end.

It projects every entity's cash position across a 14-day horizon, identifies who
will breach their floor and when, and then solves for the cheapest set of
transfers that keeps every entity above its minimum — respecting FX spreads,
wire fees, intercompany lending limits, settlement lag, and the pairs that are
prohibited from lending to each other at all. It writes the treasurer a one-page
memo explaining what is moving, what it costs, and what was rejected and why.

The part we care most about is what happens when there is no answer.

When no legal set of transfers covers every shortfall, Sluice does not produce a
plausible-looking plan anyway. It identifies the binding constraint, works out
which remedies are actually available — draw on the revolver, breach a soft
internal policy with sign-off, delay a payable — ranks them by business cost,
and escalates to a human with a recommendation. Contractual covenants are never
in the option set. When a human rejects a remedy, that rejection is written back
as a constraint, and it never resurfaces in a later run.

## How we built it

The core insight is that this is not a language model problem wrapped in a
finance story. Underneath is a real optimisation problem, and we built it as
one.

**The solver.** Minimum-cost flow over an (entity × day) grid, in PuLP. Decision
variables are transfer amounts between accounts on a given day; the objective
minimises FX spread plus wire fees plus intercompany interest. Constraints
enforce the covenant floor per entity per day, cumulative exposure limits per
lender/borrower pair, prohibited pairs, and settlement lag — money sent on day
*d* only lands on day *d + n*. It returns a provably optimal plan or it returns
INFEASIBLE. There is no middle ground and nothing to hallucinate.

**Where the model earns its place.** The solver cannot tell you that breaching
an internal policy minimum is recoverable while breaching a term loan covenant
is not. It cannot read an infeasibility certificate and explain it to a CFO. It
cannot weigh a damaged vendor relationship against a drawn revolver. Those are
the three jobs we gave the model: diagnose infeasibility, rank remedies, write
the memo.

**Data.** A fictional six-entity group, Meridian Systems, across USD, EUR and
GBP with ten bank accounts. Three seeded scenarios drive the demo: a normal day,
a covenant shock, and an unsolvable day. Prohibited lending pairs carry their
legal justification in prose, because that justification is what the memo has to
quote.

**Inference.** TensorMux, using GLM-4.7-Flash. Temperature is pinned at zero
throughout — treasury output that changes between identical runs is not
auditable.

**Money is integer minor units everywhere** below the presentation layer. Float
drift in a constraint solver produces plans that are wrong by a cent and
rejected by a bank.

## How we used AO

«Fill from your actual AO history before submitting — session count, which
workers ran in parallel, and one concrete example of a CI failure or review
comment that AO routed back to the owning agent.»

We built through AO from the start. The schema, solver and baseline are a serial
spine — everything depends on the data model — so those ran as sequential
sessions. Once the solver landed, the remaining work fanned out into parallel
worktrees that barely touch each other: infeasibility diagnosis, memo
generation, the Streamlit review UI, and tracing plus the metrics panel. «N»
sessions total across the build.

The kanban view mattered more than we expected. With four workers running, the
thing that actually goes wrong is not agent quality, it is losing track of what
is in flight.

## Challenges we ran into

«Replace with what actually bit you — this section is where judges look for
evidence you built the thing rather than described it. Candidates below, keep
the ones that are true.»

**Designing a dataset that fails correctly.** Making a scenario infeasible is
easy; making it infeasible for an *interesting* reason is not. Our first attempt
had the parent company holding so much cash that it trivially solved every
shortfall, which made the constraint model pointless. We had to tune balances,
flows and covenant floors until the cheap route was blocked for a legal reason
rather than an arithmetic one. Under our covenant shock scenario the group's
largest lender flips to being a borrower, which is the moment the whole idea
becomes legible.

**Context budget.** GLM-4.7-Flash has a 32k window. The raw inputs — 84 forecast
rows, 30 intercompany agreements, covenants, FX — crowd it and degrade the
reasoning. We pass summarised positions and only the constraints that actually
bind. This turned out to improve the memos as well: the model stops narrating
the data and starts explaining the decision.

**Settlement lag in the constraint model.** «...»

## Accomplishments that we're proud of

«Needs real numbers from your runs.»

- Zero constraint violations across «N» runs, asserted programmatically on every
  plan rather than eyeballed
- «X%» cheaper than the naive baseline of funding every shortfall from the
  parent company
- «N» seconds to produce a plan and memo, against a one-to-two hour manual
  process
- An agent that escalates rather than guessing when the problem has no solution

## What we learned

The most useful design question we asked was: *if we removed the language model,
would there still be a system?* For this project the answer is yes — a ledger, a
constraint model, and a solver. That test kept us honest, and it is why the
output is verifiable rather than merely fluent.

The second thing: the interesting engineering in an agent is almost never the
happy path. Building the infeasible branch — diagnosing the binding constraint,
ranking remedies, knowing which lines can never be crossed — took longer than
building the solver, and it is the only part that would matter to a real
treasury team.

## What's next for Sluice

- Real bank connectivity. The `BankConnector` interface is mocked but isolated,
  so swapping in a live API is a config change rather than a rewrite.
- Covenant extraction from loan agreement PDFs, so the constraint set maintains
  itself as documents are amended.
- Intraday positioning, and same-day cutoffs by bank and currency.
- Learning across runs, so the remedy ranker reflects a specific treasurer's
  revealed preferences and not a generic policy.

---

## Built with

```
python
sqlite
pulp
linear-programming
streamlit
tensormux
glm-4.7-flash
ao
neatlogs
openai-api
```
