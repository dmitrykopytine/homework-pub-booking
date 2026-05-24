# Ex5 — Edinburgh research loop scenario

## Your answer

Implemented 4 required tool calls: `venue_search`, `get_weather`,
`calculate_cost`, `generate_flyer`.

Offline run `make ex5` completed session **sess_08a7ead53e4b**
(FakeLLMClient). The planner issued two subgoals (`sg_1` research,
`sg_2` flyer). Final answer: "Booking researched; flyer at
workspace/flyer.html".

**Trace tool order**: `venue_search` → `get_weather` → `calculate_cost`
→ `generate_flyer` → `complete_task`.

**Fabrication test**: I edited `workspace/flyer.html` in the latest
session (changed the total from £540 to £9999) and re-ran
`verify_dataflow` on the file directly. The check failed as expected -
`dataflow FAIL: 4 unverified fact(s): ['£9999', '£0', '12',
'cloudy']`.

## Citations

- `sessions/examples/ex5-edinburgh-research/sess_08a7ead53e4b/logs/trace.jsonl` - tool call order
- `sessions/examples/ex5-edinburgh-research/sess_08a7ead53e4b/workspace/flyer.html` - Haymarket Tap flyer
- starter/edinburgh_research/tools.py - Tools