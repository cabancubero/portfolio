# Algorithm - Portfolio Breakdown

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-------------------|
| **Python** | Core language for all algorithm modules |
| **Algorithm Design** | Multi-dimensional distance-based matching system with weighted scoring |
| **Data Normalization** | Scale factor normalization, set intersection optimization, category mapping inversion |
| **Modular Architecture** | Decomposition of monolithic algorithm into single-responsibility modules |
| **Type-Annotated Python** | Function signatures with `dict`, `list`, `float`, `set` type hints throughout |
| **Set Theory / Jaccard Similarity** | Applied in action scoring for measuring overlap between user and org behavior |
| **Mathematical Notation & Formalization** | Designed formal mathematical definitions for each scoring function before writing code; learned mathematical notation to document the algorithm's logic independently of its implementation |

---

## Mathematical Foundations

The algorithm was formalized in mathematical notation before any code was written. This math-first approach forced precision in the design and served as a specification the code was built against. The current codebase is one iteration behind the latest mathematical definitions, but the core structure holds. Below are the formulas that drove the implementation.

**Scale Factor:**

```
s = mᵤ / mₒ
```

Where `mᵤ` = max user rank, `mₒ` = max org rank. Normalizes org rankings into the user's ranking space.

**Issue Score per org issue `i`:**

```
I(i) = wₑ · dₑ(i) + w_c · d_c(i)
```

Where `wₑ` = exact match weight (0.7), `w_c` = category match weight (0.3).

**Exact Distance:**

```
dₑ(i) = |U(i) - s·O(i)|    if Dₑ(Uᵢ) = Dₑ(Oᵢ)
dₑ(i) = mᵤ                  otherwise
```

The exact issue at the org must be the exact issue at the user. If not, the penalty is the max number of user options (bounded by the scale factor).

**Category Distance:**

```
d_c(i) = min { |U(j) - s·O(i)| : C(j) = C(i) }
```

Finds the user issue `j` in the same category as org issue `i` that produces the smallest distance. This is the graceful degradation mechanic - partial category alignment is measured rather than ignored.

**Action Score:**

```
A = ‖Uₐ‖ - Σ( Uₐ(i) ∩ Oₐ(i) )
```

Action score equals the total count of user-selected actions minus the sum of intersections between user and org action sets. A perfect score has 100% intersection, yielding 0.

**Value Score:**

```
dᵥ(q) = |Uᵥ(q) - Oᵥ(q)|
V = Σ dᵥ(q) / |Q|
```

Where `Q` is the list of required value questions. The distance for each question is the absolute difference between user and org responses, then the total is averaged across all questions answered.

**Final Score:**

```
D = Σ( I · wᵢ + A · wₐ + V · wᵥ )
```

The total matching distance is the weighted sum of all three dimensions. Lower `D` = better match.

---

## Architectural Decisions

### 1. Two-Tier Weighting System
The algorithm separates weights into **inner-loop** weights (how to score within a dimension) and **outer-loop** weights (how to balance dimensions against each other). This allows tuning the sensitivity of issue matching independently from tuning how much issues matter relative to actions or values.

```python
DEFAULT_WEIGHTS = {
    'exact_match': 0.7,    # Inner-loop: exact issue matching
    'category_match': 0.3, # Inner-loop: categorical matching
    'issue_weight': 0.6,   # Outer-loop: issues in total score
    'action_weight': 0.2,  # Outer-loop: actions in total score
    'value_weight': 0.2    # Outer-loop: values in total score
}
```

### 2. Distance-Based Scoring (Lower = Better Match)
All three scoring modules produce distance values where **0 is a perfect match**. This makes the scores additive and composable - the total score is simply the sum of weighted distances. It also makes the system easy to extend with new dimensions without restructuring.

### 3. Monolithic-to-Modular Refactor with Equivalence Testing
The algorithm was originally a single function. The refactor split it into `issues.py`, `actions.py`, `values.py`, and `data_normalization.py` - each with a single responsibility. A comparison test (`comparison_test.py`) runs both implementations against the same data and asserts identical output, guaranteeing the refactor preserved correctness.

---

## Code Snippets (Core Competencies)

### 1. Issue Scoring - Scaled Distance with Category Fallback
**Skills:** Algorithm design, dictionary comprehension, normalization

This is the core matching function. It scales org rankings to the user's ranking space, then calculates a weighted combination of exact-match distance and category-level distance for each org issue.

```python
def calculate_issue_score(
    user_rankings: dict,
    org_rankings: dict,
    issue_categories: dict,
) -> float:
    total_distance = 0
    user_max_rank = len(user_rankings)
    scale_factor = normalize_rankings(user_rankings, org_rankings)
    scaled_org_rankings = {
        issue: issue_rank * scale_factor
        for issue, issue_rank in org_rankings.items()
    }

    category_to_issues = build_category_mappings(issue_categories)
    common_issues = get_common_issues(user_rankings, org_rankings)

    for org_issue, org_rank_scaled in scaled_org_rankings.items():
        org_category = issue_categories.get(org_issue)

        if org_issue in common_issues:
            exact_distance = abs(user_rankings[org_issue] - org_rank_scaled)
        else:
            exact_distance = user_max_rank

        category_distance = user_max_rank
        if org_category and org_category in category_to_issues:
            best_category_distance = user_max_rank
            for user_issue in category_to_issues[org_category]:
                if user_issue in user_rankings:
                    current_distance = abs(user_rankings[user_issue] - org_rank_scaled)
                    best_category_distance = min(best_category_distance, current_distance)
            category_distance = best_category_distance

        weighted_distance = (exact_distance * DEFAULT_WEIGHTS['exact_match'] +
                            category_distance * DEFAULT_WEIGHTS['category_match'])
        total_distance += weighted_distance

    final_issue_score = (total_distance / user_max_rank) * DEFAULT_WEIGHTS['issue_weight']
    return final_issue_score
```

### 2. Action Scoring - Jaccard Similarity Inversion
**Skills:** Set operations, Jaccard index, metric inversion

Converts action lists to sets and uses Jaccard similarity to measure overlap. The `(1 - similarity)` inversion converts a similarity metric into a distance metric, keeping it consistent with the rest of the scoring system.

```python
def calculate_action_score(
    user_actions: list,
    org_actions: list,
) -> float:
    user_set = set(user_actions)
    org_set = set(org_actions)

    if user_set and org_set:
        action_intersection = len(user_set & org_set)
        action_union = len(user_set | org_set)
        action_similarity = action_intersection / action_union
        action_score = (1 - action_similarity) * len(user_set)
    else:
        action_score = len(user_set)

    final_action_score = action_score * DEFAULT_WEIGHTS['action_weight']
    return final_action_score
```

### 3. Data Normalization Utilities
**Skills:** Set intersection, dictionary inversion, edge case handling

Three utility functions that each solve a specific normalization problem. `normalize_rankings` handles the scale mismatch between user and org ranking spaces. `build_category_mappings` inverts a lookup table as a preprocessing optimization. `get_common_issues` reduces comparison space using set intersection.

```python
def get_common_issues(user_rankings: dict, org_rankings: dict) -> set:
    return set(user_rankings.keys()) & set(org_rankings.keys())

def build_category_mappings(issue_categories: dict) -> dict:
    category_to_issues = {}
    for issue, category in issue_categories.items():
        if category not in category_to_issues:
            category_to_issues[category] = []
        category_to_issues[category].append(issue)
    return category_to_issues

def normalize_rankings(user_rankings: dict, org_rankings: dict) -> float:
    user_max_rank = len(user_rankings)
    org_max_rank = len(org_rankings)
    if org_max_rank == 0:
        return 1.0
    scale_factor = user_max_rank / org_max_rank
    return scale_factor
```

### 4. Refactor Equivalence Test
**Skills:** Testing methodology, regression testing, dictionary comparison

Runs both the original monolithic algorithm and the refactored modular version against identical data, then compares every output field. This pattern guarantees behavioral equivalence across a refactor.

```python
def run_comparison():
    original_results = calculate_total_score(
        user_rankings, org_rankings, issue_categories,
        user_actions, org_actions, user_values, org_values
    )

    modular_issue_score = calculate_issue_score(
        user_rankings, org_rankings, issue_categories
    )
    modular_value_score = calculate_value_score(
        user_rankings, org_rankings, user_values, org_values, questions
    )
    modular_action_score = calculate_action_score(
        user_actions, org_actions
    )
    modular_total_score = round(
        modular_issue_score + modular_value_score + modular_action_score, 2
    )

    modular_results = {
        'issue_score': round(modular_issue_score, 2),
        'value_score': round(modular_value_score, 2),
        'action_score': round(modular_action_score, 2),
        'total_score': modular_total_score
    }

    scores_match = all(
        original_results[key] == modular_results[key]
        for key in original_results
    )
```

---

## Clever Implementations

### 1. Category-Based Graceful Degradation
The issue scoring doesn't just check for exact matches - it falls back to category-level proximity. If a user cares about "Mental Health" and an org focuses on "Depression Awareness", the algorithm recognizes both belong to "Healthcare" and finds the minimum distance across all category peers rather than assigning a maximum penalty. This means partial alignment is rewarded instead of ignored.

The key mechanic is the inner loop over `category_to_issues[org_category]`, which searches for the user's closest related issue:

```python
if org_category and org_category in category_to_issues:
    best_category_distance = user_max_rank
    for user_issue in category_to_issues[org_category]:
        if user_issue in user_rankings:
            current_distance = abs(user_rankings[user_issue] - org_rank_scaled)
            best_category_distance = min(best_category_distance, current_distance)
    category_distance = best_category_distance
```

This is then blended with the exact match at a 70/30 split, so direct alignment still dominates but categorical proximity softens the penalty for adjacent interests.

### 2. Scale-Aware Penalties Using `user_max_rank`
Rather than using a hardcoded penalty value when data is missing (e.g., an org issue the user hasn't ranked), the algorithm uses `user_max_rank` - the number of issues the user ranked. This means:

- A user who ranked 3 issues gets a max penalty of 3 for an unranked issue
- A user who ranked 10 issues gets a max penalty of 10

The penalty scales with the problem space. A missing issue in a small ranking set matters proportionally less than a missing issue in a large one. This avoids the common pitfall of magic numbers like `999` or `float('inf')` that would distort the scoring when combined with normalized values.

```python
user_max_rank = len(user_rankings)

# Used as the worst-case distance when no match exists
if org_issue in common_issues:
    exact_distance = abs(user_rankings[org_issue] - org_rank_scaled)
else:
    exact_distance = user_max_rank  # Scales with data, not a magic number
```
