# Project Log

Dated record of what was done, what was decided, and what's next.
Newest entries first. Code changes themselves are in `git log`.

## 2026-10-17

**Done**
- Created the project folder and its own git repo, separate from the home-folder repo.
- Commit 1: initial project shell (README, .gitignore, pyproject, smoke test).
- Added the original proposal and this log.

**Decided**
- Independent project, no longer for CS221.
- Small, reviewed commits, one logical change each; Claude credited as co-author.
- Build the parts that don't need an LLM first (simulator, evaluator, oracle) on a toy map;
  real OSM data comes later.

**Open questions**
- The seven design decisions (objective, waiting, submit feedback, failure kinds,
  limits, LLM choice, GitHub visibility). They go in decisions.md once settled.

**Next**
- Commit 3: data README with OSM attribution.
- Create the GitHub repo and push.
