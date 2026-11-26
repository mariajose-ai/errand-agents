# Design Decisions

Settled design choices for the simulator, planner, and experiments. Each entry records
what was decided, why, and what it affects in the code. Changes from the original
proposal are noted. Progress and dates of work are in [LOG.md](LOG.md).

Status key: **Settled** · **Open**

---

## D1. Waiting is allowed — Settled (2026-11-25)

**Decision:** An agent that reaches a place before it opens may wait outside until
opening time. Arriving early is not a violation. Waiting time counts toward the cost
(see D2). There is no cap on waiting for now.

**Why:**
- It matches real errands: people do wait for a shop to open.
- It keeps the search simple and exact. With waiting, arriving earlier at a place is
  never worse than arriving later, because you can always wait. So the planner only
  needs to keep the earliest time for each (location, remaining errands) state.
- Without waiting, an early arrival could fail at a closed place, so time would have to
  become part of the state. That makes the state space much larger and the code more
  complex.

**Excessive early arrival:** handled by the objective, not by a rule. Because waiting
counts toward the finish time, a plan that waits a long time is more expensive, and the
search picks a different order whenever one exists.

**Affects:**
- `world.service_start` and `Belief.service_start`: return `max(arrival, open)`.
- `evaluator.simulate`: an early arrival is valid; the service starts at opening.
- `search.astar`: dominance pruning (`best_g` keeps the earliest time per state) is exact
  under this decision.

**Possible later change:** a maximum-wait limit (for example 30 minutes) as a hard
constraint. Not in the core study.

---

## D2. Objective is earliest finish time — Settled (2026-11-25)

**Decision:** The best plan is the feasible plan that reaches the final destination
earliest. Travel, waiting, and service time all count, because they all move the clock.

**Change from the proposal:** the proposal said "minimum travel time". That would make
waiting free, which brings back the problem of plans that arrive very early and wait.
Earliest finish avoids this and matches what the oracle already computes.

**Why:**
- One number captures everything the user experiences (when they are done).
- The search cost `g` is already clock time, so no code change is needed in `search.py`.
- It is consistent with D1: dominance pruning is exact for this objective.

**Affects:**
- `search.astar` / `search.oracle`: no change; they already minimize arrival time.
- `evaluator.score`: the **optimality gap** metric is `finish_gap` (agent finish minus
  oracle finish, in minutes). `travel_gap` stays as a secondary diagnostic only.
- Proposal and write-up: say "optimality gap in finish minutes" instead of
  "travel minutes".

---

## D3. Submit feedback is off in the main experiment — Settled (2026-11-25)

**Decision:** In the main experiment, `submit_plan` gives no feedback. It does not say
whether the plan is valid or which rules it broke. The first plan an agent submits is
its final answer and ends the scenario. Feedback on (agents see violations and may
resubmit) is kept as an optional side comparison, not part of the core study.

**Why:**
- The research question is whether each agent can *plan* correctly under tool failures,
  not whether it can patch a plan after being told what was wrong.
- Feedback lets an agent guess, read the violations, and fix them. That mostly helps
  ReAct and hides how good its planning actually was.
- Feedback comes from the evaluator, which checks against the true world. Feedback would
  therefore give agents ground-truth information that bypasses the injected tool
  failures, which weakens the stress test.
- With feedback off, the comparison is clean: both agents get one shot, the same tools,
  and the same failures.

**What still counts as fair:** an agent may check its own plan using normal tool calls
before submitting. For example, the search agent re-looks-up hours it had to assume
were open. Those calls count against the budget and can fail like any other call. This
is planning, not feedback.

**Affects:**
- `tools.ToolEnv`: change the default to `submit_feedback=False`. With feedback off,
  `submit_plan` returns only an acknowledgement, and later submissions are rejected
  (or ignored), so the first submission is final. Today the code keeps the *last*
  submission, so this needs a code change.
- `agents/search_agent.py`: the replan-after-rejection loop (`belief.blocked`) only runs
  in the feedback-on side comparison. Verify-before-submit stays in both settings.
- ReAct agent (not built yet): its prompt must say that submission is final when
  feedback is off.
- Results: report the main numbers with feedback off. If the side comparison is run,
  report it separately and label it clearly.

---

## D4. Failure kinds: keep the 50/50 split, tag three kinds — Settled (2026-11-25)

**Decision:** Keep the current split. When a call is chosen to fail, half the time it
fails loudly (an error) and half the time it comes back incomplete. Silent drops stay in
the main experiment. Every injected failure is tagged with one of three kinds:

| Kind | What happens | Example | What the agent can see |
|---|---|---|---|
| `error` | The call fails outright (timeout, 5xx). | `walk_time(Union, Pharmacy)` raises a transient error. | Yes. It knows to retry. |
| `missing_field` | The call answers, but a field is empty. | `get_hours(PrintShop, Tue)` returns `hours: null`. | Yes, if it checks the field. |
| `silent_drop` | `find_places` returns a list with one place removed. | `find_places(print)` omits `PrintShop`. | No. The response looks normal. |

**Why:**
- The three kinds test different skills. Errors test retrying. Missing fields test
  noticing a gap and treating it as unknown rather than guessing. Silent drops test
  whether an agent's plan still holds up when it doesn't know something went wrong.
- Silent drops are realistic (incomplete search results happen with real tools) and are
  probably the most interesting result, so they belong in the main test.
- Tagging lets the results be broken out by kind, so one hard kind does not hide how
  the agents handle the others.

**How tagging works:** nothing is tagged by hand. The failure injector already chooses
the kind when it breaks a call, so it writes the kind into the trace at that moment.
Each trace entry records the scenario, the call number, the tool, the kind, and what was
removed (for example, which place a silent drop left out). Analysis groups results by
the `kind` field.

**Fairness rule:** the tags live only in the environment's trace. Agents never see them.

**Affects:**
- `tools.ToolEnv._perturbation` / `find_places`: today there are only two labels,
  `error` and `missing`. Split them so `find_places` records `silent_drop` and the other
  tools record `missing_field`.
- Tag a `silent_drop` only when a place was actually removed. Today, if a service has
  only one place, nothing is dropped but the trace still says `missing`. Record that
  case as no effective failure so the counts stay honest.
- Trace entries: add the dropped item (place name) for `silent_drop`.
- Results: report CSR and tool calls by failure kind, in addition to by failure rate.

---

## D5. Limits: 80 tool calls, 2 retries — Settled (2026-11-25)

**Decision:**
- **Budget:** 80 tool calls per scenario, matching the draft code. 50 was considered
  first, but the current search agent needs about 47 calls on the toy map with no
  failures (see below), so 50 would measure who ran out of calls rather than who
  planned better.
- **Retries:** if a lookup fails, the agent may try that same lookup up to 2 more times
  (3 attempts in total) before giving up and treating the result as unknown.
- **Retries count toward the budget.** Every attempt is one call out of the 80.
- **Submission is not a tool call** and is always allowed, even with the budget used up.
  Per D3, it happens once.
- Both agents get exactly the same budget and retry rule.

**Terms:** the agent receives the errand request and builds the plan itself. A *tool
call* is one lookup, such as `get_hours(Pharmacy, Tue)` or `walk_time(Union, QuadCafe)`.
A *retry* is the same lookup sent again after it failed. The *budget* is the total
number of calls, retries included, allowed in one scenario.

**Why:**
- Limits stop an agent from hammering the tools until it gets lucky, and they make tool
  calls a real cost, which is part of the research question.
- Counting retries keeps it fair: an agent that retries a lot pays for it.
- The budget must not be so tight that results measure who ran out of calls instead of
  who planned better. 80 leaves room for retries at 15% and 30% failure rates.

**Why the search agent uses so many calls:** its gathering step looks up the walk time
between *every pair* of relevant places before planning starts. On the toy map with all
four services that is about 4 place searches + 7 hours lookups + 36 walk times ≈ 47
calls, before any failures. This cost comes from the gathering step, not from A* or UCS:
the planner itself makes no tool calls.

**Pilot check:** run both agents on a few scenarios, including 5-errand ones, at 0%
failure, and record calls used. Record the numbers here and in LOG.md. If the busiest
runs approach 80, revisit. Report how often each agent hits the budget in the main
results.

**Affects:**
- `tools.ToolEnv`: `budget=80` already matches; no change.
- `agents/search_agent.py`: `max_retries=2` already matches. The existing TODO (fetch
  walk times lazily, only when the search needs them) is the main lever on tool-call
  cost and is worth doing before the main experiment.
- ReAct agent (not built yet): apply the same retry rule, and tell it the budget in its
  prompt.

## D6. LLM: Claude Sonnet 5.5 — Settled, provisional (2026-11-25)

**Decision:** Use Claude Sonnet 5.5 (API model string `claude-sonnet-5-5`) for every
LLM role: the request parser, the ReAct agent, and the direct-LLM baseline. One model
for everything, with the exact model string, temperature, and run date recorded with
every run.

**How we decided:** one guiding question, checked four ways: *will this model let the
real difference (planning vs. improvising) show up clearly?* In any comparison, the
difference you measure should come from the thing being tested, not from something
else. Each check guards against a different way that goes wrong.

| Check | Question | What it guards against | Sonnet 5.5 |
|---|---|---|---|
| 1. Tool-use reliability | Can it call tools correctly and follow the schema? | A **confound**: if the model fumbles tool calls, that noise swamps the planning effect, and we measure the model's weaknesses instead. | Strong at tool use. |
| 2. Ceiling / floor | Will ReAct land somewhere in the middle, not near 0% or 100%? | A **ceiling or floor effect**: if every agent scores near 100% (or 0%), no difference can show up. A mid-tier model often gives a more informative comparison than the top model. | Capable but not top tier, so it is less likely to be at the ceiling. To be confirmed in the pilot. |
| 3. Cost | Can we afford enough runs? | Low **statistical power**: ~100 scenarios × 3 failure rates × several agents, each up to 80 tool calls, adds up. A cheaper model lets us run more scenarios, which tightens the Wilson intervals and paired-bootstrap results. | Much cheaper than the top model. |
| 4. Reproducibility | Can someone rerun this exactly? | Results nobody can check. A research norm. | Pin the exact model string; record it in every results file. |

**Fairness:** the same model, prompt information, tools, failure conditions, and budget
for every agent (as in the proposal). We compare planning methods, not model families.

**Note on where the model matters:** in the search-guided agent, the LLM only parses the
request; A* itself is ordinary code. So the model mostly affects the ReAct agent's
results, plus parsing for both. The gold-parsed run separates parsing errors from
planning errors.

**Caveat: may change after the pilot.** This choice is locked in for now but will be
revisited after the pilot run, based on how the agents perform. Switch models if:
- ReAct is near the ceiling (close to 100% CSR even at 30% failure), leaving A* no room
  to show an advantage, or near the floor (close to 0%), so the comparison is uninformative;
- ReAct often fails because of tool-call or format mistakes rather than planning;
- the parser makes so many errors that they hide the difference between agents.

If the model changes, it changes **before** the main experiment, for all agents at once,
and every pilot run is redone with the new model. Record the change and the reason here
and in LOG.md.

**Affects:**
- Agent and parser code (not built yet): read the model string from one config value so
  it cannot differ between agents.
- Results files: store model string, temperature, and run date with every run.

---

## D7. Walk-time fetching: up front first, lazy as a switch — Settled (2026-11-25)

**Decision:** Build the search agent in two stages.
1. **Up front (build first):** the agent looks up the walk time between every pair of
   relevant places before planning starts. This is what the draft code does today. It is
   the simple, working baseline.
2. **Lazy (add second, as a switch):** the agent looks up a walk time only when the
   search actually considers that move, and stores it so it is never fetched twice.
   Turned on with a flag (for example `lazy_walks=True`), the same way UCS is the
   `use_heuristic=False` flag on A*.

Place searches and hours stay up front in both modes. Only walk times are lazy.

**Why:**
- Walk-time lookups grow with every *pair* of places (9 places → 36 pairs, 20 → 190),
  so they are most of the tool-call cost. Hours are one lookup per place, so making them
  lazy saves little.
- Many up-front walk times are never used (for example, coffee shop to coffee shop).
- "Lazy" means "only when needed". It finds the same best plan and spends fewer calls.
- In up-front mode, A* and UCS make exactly the same tool calls; they differ only in
  states expanded. In lazy mode, A* considers fewer moves, so it should also make fewer
  tool calls. That turns the heuristic's benefit into a real tool-call saving.
- Building up front first gives a working baseline before adding complexity.

**Extra result this enables:** report tool calls and states expanded for all four
combinations: {up front, lazy} × {A*, UCS}.

**Open sub-question (resolve when building lazy mode):** the heuristic uses walk times
(to the nearest remaining errand and to the destination). In lazy mode those may not be
fetched yet, and the code treats an unknown walk as 0, which keeps the heuristic
admissible but makes it weak, so A* behaves more like UCS. Options: (a) accept the
weaker heuristic, (b) fetch the few walk times the heuristic needs (costs calls), or
(c) use a free lower bound such as straight-line distance divided by walking speed,
which needs place coordinates as static task information. Decide this before running
the lazy-mode results.

**Affects:**
- `agents/search_agent.py`: `_gather` keeps the up-front behavior by default. Lazy mode
  moves walk-time lookups into `Belief.walk_time`, which calls the tool on a cache miss.
  Lookups during search count against the budget, can fail, and use the same retry rule
  (D5).
- `search.astar`: no change needed for up-front mode. Lazy mode must handle the budget
  running out partway through a search.
- Tests: up-front and lazy mode must return the same plan at 0% failure, and lazy mode
  should use no more walk-time calls than up front.
