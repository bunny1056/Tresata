# Blocked-Credit Guard: GST ITC Eligibility Checker

An AI-assisted prototype that flags GST **input tax credit (ITC)** that is blocked under **Section 17(5)**, or that fails the **Section 16** conditions, of the CGST Act 2017 (India), as amended by the Finance Act 2025. Every decision cites the relevant clause and quotes the invoice text it is based on. A CA signs off on anything the system is unsure about.

> Built for the Tresata take-home assignment. All invoice data is **synthetic** and was made up by me. The rules are built from **public statute text**. This is not tax advice.

## Live demo
**[Open the clickable demo](<paste public artifact link>)**. Nothing to install.
1. Pick one of the 14 synthetic cases, or type your own invoice line.
2. Set the buyer's business activity, GSTR-2B status and payment status.
3. Click **Check ITC**. You get the decision, the eligible ₹ amount, the cited clause, the evidence and a 5-step audit trail.

## How it works
```
Invoice line ──> LLM classifier ──> Validator ──> Rules engine ──> Decision + clause + ₹ ──> CA review
 (text)          (1 of 14 labels,    (label valid?   (s.16 + 17(5),
                  evidence span,      evidence in     versioned
                  confidence)         text? conf≥0.7) rule pack)
                                           │ fail
                                           └──────────> HUMAN_REVIEW
Company profile (CA-confirmed) ──> exceptions in the rules engine
GSTR-2B + payment ledger      ──> Section 16 checks
```
- **The AI only labels.** It maps a line to a fixed category and must quote the exact words on the invoice that support the label.
- **Code decides.** Eligibility, the amount and the clause all come from plain, auditable Python in `rules.py`.
- **Humans are accountable.** Every BLOCKED, UNCLEAR and high-value line goes to a CA before GSTR-3B is filed.
- **Law is versioned.** Every decision is stamped with the rule-pack version, so past invoices can be re-run when the law changes, as it did with the retroactive Finance Act 2025 amendment.

## Repo layout
| File | Role |
|---|---|
| `demo.html` | Clickable browser demo. It uses the keyword baseline, plus a manual label override to show what the rules do with a correct label. |
| `app.py` | Streamlit UI that runs the full pipeline with the real LLM |
| `rules.py` | Rule pack: s.16(2)(aa), 16(2)(b), 16(4) and the 180-day check, plus 17(5) blocks and exceptions |
| `classifier.py` | `llm_classify` (OpenRouter, temperature 0, JSON output, validated) and `keyword_classify` (non-AI baseline) |
| `run_tests.py` | Runs all test cases and writes `results_*.json` |
| `test_cases.json` | 14 synthetic invoice lines with business context and the expected outcome |
| `data/synthetic_invoice_lines.csv` | The same test set as a CSV |
| `data/SOURCES.md` | Links to the public statute and schema sources |
| `Tresata_Brief_GST_ITC.pdf` | The written brief, including the architecture, risks and appendix |

## Run locally
Requires Python 3.9+. The core code uses only the standard library.
```bash
# 1. Test suite with the keyword baseline (no key needed)
python run_tests.py

# 2. Test suite with the LLM
export OPENROUTER_API_KEY=sk-or-...
export OPENROUTER_MODEL=meta-llama/llama-3.3-70b-instruct:free
python run_tests.py --llm --threshold 0.7

# 3. Interactive app
pip install streamlit
streamlit run app.py
```
**Model used:** `meta-llama/llama-3.3-70b-instruct:free` via OpenRouter, the free tier. Calls are spaced 3 seconds apart because of the free-tier rate limit.

## Test cases and results
These 14 adversarial cases are designed to break naive approaches:

| # | Invoice line (synthetic) | Expected | Why it's hard |
|---|---|---|---|
| 1 | HDPE granules | ELIGIBLE | Plain input |
| 2 | Innova Crysta for MD (manufacturer) | BLOCKED 17(5)(a) | Passenger vehicle |
| 3 | Innova Crysta, fleet (cab operator) | ELIGIBLE_EXCEPTION | Same car, different buyer |
| 4 | "Buffet for 300 pax" | BLOCKED 17(5)(b)(i) | Food without the word "food" |
| 5 | "Diwali dry-fruit boxes for dealers" | BLOCKED 17(5)(h) | Gift without the word "gift" |
| 6 | RCC foundation for moulding machine | ELIGIBLE | Contested question: is it plant and machinery or immovable property? |
| 7 | Factory shed works contract | BLOCKED 17(5)(c) | Immovable property |
| 8 | Hydraulic oil, not in GSTR-2B | NOT_ELIGIBLE_S16 | s.16(2)(aa) |
| 9 | Cartons, supplier unpaid >180 days | NOT_ELIGIBLE_S16 | s.16(2) second proviso |
| 10 | "Professional charges as per agreement" | HUMAN_REVIEW | Too vague; the model must not guess |
| 11 | "Car brake pads… for resale" | ELIGIBLE | The word "car" is a false trigger |
| 12 | "Corporate wellness plan… Cult centre" | BLOCKED 17(5)(b)(ii) | A gym described as "wellness" |
| 13 | Outdoor catering sub-contract (caterer) | ELIGIBLE_EXCEPTION | Same-category exception |
| 14 | Laptops donated under Sch VII | BLOCKED 17(5)(fa) | CSR without the word "CSR" |

| Classifier | Score | Failures |
|---|---|---|
| Keyword baseline | **7/14** | #4, #5, #12, #14 missed; #6, #11 falsely blocked; #1 sent to review needlessly |
| LLM (`llama-3.3-70b:free`) | **<fill in>/14** | <fill in> |

**Where it still fails:**
- **#6** is legally contested, so the correct behaviour is low confidence and HUMAN_REVIEW, not a confident answer.
- **#10** must come back as `unclear`; any guess counts as a failure.
- **Hallucinated evidence** is caught by `validate()`, which sends any line whose evidence is not in the invoice text to review.
- Fourteen cases are too few for per-category accuracy figures.

## Known limitations
- The s.16(4) check ignores the "annual return filed earlier" branch.
- Rule 42/43 apportionment, reverse charge and re-claiming reversed credit are out of scope.
- The 180-day check uses the run date rather than the GSTR-3B period.
- Business-context exceptions are structured inputs that a CA confirms in production; they are not inferred.

## Sources
- [CGST Act s.17: Blocked credits](https://www.knowyourgst.com/gstlaw/cgst-act/17-apportionment-of-credit-and-blocked-credits-17)
- [Finance Act 2025 retrospective amendment to s.17(5)(d)](https://taxguru.in/?p=1080251)
- [Section 16: ITC conditions and time limit](https://gstgyaan.com/section-16-eligibility-conditions-time-limit-for-taking-input-tax-credit)
- [GST e-invoice JSON schema](https://gstzen.in/a/format-of-e-invoice-schema.html)
