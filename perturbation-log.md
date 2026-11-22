# Perturbation Log

## System 1 — Validated, Routed Pipeline

- **Change I made (file + what I changed):** Used the retry and failure scenarios in `tests/test_us01_retry.py` to exercise malformed responses, validation failures, and missing-source behavior.
- **Command I ran:**
  ```bash
  .venv/bin/pytest tests/test_us01_retry.py -v
  ```
- **What I predicted:** Invalid extraction responses would trigger retries, while a missing required source would stop processing immediately rather than allowing an incomplete extraction to continue.
- **What actually happened:**
  ```
  14 passed in 3.59s
  test_ac_01_04_missing_source_halts_immediately PASSED
  ```
- **How this differs from the unperturbed run:** The normal pipeline successfully extracted policy information and returned HTTP 200 OK. The failure scenarios demonstrated that validation and missing-source conditions are handled explicitly rather than silently accepting unreliable results.

## System 2 — Schema-Enforced Two-Pass Extraction

- **Change I made (file + what I changed):** Used the `income_sum_mismatch.txt` fixture, which contains an intentionally inconsistent stated monthly income total.
- **Command I ran:**
  ```bash
  .venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay
  ```
- **What I predicted:** The extractor would identify the income components, calculate their total, compare it with the stated total, and report a validation discrepancy.
- **What actually happened:**
  ```json
  {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0
      }
    ]
  }
  ```
- **How this differs from the unperturbed run:** A consistent extraction can pass validation. With the mismatched income fixture, the system detected a $1,250.00 discrepancy and marked the result inconsistent instead of accepting the stated total.

## System 3 — Multi-Source Synthesis

- **Change I made (file + what I changed):** Enabled the simulated logistics-source timeout using the `--simulate-timeout` command-line option.
- **Command I ran:**
  ```bash
  .venv/bin/supply-chain-investigate meridian --offline --simulate-timeout
  ```
- **What I predicted:** The investigation would continue using available sources, identify the logistics source as unavailable, and distinguish missing information caused by a timeout from information that no source reported.
- **What actually happened:**
  ```
  Sources unavailable: logistics unavailable (timeout)
  ```
  The `late_shipment_count` metric moved to Incomplete because the logistics source timed out. `production_capacity_utilization` also remained Incomplete because no source reported it.
- **How this differs from the unperturbed run:** In the normal run, logistics provided an 11-shipment late count and a 78% on-time delivery rate, conflicting with the supplier audit's 95%. In the simulated-timeout run, those logistics claims were unavailable. The system continued with the remaining evidence instead of aborting the entire investigation.

---

## Overall Observation

These experiments demonstrate three important reliability behaviors:

1. **Policy extraction:** Validation failures and missing required sources are handled through retries and controlled stopping.
2. **Mortgage extraction:** Arithmetic inconsistencies are detected instead of being silently accepted.
3. **Supply-chain synthesis:** Partial source failures are isolated, and unavailable evidence is distinguished from genuinely unreported metrics.

The experiments show why observability must capture not only successful outputs but also retries, validation discrepancies, source failures, and escalation decisions.