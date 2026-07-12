# Analysis — Intelligent E-commerce Supply Chain Manager

## 4.1 Algorithm comparison

**SKU-007 (Wool Gloves) — the anomaly case.** The classical engine computes a moving average of 376.7 (mean of the last three months: Oct 218, Nov 394, Dec 518) times a seasonal index of 0.94 for January, giving a forecast of **354 units**. `statistical_confidence` comes out at **0.25** — low, because the formula's own coefficient-of-variation check sees the huge swing between the trailing months and the rest of the year and correctly flags itself as shaky. The LLM's revision: **354 units, `delta_pct: 0`** — it defers entirely, with the reasoning *"No qualifying contextual signals were found in the notes... deferring to the classical forecast despite low statistical confidence."*

That's not a failure of DF-3's design — it's the correct behavior given the input. The `notes` field the Forecast Context Adjuster receives is always an empty array in this dataset (no external context ever gets attached to a forecast row), so there is genuinely nothing for the LLM to act on. Low *statistical* confidence and a missing *contextual* signal are different things, and the prompt explicitly tells the model not to conflate them. Catching SKU-007's viral trajectory turns out to be EOQ-2's job, not DF-3's — `detectViolations()` flags `viral_spike` on the same SKU using the same underlying data, and *that* branch does take action (see 4.2).

**SKU-001 (Sunglasses) — the well-behaved case.** Classical forecast: moving average 32.3 × seasonal index 0.31 (January is deep off-season for sunglasses) = **10 units**. `statistical_confidence` is **0** here too — the formula is just as unsure about SKU-001 as it is about SKU-007, because the coefficient-of-variation check doesn't distinguish "noisy but predictable seasonal swing" from "someone should look at this." The LLM again returns **10 units, `delta_pct: 0`**, same reasoning pattern.

**Which would I trust to size a reorder?** For SKU-001, the classical number outright — a 10-unit January order for sunglasses is exactly what a seasonal business should do, and there's no reason to second-guess it. For SKU-007, I'd trust the **classical number as a floor, not a ceiling** — 354 units is defensible given the data the forecaster has, but the EOQ branch's `viral_spike` flag and its resulting recommendation of 560 units (roughly 1.3× the naive EOQ figure, sized off the accelerating trend) is the number I'd actually order against. The forecast and the inventory recommendation shouldn't be read in isolation; the forecast tells you demand is elevated, and EOQ-3's exception handling is what actually adjusts the order size for the fact that demand is *still climbing*.

## 4.2 EOQ assumption analysis

| Flag | Threshold | Triggering SKU | Downstream action | Agree? |
|---|---|---|---|---|
| `viral_spike` | mean(last 3mo) > 2.5× mean(prior 9mo) | SKU-007 (18 on hand vs. 80 reorder point) | `reorder`, qty 560 (~1.3× the 431-unit EOQ figure, sized off the trend) | Yes — the formula's own number is already known to be lagging an accelerating trend; ordering above it and flagging urgency is the right call given the stockout is already active. |
| `declining` | mean(last 3mo) < 0.5× mean(first 3mo) | SKU-013 (540 on hand vs. 80 reorder point, ~6.75×) | `liquidate`, qty 0 | Yes — reordering into a falling-demand, already-massive overstock would only compound the capital lockup. |
| `low_velocity` | annual demand < 60 units | *(none in this dataset)* | — | N/A — no SKU in `current_inventory.csv` has annual demand this low; all 15 SKUs move at least ~700+ units/year. The check is implemented and would fire on a slow-mover if one existed, but I have no real trigger to evaluate here. |
| `long_lead_time` | lead_time_days > 28 **and** demand is volatile (coefficient of variation > 0.4) | SKU-005, SKU-006, SKU-009 (all winter items with 30–42 day lead times) | `reorder` at the standard EOQ figure, with a note to place the order earlier than the reorder point would normally trigger | Yes — these SKUs aren't in crisis, so the standard EOQ quantity is still correct; the only adjustment needed is *timing*, which is what the recommendation actually does. |

**Perishability — intentionally not implemented**, per the assignment's explicit note that none of the demo SKUs are perishable. A real implementation would need a holding-cost-cliff check — something closer to "does this SKU have a hard shelf-life window that `holding_cost_per_unit_year` doesn't capture" — since the current EOQ model treats holding cost as a smooth linear cost, not a cliff at expiration. I didn't add a placeholder check for it since there's no data to validate it against.

## 4.3 Supplier rubric defense

Weights: **reliability 35% / quality 30% / lead time 20% / cost 15%** (sums to 100). I assumed a **DTC e-commerce business model** — closer to Coastal Goods' actual scenario than, say, a hospital supply chain (which would push reliability/quality even higher and cost near zero) or a bulk-materials distributor (which would weight cost and lead time far more heavily and tolerate more quality variance). Reliability got the top weight because a late shipment cascades directly into the stockouts this entire workflow exists to catch — it's the failure mode, not just an inconvenience. Quality is nearly as high because defects compound into returns and lost trust and, unlike a late shipment, can't be absorbed with a few extra days of safety stock. Lead time sits lower because EOQ and reorder points already partially buffer against it — it's a planning input more than an acute risk. Cost is weighted lowest: `payment_terms_days` affects margin and cash position, not whether the customer's order shows up correct and on time.

From a real scored run: **top-ranked SUP-006** (Bavaria Paper Mills, score 87.2, `preferred` — 100 on both reliability and quality, the best in the roster on the two dimensions weighted highest) and **bottom-ranked SUP-002** (Pacific Rim Trading, score 28.1, `deprioritize` — 0 on both reliability and quality, despite having the *best* cost score in the roster because of its 60-day payment terms). This matches intuition directly: SUP-002 has the worst on-time rate (0.84) and highest defect rate (0.022) in `suppliers.csv`, and no amount of favorable payment terms should outweigh that under a DTC model where a stockout or defective shipment costs far more than a few extra days of float.

## 4.4 Run metrics

Three real end-to-end runs, covering three of the four `next_subgoal` branches (all measured against Claude Sonnet 4.6 via the Anthropic API, `usage.input_tokens`/`usage.output_tokens` summed across every LLM call in the run):

| Branch | LLM calls | Input tokens | Output tokens | Cost | Wall-clock |
|---|---|---|---|---|---|
| `forecast` | 16 (1 planner + 15 per-SKU adjustments) | 8,614 | 1,926 | **$0.0547** | 16.2s |
| `inventory` | 6 (1 planner + 5 flagged-SKU exceptions) | 4,319 | 1,220 | **$0.0313** | 14.1s |
| `supplier` | 2 (1 planner + 1 batch scoring call) | 2,552 | 1,140 | **$0.0248** | 19.5s |

All three land comfortably inside the assignment's ~$0.05–$0.10 target — the `inventory` and `supplier` branches in particular come in well under, since the inventory branch's IF-gate filter (see the workflow's architecture notes) keeps the LLM off the 10 clean SKUs entirely, and the supplier branch scores all six suppliers in a single batched call rather than one call each.

**What surprised me:** the `forecast` branch's cost is dominated almost entirely by input tokens (8,614 in vs. 1,926 out) — each of the 15 per-SKU Forecast Context Adjuster calls resends the full forecast row, and with no prompt caching wired up (each is a cold, independent HTTP call), that adds up faster than the output side does. If I were optimizing this for a larger SKU catalogue, batching the forecast-adjustment calls the way the Supplier Monitor batches all six suppliers into one call would likely cut the forecast branch's cost by more than half.

## 4.5 Reflection on the four primer questions

**1. Where does a classical algorithm produce a wrong answer the LLM catches, and vice versa?** EOQ would happily recommend reordering SKU-013 (397 units) despite its demand actively declining — `detectViolations()` flags it, and the LLM exception handler correctly overrides that to `liquidate`. In the other direction: the Logistics Coordinator's `use_classical_fallback` design means the deterministic planner independently re-derives the LLM's answer whenever the LLM defers — in every test run the two agreed, which is exactly the audit property that would catch an LLM arithmetic error before it reached a human, had one occurred.

**2. If the planner emitted `next_subgoal: "audit_warehouse"` — an unknown value — what happens, and what would I change?** The Switch node's `fallbackOutput: "extra"` branch isn't wired to anything, so the run would complete with `status: success` and simply produce no `Final Output` — a silent failure, exactly as the primer flags it. I'd wire that fallback output into a Code node that constructs an explicit `{error: "unrecognized next_subgoal", value, plan_id}` record and routes it to `Final Output`, so an unrecognized subgoal fails loudly instead of vanishing.

**3. Why does EOQ give a confident, wrong answer for SKU-013?** The formula only sees `annual_demand` — a trailing-12-month total — and has no notion of trend direction. A SKU that sold heavily early in the year and is now declining still shows a healthy annual total, so EOQ recommends reordering into an already 6.75×-overstocked position. The `declining` flag exists specifically because total volume and trend direction are different signals, and the formula only has access to the first one.

**4. Why is supplier scoring an LLM job and EOQ a classical job — what would change if I swapped them?** EOQ is a closed-form optimization with one mathematically correct answer given its inputs — there's no judgment call, just arithmetic, and an LLM computing it would be slower, non-reproducible, and harder to audit for zero benefit. Supplier scoring is inherently a weighted, business-context-dependent judgment call — the "right" weights depend on the business model (as 4.3 shows), and an LLM can apply an explicit, explainable rubric from a sentence of context. Swapping them would mean paying LLM latency and cost for arithmetic that doesn't need it, and losing the auditability of a fixed rubric.

## Demo video

*Not yet recorded — see `docs/submission.md` §5 for the required format (≤5 minutes, all five components, both branches, SKU-007/SKU-013 walkthrough).*
