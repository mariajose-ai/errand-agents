> **Note:** This idea was first drafted as a CS221 project proposal. It is now an
> independent project; the plan below is the original reference. Changes to the plan
> are recorded in [LOG.md](LOG.md) and decisions.md.

# Search-Guided Agents for Multi-Errand Planning Under Tool Failures

*CS221 Autumn 2026 Project Proposal Draft*
*Research by Maria Jose Rios*

**Goal:** We're testing whether an AI assistant that plans with a search algorithm can still build a workable route for several campus errands when its tools (like opening-hours lookups) fail, compared with one that just calls the tools step by step.

## Problem Statement and Task Definition

Tool-using agents can gather current information, but a tool may fail or return an incomplete response. We ask whether explicit search and replanning help an agent complete a multi-step task when tool calls fail.

- **Research question:** Under controlled tool-call failures, does an A* search-guided tool agent produce more feasible errand plans than a sequential ReAct-style tool agent, and what is the added cost in tool calls?
- **Hypotheses:** (H1) The search-guided agent will have higher constraint-satisfaction success than the sequential agent when tool calls fail. (H2) Its advantage will increase as the failure rate rises, though it may require more tool calls. The project uses an offline simulator and one fixed simulated day. It does not depend on live campus services.

### Input and Output

- **Input:** A natural-language errand request, a start location and time, a deadline, and a fixed destination. Example: "It's Tuesday at 1:10 PM and I'm at Tresidder. Print a poster and get coffee, then arrive at Gates by 2:00 PM."
- **Output:** An ordered itinerary with arrival times, or INFEASIBLE with the violated constraint. For example: Lathrop (print, 1:22 PM) → Coupa (coffee, 1:38 PM) → Gates (arrive, 1:51 PM). The evaluator checks that each errand is completed at a location offering that service, each stop is open at the visit time, the route uses available paths, and the final destination is reached by the deadline.

The local API has five functions: `find_places(service)`, `get_hours(place, day)`, `walk_time(from, to)`, `services(place)`, and `submit_plan(plan)`. Each simulated tool response may fail or omit a field under the assigned perturbation condition.

## Dataset and Evaluation

We will use a frozen OpenStreetMap extract for Stanford and nearby Palo Alto as the source of pedestrian paths and public amenities. We will filter it to a compact, connected subgraph and a few well-represented venue types, such as cafes, libraries, and pharmacies. OSM amenity tags and pedestrian paths provide a useful starting point, but opening hours and service details may be absent or inconsistent. To keep the task checkable, we will include only venues whose hours and relevant services can be verified from a public source, and manually curate those fields for at most 25–30 venues. We will record the OSM snapshot date, source links, and curation date. A set of roughly 100 scenarios will be generated from templates and stratified by number of errands (one to five), deadline tightness, and feasibility; about 20% will be infeasible. We will hand-check a sample from every scenario category before testing. Tool failures will be injected by a local wrapper over the frozen data, so experiments remain offline and repeatable.

The primary metric is constraint-satisfaction rate (CSR): the fraction of plans meeting all task constraints. Secondary metrics are optimality gap in travel minutes relative to the best feasible plan, infeasibility-detection F1, and tool calls per completed plan. The primary experiment compares agents on the same scenarios with tool-call failure rates of 0%, 15%, and 30%. We will report paired success-rate differences and confidence intervals, plus success by number of errands and failure rate. Error analysis will categorize failures as request-parsing, planning, tool-recovery, or final-validation errors, with representative traces. We will not require an advanced statistical model; a paired bootstrap or Wilson intervals are sufficient for this project scale.

## Baselines and Oracle

- **Direct LLM baseline:** The model receives the initial simulator information and request, then produces a plan without tools. We will report it on clean cases as a reference for direct answering; it cannot observe later tool failures.
- **Sequential tool-agent baseline:** A ReAct-style agent calls the same tools in sequence, observes results, and decides the next action, without explicit search over alternative plans.
- **Main approach:** The same LLM parses the request, then an explicit A* planner searches for a feasible itinerary and replans after tool errors.
- **Oracle:** An exact solver with full visibility of the realized simulator state returns the minimum-travel-time feasible itinerary, or correctly identifies infeasibility. It is an upper bound on task quality, not an agent comparison. An optional gold-parsed-request run can diagnose whether errors came from parsing or planning.

For a fair main comparison, the sequential and search-guided agents will use the same model version, prompt information, tools, failure conditions, and maximum tool-call budget. We will use one model rather than comparing model families.

## Methodology

The LLM converts each request into a fixed schema: start, start time, final destination, deadline, and errands. A validator checks the parsed fields. A search state records current location, current time, completed errands, and remaining tool-call budget. Actions visit candidate locations and complete available errands; travel and service time advance the clock. A* prioritizes states by elapsed time plus an admissible lower bound: the maximum of the shortest travel time from the current location to any remaining errand and the shortest travel time to the final destination. This lower bound ignores extra detours and service time, so it cannot overestimate the remaining cost. The planner accepts only plans that pass final simulator validation.

Tool failures are transient errors or missing fields at the selected rate. Both tool-using systems receive the same failure injection and retry limit. The main agent retries a failed lookup within budget, updates its search state from successful observations, and replans when a candidate itinerary is invalid. It must treat an exhausted lookup as unknown rather than silently assuming the place is closed. This separates recovery behavior from changes in the task data.

## Related Work

ReAct interleaves language-model reasoning with actions and observations, motivating our sequential tool-agent baseline (Yao et al., 2023). LLM+P combines an LLM with a classical planner, a useful point of comparison for our division between language parsing and symbolic search (Liu et al., 2023). HELM's standardized, multi-metric evaluation motivates matched conditions, robustness and efficiency measures, and preservation of prompts and raw outputs (Liang et al., 2023). Our project is a small controlled study of recovery from simulated tool failures, not a reproduction of these works.

## Challenges, Scope, and Learning Value

The main implementation challenge is coordinating A* with incomplete or failed tool observations while cleaning a real but imperfect map extract. We will limit the network to a compact connected area, use at most five errands, one model, and three failure rates. Attribute drift, multiple model families, a larger region, and learned failure policies are stretch goals and are excluded from the core deliverable. If the OSM extract has too few venues with verifiable hours, we will reduce the venue types or use a small public open-data directory for those attributes rather than inventing map locations. We will attribute OpenStreetMap and follow its data-use requirements.

This project practices dataset construction, controlled experiments, paired statistical comparisons, error analysis, and communicating limitations—skills relevant to MIDS. A limitation is that synthetic errors may not match real tool reliability; conclusions will be stated as applying to this simulator. The data contains no personal information. A future real deployment could give users incorrect hours or routes, so it would need verified current data and uncertainty disclosures.

## References

1. Stanford CS221, Autumn 2026, [Final Project Guidelines: Proposal](https://stanford-cs221.github.io/autumn2026/project.html#p-proposal).
2. Yao et al. (2023). ["ReAct: Synergizing Reasoning and Acting in Language Models."](https://arxiv.org/abs/2210.03629) ICLR.
3. Liu et al. (2023). "LLM+P: Empowering Large Language Models with Optimal Planning Proficiency." [arXiv:2304.11477](https://arxiv.org/abs/2304.11477).
4. Liang et al. (2023). ["Holistic Evaluation of Language Models."](https://arxiv.org/abs/2211.09110) TMLR.
5. Percy Liang, [Research page](https://cs.stanford.edu/~pliang/).
6. OpenStreetMap Wiki, [Pedestrian navigation guidelines](https://wiki.openstreetmap.org/wiki/Guidelines_for_pedestrian_navigation) and [opening-hours tag](https://wiki.openstreetmap.org/wiki/Key:opening_hours).
