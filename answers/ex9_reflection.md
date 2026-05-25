# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session `sess_f02826dbf06b` (`make ex7-real`), the planner never
assigned a subgoal to the structured half. Both planner tickets show
`assigned_half: "loop"`.

Round 1 (`tk_e6bc4eea/raw_output.json`): subgoal `sg_1` is
`"find venue near haymarket for 12"` with `"assigned_half": "loop"`.
Round 2 (`tk_641bb2a4/raw_output.json`): after Rasa rejected party 12,
the planner replans `"retry with larger venue after rejection"` — still
`"assigned_half": "loop"`.

Structured half involvement came from the **executor**, not the planner.
Executor turn 1 logged `venue_search(Haymarket, party=12): 0 result(s)` in
`trace.jsonl` line 4 (Haymarket Tap has only 8 evening seats in
`venues.json`, below party 12). Despite zero matches, turn 2 still
handed off Haymarket Tap — because offline Ex7 **hard-codes** that second
executor response in `starter/handoff_bridge/run.py`
(`_build_fake_client_two_rounds`, the `ScriptedResponse` with
`handoff_to_structured` at ~lines 69–87). The fake client does not read
search output; it always plays planner -> search -> handoff.

In `tk_c7703d03/raw_output.json`, that scripted call passes reason
*"passing to structured half for confirmation under policy rules"* and
booking `data` (`Haymarket Tap`, party `"12"`). `trace.jsonl` line 5
records it as `executor.tool_called`; line 6 is `session.state_changed`
loop -> structured once `LoopHalf` sees `handoff_requested: true`.

What routed work to Rasa was not planner prose or model reasoning — it was
the **next hard-coded `ScriptedResponse`** in `FakeLLMClient`, which
always emits a `handoff_to_structured` tool call with fixed `reason`,
`context`, and `data` fields (including the string *"under policy
rules"*). That wording appears in logs but was never interpreted; it is
literal JSON in `run.py`. The bridge only checks
`next_action=handoff_to_structured` and forwards `handoff.data` to Rasa.
Even in `make ex7-real`, the loop half stays on `model="fake"`; only
Rasa is live. Booking rules actually applied on round 1 in
`ActionValidateBooking` (`party_too_large`), not in planner or executor
text.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_f02826dbf06b/logs/tickets/tk_e6bc4eea/raw_output.json` — round 1 planner (`assigned_half: "loop"`)
- `sessions/examples/ex7-handoff-bridge/sess_f02826dbf06b/logs/tickets/tk_c7703d03/raw_output.json` — round 1 executor handoff
- `sessions/examples/ex7-handoff-bridge/sess_f02826dbf06b/logs/tickets/tk_641bb2a4/raw_output.json` — round 2 planner (still loop)
- `sessions/examples/ex7-handoff-bridge/sess_f02826dbf06b/logs/trace.jsonl` — line 4 (`0 result(s)`), lines 5–6 (handoff)
- `starter/handoff_bridge/run.py` — `_build_fake_client_two_rounds()` scripted handoff

---

## Q2 — Dataflow integrity catch

### Your answer

During Ex5 development my integrity check caught a subtle fabrication
that manual review missed. In session sess_de44a1b8eb12 the flyer
claimed "Total: £560" and "Deposit: £112" — plausible numbers that
followed the deposit formula in catering.json. I skimmed and moved on.

verify_dataflow returned ok=False with unverified_facts=['£560','£112'].
The trace showed calculate_cost returned total_gbp=540, deposit=0. The
real total was £540 under the £300 deposit threshold. The LLM had
written "£560" plausibly — close enough that a human reviewer wouldn't
notice without cross-referencing.

The check caught it because it compared against ground truth in
_TOOL_CALL_LOG, not against "does this look reasonable." The lesson
generalises: if the validator would pass a human skim, plant a
deliberately-weird value like £9999 and confirm it's caught.

### Citation

- sessions/sess_de44a1b8eb12/workspace/flyer.md:12
- sessions/sess_de44a1b8eb12/logs/trace.jsonl:15

---

## Q3 — Removing one framework primitive

### Your answer

I'd keep session directories (Decision 1) as the last thing standing
and rebuild everything else if forced. The forward-only state machine
(Decision 2) is important but fragile without directories. Tickets
(Decision 3) I could rebuild as .jsonl files inside the session.
Atomic-rename IPC (Decision 5) is replaceable by directory polling.

Session directories are the irreplaceable piece. Losing them:
cross-tenant data leaks, reconstructing per-run state from logs,
"how did this session end up this way" becomes SQL archaeology
instead of cat. The slides compare it to git commits being the
foundation — you can rebuild merge, diff, blame from commits but
not commits from the rest. Session directories are commits.

### Citation

- sessions/sess_de44a1b8eb12/ — the directory itself
- sessions/sess_a382a2149fc1/logs/trace.jsonl
