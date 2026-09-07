# Project Summary — Customer Support Agent Evaluation

## Objective

Build a small e-commerce customer-support routing agent and evaluate it like a
real ML system: measure it, find out *what kind* of mistakes it makes, make one
focused prompt change, and prove whether the change helped. Tracing is wired
through OpenTelemetry into LangSmith so every prediction is inspectable.

The agent reads one support ticket and assigns exactly one category:

| Category | What belongs here |
| --- | --- |
| `order_status` | where an order is, tracking, delivery ETA, non-delivery |
| `refund_request` | customer wants their money back (item itself is fine or already returned) |
| `product_issue` | item arrived broken, wrong, defective, or not as described |
| `account_help` | login, password, address, payment method changes |
| `other` | general questions, feedback, browsing |

## Approach

| Piece | Choice |
| --- | --- |
| Agent | LangGraph graph with a single `classify` node (kept as a graph so retrieval / tool / self-check nodes can be added later without touching the eval harness) |
| Model | `gpt-4o-mini`, `temperature=0`, Pydantic structured output (`category` + one-sentence `reasoning`) |
| Dataset | 150 human-labeled tickets from `Week 4_ AI Evals Data.csv` (`Ticket text` + `True category`); `order_status` 24, `refund_request` 42, `product_issue` 24, `account_help` 30, `other` 30; worked `EX` row skipped |
| Metrics | accuracy, per-class precision / recall / F1, confusion matrix (scikit-learn) |
| Observability | one OpenTelemetry span (`customer_support.classify_ticket`) per ticket, exported to LangSmith project `customer-support-evals` |
| Human review | export predictions to CSV, add validator comments, cluster failures, pick the costliest one |
| Judge | separate `gpt-4o-mini` LLM-as-judge call grading the model's own reasoning sentence |

## Evaluation loop

1. Run the baseline classifier on all 150 tickets.
2. Read the metrics and the confusion matrix.
3. Export `results_v1.csv`, filter to `correct == FALSE`, annotate each failure.
4. Cluster the annotations into failure categories; pick the one most expensive in production.
5. Make **one** focused edit to the classifier prompt.
6. Re-run, diff the two runs ticket-by-ticket, and separate wins from regressions.
7. Run the LLM judge on the reasoning field as a sanity check that right labels have real justifications.

## Results

Numbers are for the 150-ticket dataset and match the current `results_v1.csv` /
`results_v2.csv`. (The notebook's *saved* cell outputs are from an earlier
100-ticket version of the dataset.)

| Run | Accuracy | Correct |
| --- | ---: | ---: |
| Baseline prompt (v1) | 96.0% | 144 / 150 |
| Improved prompt (v2) | 96.7% | 145 / 150 |
| Change | **+0.7 pp** | +1 net (2 wins, 1 regression) |

### Per-class precision / recall / F1

| Category | N | P v1 | R v1 | P v2 | R v2 |
| --- | ---: | ---: | ---: | ---: | ---: |
| `order_status` | 24 | 0.96 | 1.00 | 1.00 | 0.96 |
| `refund_request` | 42 | 1.00 | 0.93 | 1.00 | 0.98 |
| `product_issue` | 24 | 1.00 | 1.00 | 0.96 | 1.00 |
| `account_help` | 30 | 1.00 | 0.90 | 1.00 | 0.90 |
| `other` | 30 | 0.86 | 1.00 | 0.88 | 1.00 |

### What the baseline got wrong

Six errors, all in three clusters — the model routed on a surface signal instead
of the customer's *primary* problem:

- `t015` "delivered but I never got it — refund please" → `order_status` (anchored on "delivered").
- `t049` / `t050` price-adjustment requests ("item dropped in price after I bought it") → `other` instead of `refund_request`.
- `t114`–`t116` newsletter promo code rejected at checkout → `other` instead of `account_help`.

`other` precision (0.86) is the weakest cell: refund and account_help tickets
leak into it.

### The prompt change

One edit to `IMPROVED_PROMPT`: four disambiguation rules plus a forced two-step
"identify the primary problem first, then classify" instruction. The
load-bearing rules:

1. Damaged / wrong / defective item mentioned → `product_issue`, regardless of the remedy asked for.
2. Order never received → `order_status`, even if the customer mentions wanting money back.
3. "Where is my refund?" (already initiated) → `refund_request`.
4. Checkout-blocking site bugs → `account_help`.

### Wins and regressions

Three tickets changed prediction: **2 wins, 1 regression.**

- **Wins:** `t015` and `t049` both moved to `refund_request`. Overall
  `refund_request` recall went 0.93 → 0.98.
- **Regression:** `t025` "carrier delivered to the **wrong address**" (true
  `order_status`) flipped to `product_issue` — rule 1 over-fired on the word
  "wrong".
- **Still wrong in both:** `t050` (price adjustment read as `other`) and
  `t114`–`t116` (promo code read as `other`). The prompt change did not target
  these clusters.

Net +1 correct. The regression is the expected cost of a broad rule — almost no
prompt change is strictly Pareto-better, and understanding the trade is more
valuable than the headline number.

## LLM-as-judge

A second `gpt-4o-mini` call grades the one artifact with no ground-truth label:
the model's own `reasoning` sentence. Anchored 1–5 scale, plus a boilerplate
flag and a `good` / `borderline` / `bad` verdict. It judges reasoning *quality
only* — it does not re-classify. Each judgement is its own OpenTelemetry span
(`customer_support.judge_reasoning`).

On the earlier 100-ticket run it produced a mean score of ~4.1 / 5 with ~7%
flagged as boilerplate, and — cross-tabbed against label correctness — the
wrong-label rows still scored as `good` reasoning. The lesson holds: the
baseline's mistakes are structural (wrong category boundary), not stylistic, so
reasoning-quality grading does not surface them. Treat the judge as a relative
v1-vs-v2 signal, calibrated against human grades. *This cell has not been re-run
against the 150-ticket set.*

## Observability

Every classification and every judgement is wrapped in an OpenTelemetry span
carrying `eval.*` attributes (`example_id`, `run_name`, `prompt_version`,
`true_category`, `predicted_category`, `correct`, `reasoning`). LangSmith
receives the spans (`LANGSMITH_OTEL_ENABLED=true`), so each CSV row links back to
the exact model run that produced it. An early `404` on span export was fixed by
pointing the OTLP exporter at `https://api.smith.langchain.com/otel/v1/traces`.
See [`docs/opentelemetry_integration_one_pager.md`](docs/opentelemetry_integration_one_pager.md).

## Deliverables

| Artifact | Contents |
| --- | --- |
| [`week4_customer_support_evals.ipynb`](week4_customer_support_evals.ipynb) | Executed notebook: agent, eval harness, baseline vs improved runs, LLM judge, tracing |
| [`README.md`](README.md) | Setup and run instructions |
| `Week 4_ AI Evals Data.csv` | The 150-ticket labeled dataset the notebook loads |
| [`results_v1.csv`](results_v1.csv) / [`results_v2.csv`](results_v2.csv) | Per-ticket prediction records (150 rows) with `validator_comment` / `failure_category` columns for human review |
| `Customer-Support-Agent-Evaluation-Summary.pdf` | Primer, architecture diagram, code-flow walkthrough, and results in one document |
| [`docs/opentelemetry_integration_one_pager.md`](docs/opentelemetry_integration_one_pager.md) | Tracing setup and troubleshooting |
| `Week 4_ AI Evals (E-Commerce Customer Support Agent).xlsx` and `Tosubmit/` | Human-review workbook for failure clustering and optional judge alignment |

## Takeaways

- **Accuracy alone is a trap.** 96% looked fine; per-class precision and the
  confusion matrix showed where the errors actually concentrate (`other`
  precision, `refund_request` / `account_help` recall).
- **Validator comments are the highest-value artifact.** Numbers say *that*
  something is wrong; human notes say *what* and *why*.
- **One focused prompt change per iteration** keeps attribution clean — you know
  which rule bought the +1 and which one caused the regression.
- **Broad rules cut both ways.** "Wrong item → `product_issue`" fixed nothing new
  and broke `t025` ("wrong address"); the wording of a rule matters.
- **Prompt iteration is measure → diagnose → fix → re-measure.** Without the
  labeled dataset and the metrics, prompt tweaks are guesswork.
