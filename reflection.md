# Reflection Brief — Harness Engineering Capstone

**Name:** Aditya Kanhere  
**Date:** 2026-09-27

## Environment

- **Model(s):** System 1 used `claude-haiku-4-5-20251001`. System 2 used the Anthropic token-counting endpoint. System 4 was verified using the recorded response fixture.
- **OS / Python:** Linux / Python 3.13.0
- **Approx. API spend:** System 1 recorded an estimated total cost of **$0.1312 USD** for its captured run. System 4 used a recorded response, so that verification run did not require a live Claude response.

---

## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.**  
→ In my captured trace, the `stop_reason` sequence was **`tool_use → tool_use → end_turn`**, recorded in `evidences/system_1_claims_loop/trace.jsonl`. The loop control is implemented in the claims-intake loop code in `loop.py`. A `tool_use` result causes the loop to execute the requested tools and continue to another model turn, while `end_turn` terminates the loop. The tests also verify that an unexpected stop reason raises instead of silently retrying (`tests/test_loop.py` and `tests/test_antipatterns.py`).

**2. Anti-pattern.**  
→ One anti-pattern checked by `tests/test_antipatterns.py` is failing to use `stop_reason` as the loop-breaking signal. If the loop ignored `stop_reason`, an unexpected termination reason could cause silent retries or an uncontrolled loop instead of stopping deterministically. The captured trace demonstrates why the distinction matters: the agent continued after `tool_use` but stopped after `end_turn`. This behavior is also covered by the System 1 loop tests, with **29 tests passed** (`evidences/system_1_claims_loop/tests.txt`).

**3. Tool design.**  
→ The captured trace shows the use of `lookup_policy` and `record_claim_fact` as structured tools. Their inputs are explicit: policy lookup uses a `policy_id`, while fact recording uses a named `field` and `value`, so the model has structured arguments rather than having to encode actions in an arbitrary text string. A structured tool error can identify which tool input failed, allowing the agent to correct or retry the specific operation instead of interpreting a generic error message. This makes tool routing and recovery more deterministic.

**4. Your numbers.**  
→ For `claim_01_kitchen_fire`, my run recorded **3 turns** and an estimated cost of **$0.0117 USD** in `evidences/system_1_claims_loop/run_summary.md`. The trace contains two tool-use turns followed by an `end_turn`, matching the three-turn total. The result is specific to this fixture and the actual model/tool path rather than being a fixed number of turns. The complete run processed **8 fixtures** with a recorded estimated total cost of **$0.1312 USD**.

---

### System 2 — Context strategy

**5. The reduction.**  
→ `evidences/system_2_context_strategy/budget.json` records a baseline of **38,708 tokens**, an assembled context of **16,793 tokens**, and a **56.62% reduction**. The `active` section dominates the assembled context at **15,789 tokens**, compared with 204 tokens for `case_facts`, 387 for `resolved_refund`, and 431 for `resolved_subscription`. The active section is kept essentially intact because it represents the current working context and losing exact active information could change the answer to the current support request.

**6. Summarize vs preserve.**  
→ The context strategy summarizes resolved or older conversation sections while preserving the information needed for the current active case. The artifact shows `resolved_refund` at **387 tokens** and `resolved_subscription` at **431 tokens**, while the active section remains **15,789 tokens**. This demonstrates the rule of compressing resolved history but retaining detailed current context. The resulting assembled context is **16,793 tokens**, down from the **38,708-token baseline**.

**7. Facts block.**  
→ The full evaluation in `evidences/system_2_context_strategy/eval.jsonl` passed all six evaluation cases, including the case requiring the exact `in_progress` status. In `eval_control.jsonl`, where case facts were stripped, that corresponding evaluation failed. Therefore, the regression shows that the case-facts block contains information required to recover an exact current-case answer. The evidence supports preserving important structured facts instead of relying only on compressed conversation history.

---

### System 3 — Claude Code config

**8. Path-scoped rules.**  
→ In `evidences/system_3_claude_config/.claude/rules/api.md`, the frontmatter contains:

```yaml
---
description: Conventions for Node.js API handlers
paths:
  - "src/api/**"
---
```

This is better for a cross-cutting convention because the rule activates based on matching file paths rather than requiring the rule to live inside one particular directory's `CLAUDE.md`. The same configuration also demonstrates rules for React files and test files. This keeps conventions targeted to the files where they apply.

**9. Forked skill.**  
→ The deploy-check skill contains `context: fork` and an `allowed-tools` declaration in `evidences/system_3_claude_config/.claude/skills/deploy-check/SKILL.md`. The forked context separates the validation work from the main working conversation, while the restricted tool set prevents the check from becoming an unrestricted editing workflow. This is appropriate for a deployment check because validation should inspect the repository without changing the implementation. Without these boundaries, a validation task could have a larger effect on the main working context or repository.

**10. Scope.**  
→ The validator output in `evidences/system_3_claude_config/validator.txt` is **`OK`**, and the configuration tests recorded **35 passed** in `evidences/system_3_claude_config/tests.txt`. The project-level example is `./CLAUDE.md`, which is shared as part of the project configuration. A user-level example documented in that configuration is `~/.claude/CLAUDE.md` or `~/.claude/commands/`, which is personal rather than project-shared. The configuration therefore distinguishes project conventions from personal Claude Code configuration.

---

### System 4 — Orchestration

**11. Push work down.**  
→ The warm-tier database contained **40 total defects**, while the recorded Shift C pipeline run returned **0 new defects**, as shown by the run evidence in `evidences/system_4_orchestration/shift_run.md`. The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, with the timestamp index defined as `idx_defects_ts`. The pipeline queries only defects newer than the shift's `since` timestamp and applies a limit before constructing the model input. Therefore, the model does not receive the complete 40-record history; the database and pipeline perform the filtering first.

**12. Crash recovery.**  
→ `recovery.py` uses a **30-minute** staleness threshold. A non-empty incomplete manifest whose last step is no more than 30 minutes old can resume; an older partial state is treated as stale and starts fresh. The crash-recovery tests explicitly cover 29, 30, and 31 minutes as well as empty manifests, and these tests are included in the **33 passed** System 4 test suite. A fresh start with an injected summary can be more reliable because it avoids carrying forward stale or partially corrupted execution state while still preserving the important synthesized context.

**13. Small state.**  
→ My captured `data/hot_state.json` is **643 bytes**, recorded in `evidences/system_4_orchestration/hot_state_size.txt`. The system is intended to run once per shift indefinitely, so allowing hot state to grow without a bound would eventually increase prompt size, latency, and cost. Keeping the hot state small forces older detailed history to remain in the warm tier instead of continuously accumulating in the model-facing state. This makes the per-shift execution budget more predictable.

---

## Part 2 — Synthesis

**14. Three layers.**  
→ **Model:** System 1's `evidences/system_1_claims_loop/trace.jsonl` shows Claude producing structured tool calls and an `end_turn`, so the model provides reasoning and tool-use decisions.  
→ **Harness:** System 3's `CLAUDE.md`, `.claude/rules/`, commands, and skills define the development constraints and workflows around the model.  
→ **Orchestration:** System 4's `pipeline.py`, `warm.py`, `recovery.py`, and `hot_state.json` control state, history, recovery, and what information reaches the model. Together these artifacts show that reliable behavior is not provided by the model alone; it comes from the model operating inside explicit harness and orchestration controls.

**15. Deterministic vs prompt.**  
→ A deterministic behavior is the **30-minute resume/fresh threshold** enforced in `recovery.py`, which is also covered by the crash-recovery tests. A prompt-guided behavior is the model's interpretation and synthesis of the shift information, such as the recorded Shift C summary in `evidences/system_4_orchestration/shift_run.md`. Deterministic code is appropriate for hard state, safety, and lifecycle invariants where the same input must produce predictable control behavior. Prompt guidance is appropriate for semantic interpretation and summarization where the task requires language-model reasoning.

**16. Context, two faces.**  
→ System 2 manages context **within a session**, reducing **38,708 tokens to 16,793**, a **56.62% reduction**, as shown in `budget.json`. System 4 manages context **across shifts**, keeping a **643-byte hot state** while storing historical defects in the warm SQLite tier and filtering them by timestamp before model invocation. Both systems apply the same principle: keep the model-facing context focused on information that is currently relevant while moving older or less relevant information outside the active context. The mechanism differs because System 2 compresses conversation history, while System 4 separates hot state from persistent warm history.

**17. Reliability you can't see in one run.**  
→ The crash-recovery tests guarantee behavior that a single successful run would not reveal: they verify resume versus fresh decisions at **29, 30, and 31 minutes**, plus empty and stale manifests. This matters because a normal successful run does not demonstrate what happens after an interrupted or delayed execution. The tests therefore validate lifecycle behavior before shipping rather than relying only on the happy path. The System 4 test evidence records **33 passed**.

**18. Blast radius.**  
→ In System 4, a failure could affect the generated shift summary and the hot/scratchpad state for a monitoring shift, but the architecture limits the amount of historical information passed directly to the model. The enforcement points include the warm-tier timestamp query, the small hot-state file, and the recovery logic in `recovery.py`. The operational control point is the `run-shift` invocation; stopping or disabling that scheduled invocation prevents another shift run from executing. The captured artifacts do not document a separate emergency kill-switch, so I would not claim that one exists.

---

## Part 3 — Honest assessment

**19. What broke.**  
→ One first attempt that failed in my environment was a System 1 rerun caused by an invalid or missing API key, which produced an empty trace file. I did not use that empty artifact as evidence. Instead, I used the successful run `20260925_115041`, whose summary contains eight processed fixtures and a real three-turn trace for `claim_01_kitchen_fire`. I also verified the final evidence structure before preparing the submission.

**20. What you'd change.**  
→ I would make evidence validation a first-class step immediately after every system run. My experience showed that a technically successful or partially successful run is not enough if the required artifacts are missing or a trace is zero bytes. A post-run script could check every required evidence path, reject empty files, and verify expected test counts before allowing the final ZIP to be created. This would reduce the risk of another submission being marked unable to review because the evidence package is incomplete.