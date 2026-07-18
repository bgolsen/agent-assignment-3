## Grade: 98 / 100

**Assignment:** E-Commerce Supply Chain Manager (n8n)  
**Attempt:** 1 of 2  ·  **Graded:** 2026-07-18  ·  Commit `79b32a5`

### Score breakdown
| Criterion | Max | Earned | Notes |
|-----------|-----|--------|-------|
| mp_1 | 8 | 7 | Output JSON schema defines next_subgoal constrained to the four literal types, and the Route on Subgoal Switch routes on exactly that field. Full band withheld only because the required demo video is not recorded (analysis.md:56); analysis.md 4.4 reports 3 branch runs but no verifiable demo. (`workflows/supply-chain-manager-starter.json (Master Planner Agent node, system prompt)`) |
| mp_2 | 10 | 10 | Two worked HTN-style decomposition examples (A: forecast->inventory->supplier->logistics with dependency rationale; B: supplier-only, teaching not to over-decompose). Reflective analysis.md present (1713 words) and discusses planning behavior. (`workflows/supply-chain-manager-starter.json (Master Planner Agent node: Example A and Example B) + analysis.md 4.5`) |
| df_1 | 6 | 6 | movingAverage() correctly averages the trailing 3 of the sorted series, falls back to the mean of available months when <3, and returns 0 on empty. (`custom-nodes/demand-forecast.js:68-73`) |
| df_2 | 8 | 8 | seasonalIndex() computes month-of-year mean / overall mean with correct guards (empty series and zero overall mean default to 1.0); combined as ma x si at line 121. (`custom-nodes/demand-forecast.js:93-106`) |
| df_3 | 4 | 4 | Prompt takes the numeric forecast row in, bounds revisions to named contextual signals with delta-magnitude tiers, defers (delta_pct 0) when no signal, and emits a strict schema-conformant JSON object. (`workflows/supply-chain-manager-starter.json (Forecast Context Adjuster node)`) |
| eoq_1 | 8 | 8 | eoq() implements Q* = round(sqrt(2*D*S/H)) with the required D=0 and H=0 guards returning 0. (`custom-nodes/eoq-optimizer.js:70-73`) |
| eoq_2 | 10 | 10 | detectViolations() catches both boundary cases: viral_spike (mean(last3) > 2.5x mean(prior9)) for the accelerating SKU and declining (mean(last3) < 0.5x mean(first3)) for the dying SKU, plus low_velocity and lead-time+volatility checks. analysis.md 4.2 provides a threshold/SKU/action table (SKU-007 viral, SKU-013 declining) defending each. (`custom-nodes/eoq-optimizer.js:105-145 + analysis.md 4.2`) |
| eoq_3 | 4 | 4 | Prompt enumerates a distinct action per flag (viral_spike->reorder above EOQ, declining->markdown/liquidate qty 0, low_velocity->fixed 30-day order, long_lead_time->reorder+timing flag) and returns a per-SKU action JSON. (`workflows/supply-chain-manager-starter.json (Inventory Exception Handler node)`) |
| sp_1 | 14 | 14 | 4-dimension weighted rubric with explicit weights reliability 35 / quality 30 / lead time 20 / cost 15, min-max normalized across the roster, with tier thresholds. analysis.md 4.3 defends the weights against a DTC business model with a real scored run (SUP-006 top, SUP-002 bottom). (`workflows/supply-chain-manager-starter.json (Supplier Performance Monitor node) + analysis.md 4.3`) |
| lg_1 | 6 | 6 | Prompt reasons over the 5 hard feasibility constraints first, then soft trade-offs (cost with a ~15% tie band broken by transit-time slack, plus mode reliability). (`workflows/supply-chain-manager-starter.json (Logistics Coordinator (LLM) node)`) |
| lg_2 | 10 | 10 | pickCheapestFeasible() filters against all 5 hard constraints (origin, dest, transit<=deadline, max_weight>=weight, perishable), returns null if infeasible, prices survivors by weight*cost_per_kg, and reduces to the minimum-cost option. (`custom-nodes/classical-logistics.js:68-87`) |
| lg_3 | 2 | 2 | Prompt sets use_classical_fallback: true when the choice is unambiguously cheapest (purely numeric) and false only when judgment was exercised; the Use Classical Fallback? IF node routes on it. (`workflows/supply-chain-manager-starter.json (Logistics Coordinator (LLM) node)`) |
| fn_1 | 10 | 9 | Aggregates each branch's parsed LLM/code output into key_findings and derives next_recommended_subgoal from the planner's own subgoal ordering (not hardcoded). Top band withheld only because the demo video is not recorded (analysis.md:56). (`workflows/supply-chain-manager-starter.json (Final Output code node)`) |
| Integrity deduction | — | 0 | Provided files unmodified |
| **Total** | **100** | **98** | |

### What went well
- All five classical algorithms are implemented correctly with proper edge-case guards: movingAverage, seasonalIndex, eoq (with D=0/H=0 guards), detectViolations, and pickCheapestFeasible.
- detectViolations() (the hardest TODO) genuinely catches both the viral-spike and dying-SKU boundaries, and analysis.md 4.2 defends each flag with concrete thresholds, triggering SKUs (SKU-007, SKU-013), and downstream actions.
- LLM system prompts are unusually well-specified: the Supplier Monitor states explicit 35/30/20/15 weights with min-max normalization and tier cutoffs, and the Logistics Coordinator cleanly separates hard constraints from soft tie-breaking plus the classical-fallback audit design.
- The 1713-word analysis.md is genuinely reflective, including real per-branch run metrics (tokens/cost/latency) and a thoughtful answer on the unknown-subgoal silent-failure case.

### What to improve (actionable)
- Record the required demo video showing an end-to-end run across >=3 of the 4 branches; it is the only thing capping MP-1 and FN-1 (analysis.md:56 notes it is not yet recorded).
- The 12-months-of-history seasonalIndex degenerates to a single observation per month (a per-month mean of one row); acknowledging this limitation in code is fine, but a smoothing or multi-year note in the recommendation would harden it.
- low_velocity never fires on this dataset (all SKUs move ~700+/yr, per analysis.md 4.2); the check is correct but untested against real data, so its threshold is unvalidated.
- Final Output's key_findings passes through the raw parsed component JSON; a short human-readable headline per branch would make the 'business-readable summary' goal even stronger.

### Automated checks
- ✅ All required files implemented
- ✅ Provided files unmodified
- ✅ 0/0 output artifacts committed
- ✅ Reflection 1713 words

### Resubmission
You may resubmit **once**. Push fixes to this repo, then notify the instructor; we'll re-grade as **Attempt 2 (final)**. This is attempt 1 of 2.

---
*Graded automatically with Claude Code against the course rubric. Questions → contact the instructor.*


---
<sub>🔎 **Autograder record** — attempt 1 of 2 · graded at commit `79b32a5` · delivered 2026-07-18T20:42:46Z. Commits pushed to `main` after this timestamp are treated as a resubmission.</sub>
