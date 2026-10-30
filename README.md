# Invoice Extraction with a Review Queue

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![Platform](https://img.shields.io/badge/Run%20on-Google%20Colab-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Cost](https://img.shields.io/badge/Cost-Free-success)

Extract fields from invoices, check them, and send anything doubtful to a human reviewer. A blurred total or a mixed currency never becomes a trusted record.

## Table of Contents
1. [Problem](#problem)
2. [What I Built](#what-i-built)
3. [Results](#results)
4. [Review Queue](#review-queue)
5. [Optional Groq Second Opinion](#optional-groq-second-opinion)
6. [Acceptance Tests](#acceptance-tests)
7. [Tech Stack](#tech-stack)
8. [How to Run](#how-to-run)
9. [Limitations](#limitations)
10. [About Me](#about-me)

## Problem
A small office wants searchable invoice fields: invoice id, vendor, currency, total and due date. But a wrong total in an accounting record is worse than an empty one. So this is not only extraction. It is extraction + validation + a reviewer queue, and it is **not** a payment system.

## What I Built
- **20 fictional invoices** in 3 layouts, saved as images and as OCR-style text. Layouts A and B were used to write the rules. **Layout C (6 invoices) is held out** to see how the rules do on something new
- On-purpose problem cases: 2 blurred totals, 3 missing due dates, 1 yen invoice with decimals, 1 invoice with two currencies, 1 duplicate invoice, and 1 invoice with the same total as another one (which must NOT be called a duplicate)
- **Rule-based extraction**. If a field is missing or unreadable it stays `None` with a reason, and nothing is guessed. Each field also keeps the line number it came from
- **Validation with `Decimal`**: allowed currencies, decimal places per currency (yen = 0, others = 2), total above 0, duplicate identity = same vendor + same invoice id
- **SQLite review queue** with an audit table. Original extraction is stored forever. Editing an approved amount sends the invoice back to review
- **Export of accepted invoices only** (`accepted_invoices.csv`)
- **Honest metrics split by layout**, including the number that matters most: bad invoices wrongly accepted

## Results
First run, before any reviewer action:

| Group | Invoices | id / currency / total exact | Due date exact | Accepted automatically | Sent to review |
|---|---|---|---|---|---|
| Layouts A + B (rules written on these) | 14 | 1.00 | 1.00 | 0.64 | 5 |
| Layout C (held out) | 6 | 1.00 | **0.17** | **0.00** | 6 |
| All | 20 | 1.00 | 0.75 | 0.45 | 11 |

- **Bad invoices wrongly accepted: 0** (this number must always be 0)
- Good invoices sent to review: 3 (extra work, not dangerous). All 3 are layout C invoices, because the rules had never seen the `Due on:` line, so they left the due date empty instead of guessing
- Empty (null) fields after extraction: 11
- Decimal errors caught: 1 (the yen invoice with 2 decimals)

So the rules work on the layouts they were written for, and on a new layout they fail safely: they send the invoice to review. They do not return a wrong value.

## Review Queue
11 invoices went to the queue, each with its reason:

| Reason | Count |
|---|---|
| Due date not found | 8 |
| Total is unreadable (blurred) | 2 |
| Decimal error (JPY needs 0 decimals, got 2) | 1 |
| Currency conflict (EUR and USD on one invoice) | 1 |
| Possible duplicate of row 1 | 1 |

(Some invoices have more than one reason, so the counts add up to more than 11.)

Reviewer actions in the run:
- Row 5 (missing due date): approval was **refused** (`due_date is empty`). After the reviewer typed the date from the paper copy, approval worked
- Row 4 (blurred total): approval was **refused** (`total is empty`). A blurred total can not be approved without a real value
- Final export: **10 of 20 invoices** accepted, with 5 rows in the audit table

## Optional Groq Second Opinion
Set `USE_LLM = True` and paste a Groq key (model `openai/gpt-oss-120b`). The model only gets the plain invoice text. Its answer is shown as a **suggestion**, goes through the same checks, and is never auto-accepted.

What happened on the 10 invoices in the queue:
- It read the layout C due dates that the rules missed (5 invoices)
- It returned `null` for both blurred totals and both genuinely missing due dates. It did not guess any of those 4 values
- It **did make one mistake that matters**: on the invoice with two currencies it quietly picked USD. A conflict should stay in review, which is exactly why model output is a suggestion and not a decision

## Acceptance Tests
| Test | Result |
|---|---|
| Blurred totals stay `None` and stay in review | PASS |
| Row 16 is flagged as a duplicate, row 18 (same total as row 0) is not | PASS |
| Editing an accepted total sends it back to review and writes an audit row | PASS |

## Tech Stack
| Area | Tools |
|---|---|
| Language | Python |
| Extraction and validation | regex, `decimal`, `datetime` |
| Data | pandas |
| Review queue and audit | SQLite (built into Python) |
| Images | Pillow |
| Charts | matplotlib |
| Optional LLM | Groq via the `openai` client |
| Environment | Google Colab |

## How to Run
1. Open `Invoice_Extraction_Review_Queue.ipynb` in Google Colab
2. Click **Runtime > Run all**
3. No dataset is needed, all invoices are generated inside the notebook (seed 42). The Groq step is off by default, so no API key is needed for the main run

Or locally: `pip install -r requirements.txt` and open the notebook in Jupyter.

## Limitations
- Invoices are synthetic. The notebook starts from OCR-style text, so this is **not** an OCR accuracy claim
- Only 20 invoices and 3 layouts. The held-out layout has just 6 invoices
- The rules only know the layouts they were written on. A new layout goes to review until a new rule is added
- Duplicate identity is vendor + invoice id. A real system would also need to handle vendor name spelling differences

## About Me
**Akshat Kesharwani** - Data Analyst / Data Scientist (fresher)

- Portfolio: https://akshatkesharwani-info.github.io/
- GitHub: https://github.com/akshatkesharwani-info

Idea source: "Top 50 Projects to Build in 2026" (Yasir Arafat Sharfi). The build, data and code here are my own.
