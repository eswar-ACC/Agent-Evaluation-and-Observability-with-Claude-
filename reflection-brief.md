# Reflection Brief — Evaluation and Observability Capstone

**Name:** B.Eswar Chand
**Date:** 22 November 2026

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux |
| Python version | Python 3.13.0 |
| Date run | 22 November 2026 |
| Ran any system live? (which) | Yes — System 1, Insurance Policy Extraction Pipeline. Anthropic API returned `200 OK`. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | [Copy total from `01-policy-pipeline/tests.txt`] |
| Routing output file | `01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 0 / 8 / 1 |

**1a. Retry boundary.**

The perturbation test was run using `tests/test_us01_retry.py`.

The test `test_ac_01_04_missing_source_halts_immediately` passed. The complete test file reported **14 passed**.

The missing-source case makes exactly one API call before escalation. When a required field has no supporting source, the system stops immediately instead of repeatedly calling the extraction model.

Retrying is futile because the required information does not exist in the source document. Another model call cannot recover unavailable evidence. It only increases API cost and latency. Escalation allows a human reviewer to investigate the missing information.

**Escalation evidence:** `test_ac_01_04_missing_source_halts_immediately PASSED`.

**1b. Reading the router.**

One example is policy `POL-2025-006`.

Its decision was `human_review`, even though its confidence scores were high:

- coverage_limit: 1.0
- deductible: 1.0
- endorsements: 0.95
- exclusions: 0.95
- policy_type: 1.0
- premium_amount: 1.0

The reason was `reviewer_disagreement=['deductible']`.

The reviewer signal sent this policy to human review. If the router had trusted model confidence alone, it could have approved the extraction automatically, even though a reviewer disagreed about the deductible.

**1c. Where the aggregate lies.**

The calibration report produced these results:

| Policy type | Field | Samples | Confidence | Accuracy | Brier score |
|---|---|---:|---:|---:|---:|
| auto | premium_amount | 3 | 0.95 | 1.00 | 0.003 |
| home | deductible | 1 | 0.90 | 1.00 | 0.010 |
| umbrella | exclusions | 2 | 0.93 | 0.00 | 0.865 |

**Overall Brier score: 0.291**

The most important finding is `umbrella × exclusions`: the model predicted 93% confidence, but its observed accuracy was 0%.

Slicing by `policy_type × field` reveals weaknesses hidden by an aggregate score. It identifies specific combinations where the model is highly confident but consistently wrong, allowing the system to apply stricter review or routing rules.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | [Copy total from `02-mortgage-extraction/tests.txt`] |
| Document runs | `appraisal_informal_sqft.txt`, `income_missing_bonus.txt`, `income_sum_mismatch.txt` |
| Classified type | [Copy classified type from replay output] |

**2a. Two guarantees.**

The `income_sum_mismatch.txt` replay tested mathematical consistency.

The output showed:

- `consistent`: false
- `field`: `total_monthly_income`
- `calculated`: `9642.17`
- `stated`: `10892.17`
- `delta`: `-1250.0`

The discrepancy was detected even though the extraction was valid structured JSON.

Schema or tool enforcement guarantees that the model returns data in the required structured format, such as valid JSON with the expected fields and types.

The independent validator checks whether the extracted values satisfy business rules, including mathematical consistency.

These are different guarantees. Valid JSON can still contain incorrect arithmetic. The schema cannot guarantee that the extracted numbers are correct, while the mathematical validator cannot guarantee that the model interpreted the original document correctly.

**2b. Refusing to fabricate.**

The `income_missing_bonus.txt` replay produced:

`"bonus_monthly": null`

The system returns `null` instead of inventing a bonus amount because the source document does not provide that information.

The schema permits this behavior through a nullable field. This preserves the distinction between information that is available and information that is missing.

**2c. Normalization.**

The appraisal replay demonstrated normalization:

- Source text: `about 2,400 sq ft`
- Extracted value: `2400`

Normalization during extraction produces consistent machine-readable values. Downstream components can directly compare, calculate, validate, and store them without repeatedly interpreting different text formats.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | [Copy total from `03-supply-chain/tests.txt`] |
| Briefing file | `meridian-normal.txt` |
| Section the conflict landed in | Contested |

**3a. Annotate, don't arbitrate.**

The normal Meridian briefing reported conflicting on-time delivery rates:

- **95.0 percent** — `supplier_audit`, as of `2026-04-10`.
- **78.0 percent** — `logistics`, as of `2026-04-05`.

The conflict was placed under the **Contested** section and marked for escalation because it is a high-impact metric.

Preserving both claims allows a reviewer to investigate the difference instead of silently selecting one number. This avoids creating false certainty and helps identify possible reporting differences or operational risks.

**3b. Source goes dark.**

The timeout experiment was run using:

`--offline --simulate-timeout`

The briefing explicitly reported:

`Sources unavailable: logistics unavailable (timeout)`

It also showed:

- `late_shipment_count` — missing source because of a timeout reading logistics.
- `production_capacity_utilization` — no source reported this metric, so it was marked incomplete and escalated.

An unreachable source is different from a source that successfully returns no relevant information. A timeout is an access failure, whereas no information is a valid empty result.

The coordinator records the timeout as an evidence gap and continues processing the other successful sources. Therefore, the investigation still completes without hiding the failure.

**3c. Dates as a guardrail.**

Two claims about Meridian's on-time delivery rate were:

- `95.0 percent` — supplier audit, dated `2026-04-10`.
- `78.0 percent` — logistics, dated `2026-04-05`.

The dates show that the values refer to different reporting dates. Without dates, a reader might interpret the difference as a direct contradiction without considering a change over time or different reporting periods.

Requiring dates provides temporal context and makes comparisons more meaningful.

---

## 4. Synthesis

**4a. One principle.**

System 1, the validated insurance policy pipeline, most clearly demonstrated the principle of evaluating the output rather than trusting the model's response.

The perturbation test `test_ac_01_04_missing_source_halts_immediately` passed. It verified that a required field without supporting evidence is escalated immediately instead of being repeatedly sent to the model.

The calibration report also revealed that the umbrella exclusions field had 93% predicted confidence but 0% observed accuracy. This demonstrates why model confidence alone cannot establish correctness.

**4b. Confidence ≠ correctness.**

System 1 demonstrated this most clearly.

The calibration report showed that the umbrella exclusions field had high predicted confidence of 0.93 but an observed accuracy of 0.00. A system that trusted confidence alone could have approved incorrect information.

Calibration, validation, and independent routing signals are therefore necessary to reduce the risk of incorrect automated decisions.

**4c. Apply it.**

A practical example is an invoice-processing workflow where an LLM extracts supplier names, invoice dates, tax amounts, line items, and totals from scanned invoices.

I would first use validated extraction with escalation. The system should enforce a structured schema, validate required fields and mathematical consistency, and send unsupported or inconsistent records for human review rather than inventing missing information.

I would instrument:

- Schema-validation failure rate.
- Missing-field and unsupported-value rate.
- Arithmetic discrepancy rate.
- Retry count and API calls per document.
- Human-escalation rate.
- Extraction accuracy from sampled human-reviewed records.
- Confidence calibration over time.

These measurements would reveal when extraction quality degrades, when the model starts guessing, or when validation and escalation mechanisms stop working as expected.
