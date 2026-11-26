# Project Log

Dated record of what was done, what was decided, and what's next.
Newest entries first. Code changes themselves are in `git log`.

## 2026-11-25

**Done**
- Settled the open design decisions and recorded them in docs/decisions.md (D1–D7).
- Linked decisions.md from the proposal note.

**Decided**
- D1: Waiting outside a closed place is allowed.
- D2: The objective is earliest finish time. Changed from the proposal's "minimum travel time".
- D3: Submit feedback is off in the main experiment; the first submission is final.
  Changed from the draft code, which kept the last submission and returned violations.
- D4: Keep the 50/50 failure split, tagged as three kinds: error, missing_field, silent_drop.
- D5: 80 tool calls per scenario, 2 retries; retries count toward the budget.
- D6: Claude Sonnet 5.5 for every LLM role (provisional; revisit after the pilot).
- D7: Fetch walk times up front first; add lazy fetching later as a switch.

**Learned**
- Choosing the model is about making the effect measurable, not picking the strongest
  model: a model near 100% (or 0%) leaves no room for a difference to show up.

**Open questions**
- D7 sub-question: what lower bound the heuristic uses in lazy mode when walk times
  aren't fetched yet.

**Next**
- Review the draft simulator code one file at a time, applying D3 and D4.
- Pilot run to check the tool-call budget (D5) and the model choice (D6).

## 2026-10-17

**Done**
- Created the project folder and its own git repo, separate from the home-folder repo.
- Commit 1: initial project shell (README, .gitignore, pyproject, smoke test).
- Added the original proposal and this log.
- Added data/README.md: planned data layout, sources, and OSM attribution/ODbL notes.
- Added a GitHub Actions workflow that runs the tests on every push and pull request.
- Published the repo at https://github.com/mariajose-ai/errand-agents (public); CI passes.
- Replaced the personal email in all commits with the GitHub noreply address before going
  public. The first (private) GitHub repo was deleted and recreated so no old commits
  remain online.

**Decided**
- Independent project, no longer for CS221.
- Small, reviewed commits, one logical change each; Claude credited as co-author.
- Build the parts that don't need an LLM first (simulator, evaluator, oracle) on a toy map;
  real OSM data comes later.
- Repo is public. Commits use the GitHub noreply email (set in this repo's git config).

**Open questions**
- The remaining six design decisions (objective, waiting, submit feedback, failure kinds,
  limits, LLM choice). They go in decisions.md once settled.

**Next**
- Settle the six design decisions and record them in docs/decisions.md.
- Review the draft simulator code one file at a time; commit each file with its tests.
