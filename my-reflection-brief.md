# Reflection Brief — Harness Engineering Capstone

**Name:**
**Date:**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):
- OS / Python:
- Approx. API spend:

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The trace `runs/20261005_103013/traces/claim_07_tree_falls_on_car.jsonl` shows `tool_use → end_turn`. In `claims_intake/loop.py`, the loop continues when `response.stop_reason` is `tool_use` and stops when it is `end_turn`. Any unexpected stop reason raises `UnexpectedStopReason`. This run finished in 2 turns.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `tests/test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` checks for using a fixed integer limit such as `range(10)` as the primary stopping mechanism. If the loop used a fixed cap, it could stop before the model completed all required tool calls, or make unnecessary turns when the claim finished earlier. The intended control signal is `stop_reason`, while configuration-sourced budgets can remain safety limits. This keeps the loop adaptive to the model's actual tool-use sequence.

3. **Tool design.** `route_to_adjuster` and `escalate_to_human` are both terminal tools used when the claim reaches a final decision. Their descriptions make the difference clear: `route_to_adjuster` is used when confidence is 0.6 or higher, while `escalate_to_human` handles confidence below 0.6 or cases such as unresolved ambiguity, missing information, or a policy dispute. This reduces the chance that the model chooses the wrong final action. The `_t_lookup_policy` error also returns structured fields such as `is_error`, `error_category`, and `is_retryable`; because the policy-not-found case is non-retryable, the agent knows that repeating the same lookup is not useful and can take another action.

4. **Your numbers.** For `claim_01_kitchen_fire`, the run recorded **3 turns**, **9,494 input tokens**, **566 output tokens**, an estimated cost of **$0.0123**, and **5.9 seconds** elapsed in `summary.md`. In comparison, `claim_03_water_damage` took **5 turns**, cost **$0.0207**, and took **11.8 seconds**. The water-damage claim needed one `request_clarification` round, so the extra interaction added another turn and more accumulated context, making it noticeably more expensive than the kitchen-fire claim. These figures come from the System 1 run summary rather than the README example.

### System 2 — Context strategy

5. **The reduction.** The `budget.json` artifact shows a baseline of **38,708 tokens** and an assembled context of **16,851 tokens**, which is a **56.47% reduction**. The section breakdown is `case_facts` 204, `resolved_refund` 360, `resolved_subscription` 516, and `active` 15,789 tokens. The `active` section is therefore the largest part of the assembled context, accounting for about 94% of the total. It is kept verbatim because `assemble.py` places the active issue immediately before the model's next response, so changing those recent details could cause the next turn to lose important information.

6. **Summarize vs preserve.** The rule in `compressor.py::summarize_segment` is that only segments with a `resolved` status can be compressed; unresolved or active segments are not sent through the summarization step. In the run data, the resolved sections were `resolved_refund` at **360 tokens** and `resolved_subscription` at **516 tokens**. The active section remained at **15,789 tokens** because its detailed conversation is still needed for the model's current reasoning. This separates information that is already settled from information that is still being actively worked on.

7. **Facts block.** In `eval.jsonl`, all six questions passed, including Q6 about the structured status of the payment-method update issue. The full run returned `payment_update_status: in_progress`, matching the expected answer. In `eval_control.jsonl`, where the case-facts block was removed, Q6 failed even though the active issue text was still available; the model said that there was no formal structured status token. The other five questions still passed, showing that the persistent case-facts block was particularly important for retrieving this structured field rather than simply providing information that was missing from the conversation.

### System 3 — Claude Code config

8. **Path-scoped rules.** In `.claude/rules/tests.md`, the frontmatter applies to `**/*.test.tsx` and `**/*.test.ts`. This works better than putting the rule in a directory-level `CLAUDE.md` because test files are spread across different parts of the repository. One path-scoped rule can therefore apply the same testing convention wherever a matching test file appears, without copying the rule into several directories. The evidence also includes `test_ac_02_06_test_file_matches_react_and_tests`, which checks that a test file can match both the React rule and the tests rule.

9. **Forked skill.** The frontmatter in `.claude/skills/deploy-check/SKILL.md` uses `context: fork` and an `allowed-tools` list containing read-only operations such as `Read`, `Grep`, `Glob`, and restricted Git/GitHub commands. Forking keeps the detailed checking work, such as file enumeration and Git output, out of the main conversation and returns the useful result instead. The read-only allowlist also prevents the deploy check from pushing, deploying, or changing files. Without the fork, repeated raw command output would consume the main context; without the allowlist, the skill could potentially gain state-changing capabilities if its configuration were changed.

10. **Scope.** The validator output is `OK` with exit code 0, so the configuration hierarchy passed its validation. At the project level, `./CLAUDE.md` and the files under `.claude/rules/` such as `api.md`, `react.md`, and `tests.md` are shared with the repository and apply to the team. At the user level, the example is `~/.claude/skills/deploy-check-strict/`, which the skill documentation describes as a personal stricter variant. Changes to that user-level skill stay with the individual and are not committed to the project.

### System 4 — Orchestration

11. **Push work down.** The shift run reported `shift C: 0 new defects`, while the warm tier had been seeded with **40 defect rows** from `fixtures/defects.json`. The indexed query is `WarmStore.defects_since()` in `shift_monitor/warm.py`, which uses `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` against the timestamp index. `gather_new_defects` passes the query result through without doing Python-side filtering, which is also covered by `test_gather_new_defects_has_no_python_side_filtering`. This means SQLite filters the history before the data is placed into the model's context, so the model does not need to process the complete warm-tier history on every shift.

12. **Crash recovery.** In `recovery.py`, the staleness threshold is `STALE_RESUME_THRESHOLD_MINUTES = 30`. The `decide()` function chooses `resume` when there are incomplete steps and the last step is no more than 30 minutes old; otherwise it chooses `fresh`. The tests confirm the boundary, with 30 minutes still allowing `resume` and 31 minutes selecting `fresh`. After the threshold, starting fresh with an injected summary is safer because it keeps the useful findings from the earlier run while avoiding reliance on stale partial state that may no longer match the current shift.

13. **Small state.** My `data/hot_state.json` file is **643 bytes**, which is comfortably below the roughly 5 KB budget. This matters because the monitoring process runs once per shift, so the same state is read and updated repeatedly over the system's lifetime. `HotState` also limits the stored recent defect hashes, as checked by `test_hotstate_rejects_more_than_20_hashes`. Without that limit, the state could keep growing with every shift and make later runs slower and more expensive; keeping it small helps keep the per-shift context roughly bounded.

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.**  
→ **Model:** `claims_intake/tools.py` and its `TOOL_SCHEMAS`, including the enum-constrained `classify_claim` schema. These schemas define the actions and inputs exposed to Claude while the model makes the claim-level decisions.  
→ **Harness:** `claims_intake/loop.py`, especially `run()`, which manages the `stop_reason` loop, budget checks, and tool-result feedback around the model.  
→ **Orchestration:** `shift_monitor/recovery.py`, `manifest.py`, and `fork.py`, which handle concerns across separate runs, including recovery, fork/merge behavior, and persistent state. The three layers therefore separate model decisions, control of one agent run, and coordination across multiple runs.

15. **Deterministic vs prompt.** A behavior that is guaranteed in code is the System 4 `HotState` limit, covered by `test_hotstate_rejects_more_than_20_hashes` and the atomic-write behavior in the state implementation. System 3 provides another example: the `deploy-check` `allowed-tools` list does not include `Write`, so the skill cannot modify files through that tool configuration. In contrast, the System 1 confidence threshold of **0.6** is communicated through the tool descriptions for `classify_claim`, `route_to_adjuster`, and `escalate_to_human`, rather than being enforced by `loop.py`. I would use code enforcement for safety or integrity requirements that must never be violated, and prompt guidance for decisions that require judgment about individual cases.

16. **Context, two faces.** System 2 manages context within a single conversation: the resolved sections were reduced to **360** and **516 tokens**, `case_facts` remained at **204 tokens**, and the active section stayed at **15,789 tokens**. Overall, the assembled context fell from **38,708 to 16,851 tokens**, a **56.47% reduction**, according to `budget.json`. System 4 works across separate shift invocations instead: `defects_since()` filters the warm history in SQL before the data reaches the model, while only a small persistent state is carried between runs; my `hot_state.json` was **643 bytes**. The common principle is to avoid making the model process unnecessary history, but System 2 does it through summarization and context placement while System 4 uses source-side filtering and compact persistent state.

17. **Reliability you can't see in one run.** The recovery boundary tests in `test_recovery_decide_truth_table` check both sides of the 30-minute cutoff: **30 minutes resumes**, while **31 minutes starts fresh**. `test_hotstate_rejects_more_than_20_hashes` also checks that persistent state does not exceed the intended hash limit. A normal successful run would not necessarily exercise either boundary, so it could pass while these edge cases were still broken. These tests matter before shipping because failures at recovery or state-size boundaries may only appear after unusual production conditions, when fixing them is more costly.

18. **Blast radius.** I would use System 3's `deploy-check` as the example because its capabilities are deliberately narrow. Its `allowed-tools` list contains read-only operations such as `Read`, `Grep`, `Glob`, selected Git commands, and `gh pr view`/`checks`, so the skill cannot directly push, deploy, modify files, or run migrations. The main remaining risk is an incorrect review result being trusted by a human and allowing a bad change to proceed. The configuration itself is the key enforcement point and can be tightened or the skill can be removed from `.claude/skills/` if the capability needs to be disabled.

19. **What broke.** My first environment issue was a dependency mismatch in System 1: `anthropic==0.39.0` failed with `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'` because the environment had a newer `httpx` version. I fixed that environment by pinning `httpx==0.27.2` and `httpcore==1.0.5`. A separate compatibility issue appeared in System 2 when `anthropic==0.39.0` lacked the `messages.count_tokens` method required by `retail_context/tokens.py`, producing an `AttributeError`. I restored System 2 with `anthropic==0.69.0`, which showed that the dependency versions needed to be managed separately for each system.

20. **What you'd change.** I would make dependency management more explicit by pinning the exact `httpx` and `httpcore` versions, or using a lockfile, in each system's own `pyproject.toml`. The failure documented in System 1, where the `proxies` argument was incompatible with the installed `httpx` version, showed that leaving transitive dependencies open can create avoidable runtime failures. System 2 also required a different `anthropic` version because its token-counting code depended on `messages.count_tokens`. Based on these observations, I would keep each system's dependency set isolated and reproducible rather than assuming that one shared set of SDK versions will work everywhere.

