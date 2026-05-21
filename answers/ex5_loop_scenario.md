# Ex5 — Edinburgh research loop scenario

## Your answer

Implemented 4 required tool calls: `venue_search`, `get_weather`,
`calculate_cost`, `generate_flyer`.

Offline run `make ex5` completed session **sess_7c04c8ef85c4**
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

- `file:///Users/dm9/Library/Application%20Support/sovereign-agent/examples/ex5-edinburgh-research/sess_7c04c8ef85c4/logs/trace.jsonl` - tool call order
- `file:///Users/dm9/Library/Application%20Support/sovereign-agent/examples/ex5-edinburgh-research/sess_7c04c8ef85c4/workspace/flyer.html` - Haymarket Tap flyer
- starter/edinburgh_research/tools.py - Tools