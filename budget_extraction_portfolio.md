# Budget Data Extraction -- Portfolio

## Context

This project extracts structured data from the City of Birmingham, Alabama's Municipal Performance Budget PDFs (FY25, FY26, FY27) and transforms them into machine-readable CSV files. The PDFs are real government documents with inconsistent formatting across fiscal years -- different account code formats, column labels, font-rendering artifacts, and page layouts. The architecture reflects that reality: each of the 12 extraction modules is self-contained and independently runnable, with per-year configuration that absorbs format differences without conditional spaghetti. A validation harness cross-checks the extracted data against financial invariants (e.g., detail rows must sum to reported totals) to catch regressions when the parsers change.

This is not a toy project or tutorial exercise. The source documents are 300+ page PDFs with real-world messiness, and the output CSVs are used downstream for budget analysis and visualization.

---

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-----------------|
| Python (data processing, CLI tooling) | All 12 extraction modules in `src/`, orchestrator `run_all.py` |
| Regex-based text parsing | Line classifiers and amount parsers across every module (e.g., `src/extract_revenues.py:37-38`, `src/extract_expenditures.py:192-194`) |
| Declarative architecture / separation of concerns | Each financial module follows a 7-section structure: Vocabulary, Rules, Schemas, Helpers, Classifier, Engine, Main |
| PDF data extraction (pdfplumber) | Engine functions in every module (e.g., `src/extract_revenues.py:172-218`) |
| pandas (DataFrame construction and post-processing) | Output assembly and fixups in `src/extract_expenditures.py:472-474`, `src/extract_appropriations.py:201-207` |
| Dataclass-driven configuration | `YearConfig` in `src/extract_expenditures.py:25-33` parameterizes per-year differences |
| Automated validation / regression testing | `scripts/validate_outputs.py` -- 6 cross-checks per fiscal year with tolerance-aware assertions |
| CLI design (argparse, subprocess orchestration) | `run_all.py` runs 12 modules, passes flags, reports failures; `src/parse_year.py` provides flexible input parsing |
| State machine text processing | `PageContext` dataclasses track document position across pages (e.g., `src/extract_positions.py:22-35`) |

---

## Architectural Decisions

### 1. Declarative Extraction Pattern -- Separating Rules from Engine

**Problem:** Each budget section has different structural markers, skip rules, column layouts, and output schemas. Mixing detection logic with parsing logic in a single loop creates modules that are hard to read, debug, and modify.

**Decision:** Every financial extraction module follows a consistent 7-section layout where rules, schemas, and helpers are declared separately from the engine that applies them. The engine is a generic loop; the behavior comes from the declarations above it.

**Why not a shared base class or plugin system?** Each module's rules are different enough that a shared abstraction would need so many hooks and overrides that it wouldn't simplify anything. Keeping modules self-contained means you can read one file top-to-bottom and understand it completely, without chasing inheritance chains.

```python
# Rules are data, not control flow
SKIP_LINE_RULES = [
    lambda line: not line,
    lambda line: bool(RE_PAGE_NUMBER.match(line)),
    lambda line: line == "REVENUE CATEGORIES",
    lambda line: "FY2025" in line and "FY2026" in line,
    ...
]

# The engine just applies them
if any(rule(stripped) for rule in SKIP_LINE_RULES):
    continue
```

### 2. Per-Year Configuration via Dataclasses

**Problem:** Fiscal years FY25, FY26, and FY27 use different account code formats (e.g., `123-456` vs `123456`), different column labels, different category names, and even different PDF rendering artifacts. The extraction logic is the same, but the parameters change.

**Decision:** Year-specific differences are captured in `YearConfig` dataclasses. The engine is parameterized by config, not by `if year == 2025` branches. New fiscal years require adding a config block, not modifying control flow.

**Why dataclasses over dicts or YAML?** Dataclasses give type hints and IDE autocomplete, catch typos at definition time, and keep config co-located with the code that consumes it -- no file I/O or deserialization layer needed.

```python
@dataclass
class YearConfig:
    year: int
    column_gate_keywords: tuple[str, ...]
    amount_columns: list[str]
    skip_phrases: list[str]
    account_regex: str              # FY25: r"^(\d{3}-\d{3})\s+(.+)"
    known_categories: list[str]     # FY27: 27 categories; FY25: 16
    normalize_amounts: bool         # True for FY25/26 (font artifacts)

YEAR_CONFIGS = {2025: FY25_CONFIG, 2026: FY26_CONFIG, 2027: FY27_CONFIG}
```

### 3. Validation as a Separate Regression Harness

**Problem:** PDF extraction is inherently fragile -- a parser change that fixes one department can silently break another. You need a way to know whether the overall output is still correct after every change.

**Decision:** A standalone validation script (`scripts/validate_outputs.py`) implements 6 financial invariants (e.g., non-total rows must sum to the grand total row; revenue must equal appropriations in the budget year). It reads only the output CSVs, not the source PDFs, so it runs in seconds and can be used as a CI gate.

**Why not inline assertions in each extractor?** The extractors don't know what the "right" answer is -- they parse what the PDF gives them. Correctness checks belong at a higher level, comparing across files (expenditures vs. appropriations) and across aggregation levels (detail vs. summary totals).

```python
ROUNDING_TOLERANCE = 5     # $5 per cell -- PDF source has rounding
GRAND_TOTAL_TOLERANCE = 20 # rounding chains accrue across many rows

def _check_grand_rollup(path, grand_title, label):
    details = [r for r in rows if not _is_total(r)]
    for col in money_cols:
        detail_sum = sum(_money(r, col) for r in details)
        grand = _money(grand_total_row, col)
        if not _within_tolerance(detail_sum, grand, tol=GRAND_TOTAL_TOLERANCE):
            discrepancies.append(...)
```

---

## Code Snippets (Core Competencies)

### 1. Right-to-Left Amount Extraction

**Skills:** Regex, string parsing, defensive programming

This function separates the text label from the trailing dollar amounts on a budget line. It walks tokens from right to left, collecting up to 3 numeric values, and returns the remaining text. This is the core parsing primitive used by 4 of the 6 financial extractors.

```python
# src/extract_revenues.py:114-137

def split_amounts_from_line(line: str) -> tuple[str, list[float | None]]:
    """Split a line into text portion and up to 3 trailing dollar amounts."""
    tokens = line.split()
    amounts = []
    i = len(tokens) - 1
    while i >= 0 and len(amounts) < 3:
        token = tokens[i]
        cleaned = (token.replace("$", "").replace(",", "")
                   .replace("(", "").replace(")", "")
                   .replace("-", "").replace(".", ""))
        if cleaned.isdigit() or token in ("0", "-"):
            amounts.insert(0, parse_amount(token))
            i -= 1
        elif token.startswith(("$", "-$", "($", "(-")):
            val = parse_amount(token)
            if val is not None:
                amounts.insert(0, val)
                i -= 1
            else:
                break
        else:
            break
    text_part = " ".join(tokens[: i + 1])
    return text_part, amounts
```

### 2. Line Classification with Enum-Based Dispatch

**Skills:** Enum design, functional decomposition, state machine input

Each line in the PDF is classified into a type (SKIP, SECTION_HEADER, DATA_ROW, etc.) before the engine processes it. This separates "what kind of line is this?" from "what do I do with it?", making both halves easier to test and reason about independently.

```python
# src/extract_revenues.py:154-167

def classify_line(line: str) -> tuple[LineType, str, list[float | None]]:
    """Classify a line and return its type along with parsed text and amounts."""
    if any(rule(line) for rule in SKIP_LINE_RULES):
        return LineType.SKIP, "", []

    text_part, amounts = split_amounts_from_line(line)

    if not amounts:
        return LineType.SECTION_HEADER, line, []

    if len(amounts) < MIN_AMOUNTS_REQUIRED:
        return LineType.SKIP, "", []

    return LineType.DATA_ROW, text_part, amounts
```

### 3. Tolerance-Aware Cross-File Validation

**Skills:** Data validation, numerical tolerance, regression testing

This check compares the total revenue against total appropriations using the budget-year column. It dynamically detects which column is the "proposed" column (last money column in the file), handles missing rows gracefully, and applies a rounding tolerance that accounts for the PDF source rounding individual line items to whole dollars.

```python
# scripts/validate_outputs.py:167-216

def check_budget_balance(year: str) -> CheckResult:
    rev = _read_csv(_output_dir(year) / f"revenues_{year}.csv")
    apr = _read_csv(_output_dir(year) / f"appropriations_{year}.csv")

    rev_total = next((r for r in rev if r["title"].strip().upper() == "TOTAL REVENUE"), None)
    apr_total = next(
        (r for r in apr if r["title"].strip().upper() == "TOTAL APPROPRIATIONS"), None
    )
    if rev_total is None or apr_total is None:
        return CheckResult(
            name=f"budget balance [{year}]", passed=False,
            discrepancies=["could not locate grand total row(s)"],
        )

    # Use the last money column of each file as the budget-year-proposed figure
    rev_cols = _money_columns(rev)
    apr_cols = _money_columns(apr)
    r = _money(rev_total, rev_cols[-1])
    a = _money(apr_total, apr_cols[-1])
    if not _within_tolerance(r, a):
        discrepancies.append(
            f"revenue [{rev_cols[-1]}] {r:,.0f} vs appropriations [{apr_cols[-1]}] {a:,.0f} "
            f"(delta {r - a:+,.0f})"
        )
    ...
```

### 4. Schema-Driven Row Construction

**Skills:** Dictionary unpacking, factory pattern, data modeling

The expenditures engine uses a schema registry that maps each document section to a build function and a predicate. The engine doesn't know whether it's building a summary row or a detail row -- it looks up the schema for the current section and calls it. Adding a new output format means adding one entry to the registry.

```python
# src/extract_expenditures.py:216-250

def _build_summary_row(ctx, text_part, amounts, config):
    is_total = text_part.startswith(("TOTAL", "GRAND TOTAL")) if text_part else True
    return {
        "department": ctx.department,
        "function":   ctx.function,
        "category":   text_part or "TOTAL",
        "is_total":   is_total,
        **_build_amounts(amounts, config),
    }

SECTION_SCHEMAS = {
    Section.SUMMARY: {
        "build":    _build_summary_row,
        "requires": lambda text_part, amounts: bool(amounts),
    },
    Section.DETAIL: {
        "build":    _build_detail_row,
        "requires": lambda text_part, amounts: bool(amounts),
    },
}

# In the engine loop:
schema = SECTION_SCHEMAS[section]
if schema["requires"](text_part, amounts):
    results[section].append(schema["build"](ctx, text_part, amounts, config))
```

### 5. Resilient Orchestration with Fault Isolation

**Skills:** subprocess management, CLI design, error reporting

The orchestrator runs all 12 extraction modules as separate subprocesses so that a failure in one module (e.g., a missing section in a particular fiscal year) doesn't prevent the other 11 from completing. It validates the `--year` flag once at the top and passes it through to each module, then reports a summary of successes and failures.

```python
# run_all.py:44-81

def main():
    pdf_path = sys.argv[1] if len(sys.argv) > 1 else "budget_files/FY27 - MPB FINAL.pdf"

    # Validate --year early so all 12 modules get a consistent value
    year_arg = None
    for i, arg in enumerate(sys.argv):
        if arg == "--year" and i + 1 < len(sys.argv):
            try:
                year_int = parse_year(sys.argv[i + 1])
                year_arg = str(year_int)
            except Exception:
                print(f"Error: invalid --year value '{sys.argv[i + 1]}'")
                sys.exit(1)

    failed = []
    for module in MODULES:
        script = SRC_DIR / f"{module}.py"
        cmd = [sys.executable, str(script), pdf_path]
        if year_arg and module in YEAR_AWARE_MODULES:
            cmd.extend(["--year", year_arg])
        result = subprocess.run(cmd, capture_output=False)
        if result.returncode != 0:
            failed.append(module)

    print(f"Complete: {len(MODULES) - len(failed)}/{len(MODULES)} succeeded")
    if failed:
        sys.exit(1)
```

---

## Clever Implementations

### 1. Longest-Match Category Splitting

**Skills:** Greedy parsing, disambiguation strategy

The expenditures detail section has lines like `123456 Salaries and Wages Overtime Premium Pay 50,000 48,000 52,000` where the category name and description run together with no delimiter. Naive splitting (e.g., take the first two words) fails because category names vary from one word ("Overtime") to four words ("Pensions - Fringe Cost"). The solution: sort the known categories by length descending and try each as a prefix match. The longest match wins, which prevents "R & M" from stealing a match that should go to "R & M - Buildings".

```python
# src/extract_expenditures.py:43-76, 320-328

# Categories sorted longest-first at config definition time
known_categories=sorted([
    "Salaries and Wages",
    "Pensions - Fringe Cost",
    "R & M - Buildings",
    "R & M - Equipment",
    ...
], key=len, reverse=True),

# At parse time: first prefix match wins (longest-first guarantees correctness)
def split_category_description(remainder: str, config: YearConfig) -> tuple[str, str]:
    """Split the text after account number into (category, description)."""
    for cat in config.known_categories:
        if remainder.startswith(cat):
            desc = remainder[len(cat):].strip()
            if desc:
                return cat, desc
            return cat, cat
    return "", remainder
```

This is a textbook greedy algorithm applied to a real parsing problem. The alternative -- regex alternation or tokenized parsing -- would be more complex and harder to maintain as categories change across fiscal years.

### 2. Pending Amounts Buffer for Wrapped PDF Lines

**Skills:** Stateful parsing, PDF artifact handling

Some PDF pages wrap a single data row across two lines: the first line contains only amounts, and the second line contains the account code, description, and remaining amounts. A naive line-by-line parser would discard the orphaned amounts or misattribute them. The solution is a `pending_amounts` buffer: when the parser encounters a line with amounts but no text, it stashes the amounts and prepends them to the next line's amounts.

```python
# src/extract_expenditures.py:459-470

# Handle wrapped lines: numbers-only continuation lines
# appear before the account line they belong to
if not text_part and amounts and len(amounts) < len(config.amount_columns):
    pending_amounts = amounts
    continue

if pending_amounts:
    amounts = pending_amounts + amounts
    pending_amounts = []

if schema["requires"](text_part, amounts):
    results[section].append(schema["build"](ctx, text_part, amounts, config))
```

The guard `len(amounts) < len(config.amount_columns)` prevents a full data row (which happens to have an empty text part, like a TOTAL row) from being mistakenly stashed. The buffer is reset on every non-expenditure page and when context changes, preventing stale amounts from leaking across departments.
