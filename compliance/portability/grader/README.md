<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# Portability fidelity grader

`grade.py` is the scoring half of issue #576. Conformance (`validate.py` passing)
is a floor and says nothing about whether a model survived a converter round-trip;
this grader is the fidelity half.

It does not drive the converters. It takes the input document and the document a
converter returned, and diffs them element by element into the outcome vocabulary
in `../manifest.yaml`. A thin runner (modelled on `cube/tools/interop_matrix.py` in
apache/ossie-converters) drives each converter and feeds the
grader, so the measurement table becomes one command.

## Usage

```
python grade.py --input model.ossie.yaml --returned returned.ossie.yaml \
    --converter dbt [--diagnostics warnings.txt] [--json]
```

## Outcomes

PORTABLE (with a TAG_PRESERVED sub-flag), REWRITTEN, SURVIVED_VENDOR_ONLY,
PLACEHOLDER, ADDED, RELABELED, DROPPED_DIAGNOSED, DROPPED_SILENT, REJECTED.

## Design

- **Standard library for the diff.** `sqlglot` (already used by the converters
  and `validate.py`) is used when available to compare normalized expression
  trees, so a REWRITTEN that is only a requalify, an unquote or a reorder is
  marked cosmetic, while one it cannot prove equivalent is flagged for review.
  Without `sqlglot` the grader still runs and marks every rewrite as needing
  review.
- **Textual by default, semantic where it can be.** PORTABLE means the converter
  returned the same expression it was given (modulo cosmetic whitespace). True
  semantic equivalence of SQL is undecidable in general, so the grader auto-clears
  only the changes a normalized tree proves equal and leaves the rest for a human.
- **Identity beyond name.** A dropped input element and an added output element
  that carry an equivalent expression are paired as one element whose identity
  (name) was not preserved, rather than scored as a drop plus an unrelated add.
- **Diagnosed vs silent, and PLACEHOLDER, need the converter's warnings.** Pass
  them with `--diagnostics`.

## Execution check

`execute.py` sits behind the grader. For each metric the grader marks REWRITTEN,
it runs the authored and the returned expression in DuckDB on the same small
edge-case data and compares the results. A rewrite sqlglot proves equal (the
grader's "cosmetic") is skipped unless `--all-rewrites` is given.

```
python execute.py --input model.ossie.yaml --returned returned.ossie.yaml \
    [--converter dbt] [--all-rewrites] [--json]
```

- **SAME_ON_TEST_DATA**: equal results on every test group and on an empty
  table. Evidence, not proof: data the test set does not cover can still tell
  the two apart. Both sides failing on the same rows counts as agreeing.
- **DIFFERENT**: the results differ, or exactly one side fails (a failed cast,
  or more or fewer than one result row, which is how a lost aggregate shows).
  It carries a counterexample, the smallest differing group shrunk greedily
  until no row can be dropped, and both results on it. For the `ELSE 0` case in
  apache/ossie-converters #20:
  `authored gives 0, returned gives NULL on 1 row` (`amount 1, status 'other'`).
- **NOT_RUN**, with the reason: a vendor dialect, more than one dataset, a
  window function, a subquery or second statement, a parse error, an expression
  DuckDB cannot bind, both sides failing everywhere, or `duckdb` / `sqlglot` not
  installed. SURVIVED_VENDOR_ONLY metrics are listed as NOT_RUN; a REJECTED
  document gives a note.

Scope is a top-level metric over one dataset, tagged `ANSI_SQL` or
`OSSIE_SQL_2026`; `sales.amount` and bare `amount` are both resolved with
sqlglot. A table holds the physical columns the dataset's fields reference, and
each document gets a view applying its own field expressions, so calculated
fields work. Each group and the empty table run as one aggregate query per
side: the six M2 metrics take about 0.4 s once DuckDB is loaded. Expressions
must be a single expression, and DuckDB runs with external access and extension
loading off and its configuration locked. Numbers match within
`|a - b| <= 1e-9 * max(|a|, |b|, 1)` (the rule proposed in apache/ossie #632),
NULL matches NULL and NaN matches NaN; other values must have the same type, so
TRUE is not 1.

The test data is deterministic. Each column gets these non-NULL values, plus
NULL; the first is the ordinary value used while another column varies:

| Datatype | Defaults | Added from the literals of both expressions |
|----------|----------|---------------------------------------------|
| `String` | `'other'`, `''` | each literal (first), then its upper-cased and space-padded forms |
| `Integer` | 1, 0, -1, 2 | n - 1, n, n + 1 per number (sign included) and midpoints of neighbours, if whole and within BIGINT |
| `Decimal` | 1, 0, -1, 2.5, 1.234567 | the same, if it fits `DECIMAL(18, 6)` |
| `Float` | 1, 0, -1, 2.5, 1.234567 | the same |
| `Date`, `DateTime`, `Time` | two fixed values | each ISO 8601 literal, and one day (`Date`) or second either side |
| `Boolean` | TRUE, FALSE | nothing |

There is one group of two identical rows per value of each column, one mixed
group and one all-NULL group: NULLs, duplicates, a filter that matches nothing,
zero denominators, negatives and fractions. A column with no datatype is
`Decimal`.

Integer division and division by zero are engine-specific. Under DuckDB's
defaults (checked on 1.5.5) `/` always returns a double and `x / 0` is `inf`
(`0 / 0` is NaN), so `SUM(a) / COUNT(DISTINCT b)` against
`SUM(a) / NULLIF(COUNT(DISTINCT b), 0)` is DIFFERENT here, while another engine
may return NULL or raise. Read a DIFFERENT that involves `/` with that in mind.

Not covered: field-level rewrites, metrics over several datasets or over other
metrics, dataset-scoped metrics (`datasets[].metrics`), window functions,
`DateTimeTz` and `Opaque` columns, numeric literals that do not fit the column,
and fields whose names differ only in case (`a` and `A` share one column).

## Tests

`test_grade.py` has one test per outcome plus an identity check that runs the real
M0-M3 fixtures through a no-op converter and asserts every element grades
PORTABLE. `test_execute.py` covers the execution check, using the real fixtures
where they fit; its tests that execute SQL are skipped without `duckdb` and
`sqlglot`. Run with `PYTHONPATH=. pytest test_grade.py test_execute.py`.
