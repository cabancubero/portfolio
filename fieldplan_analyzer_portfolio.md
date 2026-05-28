## Context

This is a **production application** used by Alabama Forward, a civic engagement organization, to process field plan and budget submissions from partner organizations across the state. It runs entirely on Google Apps Script (V8 runtime) with no external dependencies, no build step, and no transpilation — deployed via `clasp push`. The architecture reflects the constraints and advantages of that environment: no `import`/`require`, all files share a global scope, and the Google Sheets API is the primary data layer. Despite these constraints, the codebase applies object-oriented design, configuration-driven patterns, and functional programming idioms that would be at home in any modern JavaScript project.

The system processes ~50+ organization submissions per cycle, generates BigQuery SQL for voter targeting, performs cost-efficiency analysis against statistical benchmarks, and delivers branded HTML email reports to field staff — all automated via time-based triggers.

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-----------------|
| Object-Oriented JavaScript (ES6 classes, inheritance) | Three-tier class hierarchy: `FieldPlan` → `FieldProgram` → `TacticProgram` across `field_plan_parent_class.js`, `field_program_extension_class.js`, `field_tactics_extension_class.js` |
| Configuration-Driven Architecture | `TACTIC_CONFIG` object in `field_tactics_extension_class.js` replaces 7 separate classes with one parameterized class + config |
| SQL Generation & BigQuery Integration | `query_sql_templates.js` builds parameterized MERGE and SELECT queries; `query_builder.js` orchestrates the full pipeline |
| Strategy Pattern (Prioritized Matching) | `resolveVanId()` and `resolvePrecinctCode()` in `query_resolvers.js` use ordered strategy arrays with `.reduce()` to find the best match |
| Functional Programming (map/filter/reduce) | Used throughout — aggregations in `analyzeTacticFlags()`, range merging in `mapAgeDemographics()`, budget calculations in `expectedBudgetRange()` |
| Data Validation & Quality Flagging | `analyzeTacticFlags()` in `field_tactics_extension_class.js` runs 4 validation checks and returns structured, prioritized flags |
| Event-Driven Automation | Trigger system in `field_trigger_functions.js` — time-based polling, installable `onEdit` for reprocessing, and cross-system event callbacks |
| Centralized Schema Management | `_column_mappings.js` is the single source of truth for 68+ spreadsheet columns with validation functions that detect duplicates and overlaps |

## Architectural Decisions

### 1. Configuration-Driven Tactics Over Subclass Proliferation

**Problem:** The system needed to support 7 different field tactics (phone banking, door canvassing, text banking, etc.), each with different cost targets, contact rates, and reasonableness thresholds. The original implementation used 7 separate classes (~210 lines) with duplicated logic.

**Decision:** Replace all tactic subclasses with a single `TacticProgram` class that reads its behavior from a `TACTIC_CONFIG` object. Adding or disabling a tactic becomes a data change, not a code change.

**Why this over alternatives:** Inheritance (one subclass per tactic) would have been the textbook OOP approach, but the tactics share identical logic — only their parameters differ. A strategy pattern with separate strategy objects was considered, but since the parameters are static, a plain configuration object is simpler and more readable. This approach also lets non-developers (field staff) understand what each tactic expects by reading the config.

```javascript
const TACTIC_CONFIG = {
  PHONE: {
    enabled: true,
    name: 'Phone Banking',
    columnKey: 'PHONE',
    contactRange: [0.05, 0.10],
    reasonableThreshold: 30,
    costTarget: 0.66,
    costStdDev: 0.15
  },
  RELATIONAL: {
    enabled: false,   // disabled in April 2026 — no code deletion required
    name: 'Relational Organizing',
    columnKey: 'RELATIONAL',
    contactRange: [0.20, 0.30],
    reasonableThreshold: 50,
    costTarget: 0.50,
    costStdDev: 0.15
  },
  // ... 5 more tactics
};
```

### 2. Centralized Column Mappings as Single Source of Truth

**Problem:** The application reads from spreadsheets with 68+ columns that change when the JotForm is updated. Column indices were scattered across class constructors, trigger functions, and analysis code — a recipe for silent bugs when columns shift.

**Decision:** Extract every column index into `_column_mappings.js` as named constants (`FIELD_PLAN_COLUMNS`, `BUDGET_COLUMNS`, `PROGRAM_COLUMNS`), with validation functions that detect duplicates and cross-group overlaps.

**Why this over alternatives:** Magic numbers in constructors are fragile and hard to audit. A database schema migration approach doesn't apply to Google Sheets. Mapping objects with validation strike the right balance: one file to update when the spreadsheet changes, and `validateColumnMappings()` catches errors before they reach production.

```javascript
const PROGRAM_COLUMNS = {
  PHONE: {
    PROGRAMLENGTH: 37,
    WEEKLYVOLUNTEERS: 38,
    WEEKLYHOURS: 39,
    HOURLYATTEMPTS: 40
  },
  DOOR: {
    PROGRAMLENGTH: 41,
    WEEKLYVOLUNTEERS: 42,
    WEEKLYHOURS: 43,
    HOURLYATTEMPTS: 44
  },
  // ... more tactics
};
```

### 3. Resolver Functions with Prioritized Matching Strategies

**Problem:** User-submitted form data is messy — organization names have inconsistent punctuation, precinct inputs range from bare numbers to full location names with typos, and county names vary in casing. The system needs to match these against canonical lookup tables without false positives.

**Decision:** Each resolver function (`resolveVanId`, `resolvePrecinctCode`) defines an ordered array of matching strategies, from strictest to most lenient. A `.reduce()` walk returns the first match, and the match type is recorded in the result for auditability.

**Why this over alternatives:** A single fuzzy-match algorithm would either be too loose (false positives) or too strict (missed matches). The layered approach lets strict matches take priority while falling back to progressively looser strategies. Recording the `matchType` in the return value (`'exact'`, `'normalized'`, `'fuzzy'`, `'word_overlap'`) lets downstream code and email reports flag uncertain matches for human review.

```javascript
const strategies = [
  { type: 'exact', test: e => e.name.toLowerCase() === orgNameLower },
  { type: 'normalized', test: e => normalizeOrgName(e.name) === orgNameNormalized },
  { type: 'contains', test: e =>
      orgNameLower.includes(e.name.toLowerCase())
      || e.name.toLowerCase().includes(orgNameLower)
  }
];

const result = strategies.reduce((found, strategy) => {
  if (found) return found;
  const match = entries.find(strategy.test);
  return match
    ? { found: true, committeeId: match.id, committeeName: match.name, matchType: strategy.type }
    : null;
}, null);
```

## Code Snippets (Core Competencies)

### 1. Constructor Validation with Closure-Scoped Helper

**Skills:** Input validation, closures, fail-fast error handling, ES6 classes

The `FieldProgram` constructor validates all four numeric inputs at construction time using a closure-scoped helper function. Invalid data throws typed errors (`TypeError` for non-numbers, `RangeError` for non-positive values) with descriptive messages that include the column index and field name — making debugging straightforward when a form submission has unexpected data.

```javascript
class FieldProgram extends FieldPlan {
  constructor(rowData, tacticType) {
    super(rowData);

    const columns = PROGRAM_COLUMNS[tacticType];
    if (!columns) {
      throw new Error(`Invalid tactic type: ${tacticType}`);
    }

    const validateColumn = (columnIndex, fieldName) => {
      const value = rowData[columnIndex];
      if (typeof value !== 'number' || isNaN(value)) {
        throw new TypeError(
          `Invalid data in column ${columnIndex}: ${fieldName} must be a valid number. Got: ${value}`
        );
      }
      if (value <= 0) {
        throw new RangeError(
          `Invalid data in column ${columnIndex}: ${fieldName} must be greater than 0. Got: ${value}`
        );
      }
      return value;
    };

    this._programLength = validateColumn(columns.PROGRAMLENGTH, 'Program Length');
    this._weeklyVolunteers = validateColumn(columns.WEEKLYVOLUNTEERS, 'Weekly Volunteers');
    this._weeklyHours = validateColumn(columns.WEEKLYHOURS, 'Weekly Hours');
    this._hourlyAttempts = validateColumn(columns.HOURLYATTEMPTS, 'Hourly Attempts');
  }
}
```

### 2. Age Range Merging with Reduce

**Skills:** Functional programming, array reduction, SQL generation, interval merging algorithm

This function converts user-selected age ranges (like "18-19", "20-29", "30-39") into optimized SQL `BETWEEN` clauses by merging contiguous ranges. If a user selects ages 18-19, 20-29, and 30-39, instead of generating three separate `BETWEEN` clauses, the reducer merges them into a single `p.age BETWEEN 18 AND 39`. The function also short-circuits when all ranges are selected (no filter needed) and tracks unmapped values as SQL comments for query reviewers.

```javascript
function mapAgeDemographics(demoAgeArray) {
  if (!demoAgeArray || !Array.isArray(demoAgeArray) || demoAgeArray.length === 0) {
    return { hasFilter: false, ranges: [], sqlFragment: '', unmapped: [], unmappedComment: '' };
  }

  if (demoAgeArray.length >= AGE_RANGE_TOTAL_OPTIONS) {
    return { hasFilter: false, ranges: [], sqlFragment: '', unmapped: [], unmappedComment: '' };
  }

  const unmapped = [];
  const parsedRanges = demoAgeArray
    .map(v => {
      const formValue = v.toString().trim();
      const range = AGE_RANGE_MAP[formValue];
      if (!range) unmapped.push(formValue);
      return range;
    })
    .filter(range => range)
    .sort((a, b) => a.min - b.min);

  const merged = parsedRanges.reduce((acc, range) => {
    const last = acc[acc.length - 1];
    if (last && range.min <= last.max + 1) {
      last.max = Math.max(last.max, range.max);
    } else {
      acc.push({ min: range.min, max: range.max });
    }
    return acc;
  }, []);

  const clauses = merged.map(r => 'p.age BETWEEN ' + r.min + ' AND ' + r.max);
  const sqlFragment = clauses.length === 1
    ? clauses[0]
    : '(' + clauses.join(' OR ') + ')';

  return { hasFilter: true, ranges: merged, sqlFragment, unmapped,
    unmappedComment: unmapped.length > 0 ? `--Unmapped age selections: ${unmapped.join(', ')}` : '' };
}
```

### 3. Cost Analysis with Statistical Bounds

**Skills:** Domain modeling, statistical analysis, structured return values

Each tactic has a cost target and standard deviation derived from historical data. This method calculates cost-per-attempt from a funding amount and classifies it as `below` (underfunded), `within` (acceptable), or `above` (overfunded) relative to the target ± one standard deviation. The structured return value lets downstream code (email builders, gap analysis) render the result however they need without re-calculating.

```javascript
analyzeCost(fundingAmount) {
  const programAttempts = this.programAttempts();
  const costPerAttempt = programAttempts > 0 ? fundingAmount / programAttempts : 0;

  const lowerBound = this._costTarget - this._costStdDev;
  const upperBound = this._costTarget + this._costStdDev;

  const status = costPerAttempt <= lowerBound ? 'below' :
                 costPerAttempt >= upperBound ? 'above' : 'within';

  return {
    tacticName: this._name,
    programAttempts: programAttempts,
    fundingAmount: fundingAmount,
    costPerAttempt: costPerAttempt,
    targetCost: this._costTarget,
    lowerBound: lowerBound,
    upperBound: upperBound,
    status: status
  };
}
```

### 4. Incomplete Submission Detection with Tri-State Categorization

**Skills:** Data quality engineering, defensive programming, augmented return values

This function inspects each tactic's four input fields and categorizes them into three states: fully empty (skipped), fully filled (valid), or partially filled (incomplete — likely a user error). Rather than rejecting incomplete submissions, it attaches structured metadata to the return array so downstream code can surface specific missing fields in email reports. The `.incomplete` and `.noTacticsAtAll` properties augment a standard array without changing its iteration behavior.

```javascript
function getTacticInstances(rowData) {
  const tactics = [];
  const incomplete = [];
  let anyDataFound = false;

  for (const [tacticKey, config] of Object.entries(TACTIC_CONFIG)) {
    if (!config.enabled) continue;

    const columns = PROGRAM_COLUMNS[config.columnKey];
    if (!columns) continue;

    const fieldChecks = [
      { name: 'Program Length',    value: rowData[columns.PROGRAMLENGTH] },
      { name: 'Weekly Volunteers', value: rowData[columns.WEEKLYVOLUNTEERS] },
      { name: 'Weekly Hours',      value: rowData[columns.WEEKLYHOURS] },
      { name: 'Hourly Attempts',   value: rowData[columns.HOURLYATTEMPTS] }
    ];

    const filled = fieldChecks.filter(f => f.value && !isNaN(f.value) && Number(f.value) > 0);
    const missing = fieldChecks.filter(f => !f.value || isNaN(f.value) || Number(f.value) <= 0);

    if (filled.length === 0) continue;
    anyDataFound = true;

    if (missing.length === 0) {
      tactics.push(new TacticProgram(rowData, tacticKey));
      continue;
    }

    incomplete.push({
      tacticName: config.name,
      tacticKey: tacticKey,
      filledFields: filled.map(f => f.name),
      missingFields: missing.map(f => f.name)
    });
  }

  tactics.incomplete = incomplete;
  tactics.noTacticsAtAll = !anyDataFound;
  return tactics;
}
```

## Clever Implementations

### 1. Multi-Phase Precinct Resolution with Word-Overlap Matching

Users enter precinct identifiers in wildly different formats: bare numbers ("182"), full location names ("Dothan Civic Center"), partial names with numbers ("Houston Precinct 10"), or non-precinct geographic references ("Congressional District 3"). This resolver handles all of them through a four-phase pipeline where each phase is progressively more lenient, and a regex gate at the top rejects inputs that aren't precincts at all.

The word-overlap matching (Phase 2) is the non-obvious part. Simple substring matching produces false positives — "Houston" would match "Houston East" and "Houston West" simultaneously. Instead, the resolver tokenizes both the input and each precinct name, filters out stop words (common words like "community", "center", "precinct" that don't disambiguate), and counts overlapping tokens. A match requires 2+ overlapping non-stop words for multi-token inputs, or a unique single-token match. This handles truncated inputs ("Dothan Civic" matching "DOTHAN CIVIC CENTER 241") while rejecting ambiguous ones.

```javascript
const NON_PRECINCT_PATTERNS = /\b(congressional|district|city council|ward|senate|house)\b/i;

function resolvePrecinctCode(fieldPlanPrecinct, countyName, preloadedPrecinctMap, preloadedNameMap) {
  const rawValue = fieldPlanPrecinct.toString().trim();

  // Phase 0 — Reject non-precinct geographic references
  if (NON_PRECINCT_PATTERNS.test(rawValue)) {
    return { valid: false, precinctCode: '', rawValue, matchType: 'not_precinct' };
  }

  const validPrecincts = getCountyPrecincts(countyName, preloadedPrecinctMap);
  const isNumericOnly = /^\s*\d+\s*$/.test(rawValue);

  // Phase 1 — Numeric input: pad to 5 digits, exact match, then fuzzy (distance ≤ 2)
  if (isNumericOnly) {
    const padded = rawValue.replace(/[^0-9]/g, '').padStart(5, '0');
    if (validPrecincts.includes(padded)) {
      return { valid: true, precinctCode: padded, rawValue, matchType: 'exact' };
    }
    const inputNum = parseInt(rawValue, 10);
    const fuzzyMatch = validPrecincts
      .map(code => ({ code, distance: Math.abs(inputNum - parseInt(code, 10)) }))
      .filter(entry => !isNaN(entry.distance) && entry.distance <= 2)
      .reduce((best, entry) => (!best || entry.distance < best.distance) ? entry : best, null);
    if (fuzzyMatch) {
      return { valid: true, precinctCode: fuzzyMatch.code, rawValue, matchType: 'fuzzy' };
    }
  }

  // Phase 2 — Name-based: exact name, substring, then word-overlap
  const precinctNames = getCountyPrecinctNames(countyName, preloadedNameMap);
  const PRECINCT_STOP_WORDS = new Set([
    'at', 'the', 'of', 'in', 'and', 'for', 'voting', 'center', 'precinct', 'community'
  ]);
  const inputTokens = rawValue.toUpperCase().split(/\s+/)
    .filter(w => w.length >= 3 && !PRECINCT_STOP_WORDS.has(w.toLowerCase()));

  if (inputTokens.length >= 1) {
    const overlapResults = [];
    for (const [code, name] of precinctNames) {
      const nameTokens = name.toUpperCase().split(/\s+/);
      const matches = inputTokens.filter(inputWord =>
        nameTokens.some(nameWord => nameWord.startsWith(inputWord))
      );
      if (matches.length >= 1) overlapResults.push({ code, name, score: matches.length });
    }
    const strongMatches = overlapResults.filter(r => r.score >= 2);
    if (strongMatches.length === 1) {
      return { valid: true, precinctCode: strongMatches[0].code, rawValue, matchType: 'name_match' };
    }
  }

  // Phase 3 — Trailing-number extraction (fallback)
  const trailingMatch = rawValue.match(/(\d+)\s*$/);
  if (trailingMatch) {
    const padded = trailingMatch[1].padStart(5, '0');
    if (validPrecincts.includes(padded)) {
      return { valid: true, precinctCode: padded, rawValue, matchType: 'exact' };
    }
  }

  return { valid: false, precinctCode: '', rawValue, matchType: 'not_found' };
}
```

### 2. Cross-Tactic Validation Flags with Pairwise Comparison

When staff submit a field plan, it's common for them to copy-paste values across tactics or set unrealistic goals. Rather than blocking submission, `analyzeTacticFlags` runs four validation checks across all tactics and returns structured flags with priority levels. The pairwise comparison (Check 2) is the interesting piece: it compares every combination of tactics on their four input values, flagging pairs that share 3 or 4 values as likely copy-paste errors. This uses a nested loop (necessary for pairwise comparison) combined with `.filter()` to count matching fields, then builds a human-readable description that names the differing field when 3 of 4 match.

The function also computes aggregate statistics (total volunteers, FTE equivalent, program-wide attempts) using `.reduce()` and returns them alongside flags — giving email builders everything they need in a single call without re-traversing the tactics array.

```javascript
function analyzeTacticFlags(fieldPlan, tactics) {
  const flags = [];
  if (!tactics || tactics.length === 0) return { flags, aggregates: {} };

  const inputKeys = ['programLength', 'weeklyVolunteers', 'weeklyVolunteerHours', 'hourlyAttempts'];

  // Pairwise comparison: flag identical or near-identical tactic inputs
  for (let i = 0; i < tactics.length; i++) {
    for (let j = i + 1; j < tactics.length; j++) {
      const a = tactics[i];
      const b = tactics[j];
      const matches = inputKeys.filter(key => a[key] === b[key]);

      if (matches.length === 4) {
        flags.push({
          priority: 'medium',
          type: 'identical_inputs',
          title: `Review — Identical Inputs: ${a.tacticName} & ${b.tacticName}`,
          description: `${a.tacticName} and ${b.tacticName} have identical values for all 4 inputs. ` +
            `Verify the organization set realistic goals for each tactic individually.`,
          tactics: [a.tacticName, b.tacticName]
        });
      } else if (matches.length === 3) {
        const diffKey = inputKeys.find(key => a[key] !== b[key]);
        const labels = {
          programLength: 'Program Length', weeklyVolunteers: 'Weekly Volunteers',
          weeklyVolunteerHours: 'Hours/Week', hourlyAttempts: 'Attempts/Hour'
        };
        flags.push({
          priority: 'medium',
          type: 'similar_inputs',
          title: `Review — Near-Identical Inputs: ${a.tacticName} & ${b.tacticName}`,
          description: `Only ${labels[diffKey]} differs (${a[diffKey]} vs ${b[diffKey]}).`,
          tactics: [a.tacticName, b.tacticName]
        });
      }
    }
  }

  // Aggregates via reduce — computed once, used by email builders
  const aggregates = {
    totalWeeklyVolunteers: tactics.reduce((sum, t) => sum + t.weeklyVolunteers, 0),
    totalWeeklyVolunteerHours: tactics.reduce((sum, t) => sum + t.weekVolunteerHours(), 0),
    totalProgramAttempts: tactics.reduce((sum, t) => sum + t.programAttempts(), 0),
    fteEquivalent: (tactics.reduce((sum, t) => sum + t.weekVolunteerHours(), 0) / 40).toFixed(1)
  };

  return { flags, aggregates };
}
```
