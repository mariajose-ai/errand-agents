# Project Log

Dated record of what was done, what was decided, and what's next.
Newest entries first. Code changes themselves are in `git log`.

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
