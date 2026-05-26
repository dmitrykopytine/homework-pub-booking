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

I never saw `verify_dataflow` fail during a live `make ex5` run. Session
`sess_08a7ead53e4b` completed cleanly: the flyer reads as a coherent
Haymarket Tap booking (cloudy, 12°C, £540 total, no deposit), and a human
skimming `workspace/flyer.html` would likely approve it.

**Plausible catch a human would miss.** After that same run, I took
`workspace/flyer.html` and changed one number in the total —
`<dd data-testid="total">£540</dd>` → `£9999`. I re-ran
`verify_dataflow` with `_TOOL_CALL_LOG` still populated from the
scenario. A reviewer who checks venue name, weather tone, and "does the
HTML look fine?" may never cross-check every `£` against `trace.jsonl`.
Meanwhile, the check failed: `extract_money_facts` pulls `£9999`,
`fact_appears_in_log` scans all logged tool `output` and `arguments`, and
`9999` appears nowhere (`calculate_cost` logged `total_gbp: 556` in trace
line 5, not 9999).

**What the check does not catch (and my run shows why).** In
`sess_08a7ead53e4b`, `calculate_cost` returned **£556 / deposit £111**
(`trace.jsonl` line 5), but the scripted `generate_flyer` call passed
`total_gbp: 540, deposit_required_gbp: 0` (line 6), and the flyer matches
those wrong inputs. `verify_dataflow` mistakenly passes because
`fact_appears_in_log` treats **`generate_flyer` arguments as valid
sources** (not just upstream tool outputs) — so invented or, in this
case, manually written test numbers (`starter/edinburgh_research/run.py:91`)
copied into `event_details` are self-verifying. Excluding `generate_flyer`
from the log scan (or requiring cost facts to trace only to
`calculate_cost` output) would close that gap with minimal change. The
£9999 plant catches *post-hoc HTML tampering*; it does not catch *LLM
mis-wiring between tools* unless the wrong value never appears anywhere
in the log.

### Citation

- `sessions/examples/ex5-edinburgh-research/sess_08a7ead53e4b/logs/trace.jsonl` — line 5 (`£556` / `£111`) vs line 6 (`540` / `0` in `generate_flyer` args)
- `sessions/examples/ex5-edinburgh-research/sess_08a7ead53e4b/workspace/flyer.html` — rendered `£540` / `£0`
- `starter/edinburgh_research/integrity.py` — `verify_dataflow`, `fact_appears_in_log` (scans all tool `output` + `arguments`)
- `starter/edinburgh_research/tools.py` — `generate_flyer` logs `event_details` in arguments, not HTML body

---

## Q3 — First production failure

### Your answer

The first production failure I'd expect is a **false-green executor
ticket while the booking never completes**. In my `make ex5-real` run,
session `sess_0b8dccbc140a`, the live executor (Qwen3-32B) ended with
`Loop half outcome: handoff_to_structured` — not `complete`. Both
tickets show **success** (`tk_629488b6` planner, `tk_b59c1f35`
executor), yet `make ex5-real` exited with "No flyer written to
workspace/." and `generate_flyer` was never called.

The trace explains why: the LLM ignored the required Haymarket search
and called `venue_search` twice with wrong parameters — `Edinburgh`,
party 4, then `Edinburgh City Center`, party 10 (`trace.jsonl` lines
3–4, both `0 result(s)`). It then invoked `handoff_to_structured`
(line 5). Ex5 is loop-only — no `HandoffBridge` — so
`ipc/handoff_to_structured.json` was written but never consumed.
`session.json` still shows **`state: "executing"`** and **`result:
null`**; `run.py` returned exit 1 but never called
`session.mark_failed()`.

The **ticket state machine** surfaces this: executor ticket
`tk_b59c1f35` closes as **`success`** with manifest
`handoff_requested: true` and summary "Handoff to structured half
requested" — because from the executor's view, subgoal `sg_1` finished
normally (the handoff tool returned `success: true`). That contradicts
the actual outcome: no `generate_flyer`, no `workspace/flyer.html`, and
`session.json` still `executing` with `result: null`. The ticket record
says the executor did its job; the booking was never produced.

### Citation

- `sessions/examples/ex5-edinburgh-research/sess_0b8dccbc140a/logs/trace.jsonl` — lines 3–5 (failed searches, handoff)
- `sessions/examples/ex5-edinburgh-research/sess_0b8dccbc140a/logs/tickets/tk_b59c1f35/manifest.json` — ticket `success`, `handoff_requested: true`
- `sessions/examples/ex5-edinburgh-research/sess_0b8dccbc140a/logs/tickets/tk_b59c1f35/summary.md` — "Handoff to structured half requested"
- `sessions/examples/ex5-edinburgh-research/sess_0b8dccbc140a/session.json` — still `executing`, `result: null`
- `sessions/examples/ex5-edinburgh-research/sess_0b8dccbc140a/ipc/handoff_to_structured.json` — orphan handoff, no consumer
- `starter/edinburgh_research/run.py` — detects missing flyer (exit 1) but does not update session state
