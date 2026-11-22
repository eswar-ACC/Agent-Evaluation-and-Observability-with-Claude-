# Perturbation Log

This log records one deliberate failure or variation scenario for each of the three systems. Each entry includes the scenario, command, expected behavior, observed result, and comparison with the normal run.

## System 1 — Insurance Policy Extraction Pipeline

### Change / Perturbation

Used the missing-source test scenario in `test_ac_01_04_missing_source_halts_immediately`.

The recorded model response contains `endorsements: None`, representing a case where the model could not find Schedule A. The retry configuration allows up to three retries.

This is a deliberate test input, not a manual modification of the policy document.

### Command

```bash
.venv/bin/pytest tests/test_us01_retry.py -v
```

Output saved as: `04-perturbations/policy-retry-test.txt`

### Prediction

The extractor should recognize that a required field has no source evidence and stop immediately instead of repeatedly calling the model. It should return a `RetryFutileEscalation` for the `endorsements` field after exactly one API call.

### Observed Result

The test passed:

```text
test_ac_01_04_missing_source_halts_immediately PASSED
14 passed in 3.59s
```

The test assertions verify that the result is a `RetryFutileEscalation`, the affected field is `endorsements`, the detected pattern is `endorsements_absent`, and the client call count is exactly one.

### Difference from the Unperturbed Run

The normal live policy pipeline completed successfully with HTTP 200 OK. Its routing output contained:

- 8 `human_review` decisions
- 1 `spot_check` decision
- 0 `auto_approve` decisions

The perturbed test demonstrates a different failure path: when a required source is missing, the extractor stops retrying and escalates rather than continuing extraction.

---

## System 2 — Mortgage Document Extraction System

### Change / Perturbation

Used the supplied fixture:

`fixtures/documents/income_sum_mismatch.txt`

The fixture contains monthly earnings components and a stated monthly total of 10,892.17.

The extracted component values are:

- Base salary: 5,416.67
- Commission: 2,140.00
- Performance bonus: 1,250.00
- Overtime: 385.50
- Auto allowance: 450.00

The sum of these components is 9,642.17, which differs from the stated total by 1,250.00.

This is a supplied mismatch fixture, not a claim that the source document was manually edited.

### Command

```bash
.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay
```

Output saved as: `02-mortgage-extraction/income_mismatch.txt`

### Prediction

The extractor should parse the income fields, but the mathematical consistency validator should detect that the calculated total does not match the stated total. The result should be marked inconsistent and include a discrepancy.

### Observed Result

The output reported:

```text
calculated: 9642.17
stated: 10892.17
delta: -1250.0
consistent: false
```

The discrepancy was reported for `total_monthly_income`.

### Difference from the Unperturbed Run

The `income_missing_bonus.txt` replay represented unavailable bonus and other income fields as `null` and returned `consistent: true` with no discrepancies.

In contrast, the mismatch fixture contains a stated total that does not reconcile with the extracted components. The validator detects and reports the numerical inconsistency instead of silently accepting the total.

---

## System 3 — Supply Chain Risk Investigation

### Change / Perturbation

Enabled the simulated logistics-source timeout using the `--simulate-timeout` option.

### Command

```bash
.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout
```

Output saved as: `03-supply-chain/meridian-timeout.txt`

### Prediction

The coordinator should continue the investigation with the sources that remain available. It should report the logistics timeout and mark metrics that depend on logistics data as incomplete rather than aborting the entire investigation.

### Observed Result

The briefing reported:

```text
Sources unavailable: logistics unavailable (timeout)

late_shipment_count
[missing source: timeout reading logistics]

production_capacity_utilization
[missing source: no source reported this metric]
```

The high-impact `production_capacity_utilization` metric was escalated.

### Difference from the Unperturbed Run

In the normal offline run, logistics data was available. It contributed a late shipment count of 11 and an on-time delivery rate of 78.0%.

The supplier audit reported an on-time delivery rate of 95.0%, creating a contested result between the two sources.

With the simulated timeout, logistics data was unavailable. The late shipment count became incomplete, and the logistics on-time delivery figure could no longer participate in the comparison. The investigation continued using the remaining available source data.

---

## Overall Observation

The three perturbations demonstrate different resilience behaviors:

1. The policy extractor stops futile retries when a required source is missing and escalates the affected field.
2. The mortgage extractor detects a numerical inconsistency between extracted income components and the stated total.
3. The supply chain coordinator recovers from a source timeout, continues synthesis with available sources, and records missing evidence as incomplete.

These experiments show why explicit validation, controlled retry behavior, and graceful handling of partial source failures are important in reliable AI systems.