# SQL Hallucination Detector

**Can you flag wrong LLM-written SQL without knowing the right answer?**

This project builds a few detectors that never see the gold query, then scores them as honestly as a small study allows on [Spider](https://yale-lily.github.io/spider): a development set, a held-out set scored once, and two stress tests.

**Status (November 2026):** Layer 1 (parse, schema linking, execution), two cheap result-based checks, and the evaluation harness are built and scored. Layer 2 (metamorphic testing) is planned, not built.

## Findings

1. **Most "hallucinated" labels are not clear errors.** A query is labelled wrong when its result differs from Spider's gold result. I read every such query: only **7 of 41** (development set) and **11 of 76** (held-out set) are clear errors. The rest are harmless differences, ambiguous questions, or doubtful gold answers.
2. **Layer 1 had almost nothing to catch when the model saw the full schema.** None of the 549 labelled queries failed to execute, and Layer 1 flagged none of them (recall 0 of 76 on the held-out set). It also never flagged a correct query: 0 of 450 across all five eval sets (95% upper bound about 0.85%, optimistic because rows are not independent).
3. **When a table or column is hidden from the prompt, Layer 1 catches almost every structural failure.** Parse plus schema linking caught 24 of 25 failing queries (Wilson 95% interval 0.80 to 0.99; both stress sets reuse the same 100 questions). But those 25 are only about 16% of the 156 queries that differ from gold. The other 131 ran fine and returned a different answer, which Layer 1 cannot see.
4. **Two cheap checks find a useful share of real errors, with many false alarms.** On the held-out set, flagging queries that return nothing (check A) or compare against a string that is not in the database (check B) found **8 of 11** real errors (recall 0.73, 95% interval 0.43 to 0.90) at precision **0.33** (0.18 to 0.53). Read them as a "look at this one" flag, not a verdict.
5. **What they miss is the hard part.** The missed real errors all return plausible, non-empty answers: a filter silently dropped, a wrong table, an extra `WHERE`. That gap is what Layer 2 is for.

## How it works

```
Spider question + schema ──► LLM (Groq) ──► generated SQL
                                                │
                      ┌─────────────────────────┴───────────────────────┐
                      ▼                                                 ▼
        Layer 1: parse · schema linking · execution        Checks A and B on the result
                      └─────────────────────────┬───────────────────────┘
                                                ▼
                                   flag for human review

 Scoring only (never a detector input):
 gold SQL ──► compare result sets ──► labels ──► precision / recall (Wilson 95% intervals)
```

| Detector | Looks at | Flags when |
|---|---|---|
| Layer 1 · parse | the SQL text | sqlglot (SQLite dialect) cannot parse it |
| Layer 1 · schema linking | the SQL text and the real schema, no execution | a table or column does not exist. Handles aliases, CTEs, derived tables, SELECT aliases and `rowid`. Also flags an unknown double-quoted name, which SQLite would silently run as a string |
| Layer 1 · execution | runs the query read-only on the database | the database raises an error |
| Check A · degenerate result | the query's result | no rows, or one row of only 0/NULL |
| Check B · literal not in DB | the query's string literals and the whole database | a literal appears in no column, or only with different letter case (for example `'USA'` when the data stores `'usa'`). Literals containing `%` are checked as LIKE patterns |

Check B searches every column, not just the one the literal is compared with, so it under-detects.

**Gold SQL is used only to build labels and to score.** No detector reads it.

### Labels and scoring

- **Label.** A query is labelled hallucinated if it fails to execute while gold does, or if both execute and the result sets differ. Results are compared as sets of row tuples, so row order and duplicates are ignored but column order matters. Rows where the model hit its token limit, or gold itself fails, are dropped. The single rule lives in `src/relabel_eval_set.py`.
- **This label is a proxy, not proof.** Same-values-different-column-order counts as a mismatch, and so does a better answer than gold's.
- **Adjudication.** I classified every labelled-wrong query as `real_error`, `artifact` (the generated query is fine), `ambiguous`, or `gold_questionable`. The classification was made by **one rater, an AI assistant (Claude), reading the SQL and the first result rows**. I did not run alternative queries, and no second person has checked it. Treat it as a documented judgement, not a measurement. Files: `adjudication_v3.json`, `adjudication_v4.json`.
- **Policies.** `strict` uses the raw labels. `adjudicated` keeps real errors as positives, treats matches and artifacts as negatives, and leaves ambiguous and gold-questionable rows out. A third policy, `informative`, also drops correct queries whose gold answer is empty. That needs the gold answer, so it could not run in a real system, and I chose it after seeing the development-set false alarms. I do not headline it.
- **Intervals** are 95% Wilson score intervals. They assume independent rows, which Spider's paraphrased question pairs break, so read them as optimistic.

### Protocol

- Spider `dev.json` is shuffled with seed 42. Rows 0 to 149 are the **development set** (v3), used to design checks A and B. Rows 150 to 449 are the **held-out set** (v4), generated once. The detectors were frozen before v4 was scored, and v4 was adjudicated without seeing which detector flagged what.
- Rows 450 to 549 are a pilot with a smaller model (gpt-oss-20b). Rows 550 to 649 are the two stress sets.

## Results

### Label quality

| | Development (v3) | Held-out (v4) |
|---|---:|---:|
| Questions | 150 | 300 |
| Labelled hallucinated (result differs from gold) | 41 | 76 |
| of which `real_error` | **7** | **11** |
| of which `artifact` | 10 | 21 |
| of which `ambiguous` | 22 | 32 |
| of which `gold_questionable` | 2 | 12 |

Even counting every ambiguous row as an error, the totals are 29 and 43, not 41 and 76. Of v4's 21 artifacts, 20 are column-order-only differences that the scorer sets automatically.

### Layer 1

| Eval set | What the model saw | Queries scored | Labelled wrong | Flagged, of the wrong | Flagged, of the correct |
|---|---|---:|---:|---:|---:|
| v3, development | full schema | 150 | 41 | 0 of 41 | 0 of 109 |
| v4, held-out | full schema | 300 | 76 | 0 of 76 | 0 of 224 |
| Pilot, smaller model | full schema | 99 (1 cut off, dropped) | 26 | 0 of 26 | 0 of 73 |
| Stress: one **table** hidden | schema minus a table the gold query uses | 100 | 80 | 20 of 80 (0.25, 0.17 to 0.36) | 0 of 20 |
| Stress: one **column** hidden | schema minus a column the gold query uses | 100 | 76 | 5 of 76 (0.07, 0.03 to 0.14) | 0 of 24 * |

\* One extra flag appears if you run this set on an older SQLite. See Limitations.

In the stress sets, "wrong" includes queries that fail to execute (20 and 5). Those 25 are what Layer 1 can reach:

- **Parse plus schema linking caught 24 of 25** without running anything: 1 failed to parse, 18 named a missing table or column in table mode, and 5 named a missing column in column mode.
- **The one that got through both** used `split_part`, a Postgres function that does not exist in SQLite. Static schema linking does not check function names. Execution caught it.
- **The execution check catches all 25 by construction**, because "fails to execute" is also how those rows are labelled. So the informative number is the static one.
- **Schema linking never flagged a query that actually executes**, in any of the five sets. Its extra value over execution (catching an unknown double-quoted name that SQLite runs as a string) did not show up in this data.
- **Hiding something made the model guess names, but mostly it guessed wrong in ways that still run.** Typical silent substitutions: a different real column, a dropped filter, `Open_Year AS name`, `SELECT 0 AS ...`. I did not adjudicate the stress sets, so "differs from gold" there is only the strict label.

### Checks A and B (no gold)

Positives are `real_error` rows. The development numbers are not a test, because the checks were shaped on that set.

| Set | Policy | Check | Flagged | Precision | Recall |
|---|---|---|---:|---|---|
| v3 (dev) | adjudicated | A | 15 | 4/15 = 0.27 (0.11 to 0.52) | 4/7 = 0.57 (0.25 to 0.84) |
| v3 (dev) | adjudicated | B | 7 | 4/7 = 0.57 (0.25 to 0.84) | 4/7 = 0.57 (0.25 to 0.84) |
| v3 (dev) | adjudicated | A or B | 16 | 5/16 = 0.31 (0.14 to 0.56) | 5/7 = 0.71 (0.36 to 0.92) |
| **v4 (held-out)** | adjudicated | A | 24 | 8/24 = 0.33 (0.18 to 0.53) | 8/11 = 0.73 (0.43 to 0.90) |
| **v4 (held-out)** | adjudicated | B | 14 | 7/14 = 0.50 (0.27 to 0.73) | 7/11 = 0.64 (0.35 to 0.85) |
| **v4 (held-out)** | adjudicated | A or B | 24 | 8/24 = 0.33 (0.18 to 0.53) | 8/11 = 0.73 (0.43 to 0.90) |
| v4 (held-out) | strict | A or B | 27 | 11/27 = 0.41 (0.25 to 0.59) | 11/76 = 0.14 (0.08 to 0.24) |

Under strict labels recall is low simply because most strict positives are not real errors and many differ only in ways no detector could see.

**What they catch.** A literal in the wrong case or form, which makes the query return nothing: `'USA'` against stored `'usa'`, `'Yes'` against `'yes'`, `'math'` against `'Math'`, `LIKE '%Bachelors%'` against stored `'Bachelor'`. B finds these because the literal is not in the data. A finds them because the result is empty.

**What they miss.** Five real errors returned plausible rows. Examples: "US territory" read as `Name LIKE '%United States%'`; an added `WHERE LifeExpectancy IS NOT NULL` that drops countries from a population total; grouping by name instead of id, which merges two students.

**False alarms.** All 16 false alarms of A on v4 are queries whose correct answer is itself empty or zero, for example airports in a city that is not in the data. Without the gold answer, "correctly empty" and "wrongly empty" look the same.

## Limitations

- **Few real errors.** 7 and 11 real errors, about 9 independent ones in v4 because some questions are paraphrases of each other. Every interval above is wide, and the Wilson intervals are optimistic.
- **The positives favour these checks.** Real errors are the ones I could diagnose from the SQL, and a wrong literal is the easiest kind to diagnose. Recall on real errors that are harder to see is probably lower than reported.
- **One rater, no execution of alternatives.** The 22 and 32 `ambiguous` rows are excluded from the adjudicated scores, so nothing here says how the detectors do on them.
- **One dataset, one main model.** Spider dev, written by a hosted gpt-oss model via Groq. Spider may appear in the models' training data. I have not checked.
- **The stress sets are contrived.** They hide something the question needs, which a real schema retriever would only sometimes do. The same 100 questions are used in both.
- **Layer 1 schema linking has stated limits.** One flat alias namespace per query, unqualified columns checked against the union of visible tables, and no function-name check.
- **Results depend on the SQLite build.** One query in the column-hidden set uses `CONCAT()`. On the machine that produced the numbers it executed and matched gold. On SQLite 3.37.2 (a Linux re-run I used to cross-check the numbers) it fails, which adds one false flag. Everything else reproduced exactly. Check yours with `python3 -c "import sqlite3; print(sqlite3.sqlite_version)"`.
- **Regenerating will not give identical rows.** Hosted models change over time. The adjudication files are keyed by row index into my exact eval sets, and `score_detectors.py` raises an error if they do not match the labelled positives. To score a fresh run you need your own adjudication.
- **The eval sets are not in this repo.** They contain Spider questions and gold SQL, which I did not want to redistribute. The adjudication files and review printouts are included as the record of my judgements.

## What's next

**Layer 2: metamorphic testing.** Rewrite the question in a way that should not change the answer (or should change it in a known direction), regenerate the SQL, run it, and flag the query if the results disagree. The approach is from *Hallucination Detection for LLM-based Text-to-SQL Generation via Two-Stage Metamorphic Testing* ([arXiv:2512.22250](https://arxiv.org/abs/2512.22250)). I could not find public code for it, so I will reimplement a small subset, tune it on v3, freeze it, and score v4 once. The limit is known in advance: it detects instability, so a query that is wrong the same way every time will pass.

Then a small Streamlit demo.

## Repository layout

```
src/
  layer1_detector.py         parse + schema linking + execution -> one verdict
  schema_linking.py          static check for invented tables and columns
  execution_check.py         the one place SQL is executed (read-only, reads every row)
  generate_eval_set.py       Spider question -> Groq model -> SQL, labelled against gold;
                             resumable, rate-limit safe, --degrade-schema, --dry-run
  degrade_schema.py          hides one table or column from the prompt schema
  relabel_eval_set.py        the single labelling rule
  run_layer1_on_eval_set.py  Layer 1 scores, with a per-label and per-check breakdown
  diagnose_mismatches.py     why each mismatch differs; checks A and B
  score_detectors.py         Layer 1, A, B under three label policies
  show_pairs.py              gold next to generated SQL, for manual review
  evaluate.py                precision, recall, F1 helper
fixtures/                    a toy database and its builder
tests/toy_queries.py         12 hand-labelled smoke-test queries
adjudication_v3.json         my judgement for every labelled-wrong v3 row
adjudication_v4.json         the same for v4
mismatch_pairs.txt           review printout for v3
pairs_v4.txt                 review printout for v4
```

## Reproduce

Python 3 with `sqlglot`, `scikit-learn` and `groq` (developed on Python 3.14, macOS).

```bash
git clone https://github.com/ankitkkothawade/Halluciation_detector.git
cd Halluciation_detector
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Download Spider from its [project page](https://yale-lily.github.io/spider) and unzip it so that `datasets/spider_data/dev.json` and `datasets/spider_data/database/<db_id>/<db_id>.sqlite` exist. Spider is not redistributed here; see its page for its terms.

Generating needs a free [Groq](https://console.groq.com) key. Put `export GROQ_API_KEY=...` in a git-ignored `.env.local` and run `source .env.local`.

```bash
# development set (v3): shuffled rows 0-149
python3 src/generate_eval_set.py --n 150 --out eval_set_v3.jsonl

# held-out set (v4): rows 150-449, generate once
python3 src/generate_eval_set.py --n 300 --offset 150 --out eval_set_v4.jsonl

# stress sets: hide one table or one column the gold query uses (rows 550-649)
python3 src/generate_eval_set.py --n 100 --offset 550 --degrade-schema table  --out eval_set_degraded_table.jsonl
python3 src/generate_eval_set.py --n 100 --offset 550 --degrade-schema column --out eval_set_degraded_column.jsonl
```

`--dry-run` shows what would be hidden without calling the API. `--resume` continues an interrupted run. `--list-models` prints the models live on your Groq account. The default model is `openai/gpt-oss-120b` at temperature 0.

Score a set:

```bash
python3 src/run_layer1_on_eval_set.py --eval-set eval_set_v4.jsonl
python3 src/score_detectors.py --eval-set eval_set_v4.jsonl --adjudication adjudication_v4.json
```

Use Layer 1 on a single query:

```python
import sys
sys.path.insert(0, "src")
from layer1_detector import check_query

v = check_query("SELECT name, salary_amount FROM employees", "fixtures/toy_company.db")
print(v.flagged_hallucination, v.reasons)
# True ["unknown_columns: ['salary_amount']", 'execution_error: OperationalError: no such column: salary_amount']
```

## Credits

Built solo as a portfolio and validation project, with an AI assistant (Claude) helping with code and with the adjudication described above. Data: [Spider](https://yale-lily.github.io/spider) (Yu et al., EMNLP 2018). SQL parsing: [sqlglot](https://github.com/tobymao/sqlglot). Generation: [Groq](https://groq.com).
