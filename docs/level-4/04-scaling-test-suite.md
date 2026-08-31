# 04 · Scaling a Test Suite

A suite that takes 5 minutes at 50 tests can take an hour at 2,000 if nobody
actively manages its shape. This module covers the structural decisions that
keep a large E2E suite fast and maintainable rather than something the team
learns to dread.

## Diagnosing where time actually goes

```bash
pytest tests/ --durations=20
```

```text
======================== slowest 20 durations =========================
14.32s call     tests/test_reports.py::test_export_large_csv
9.87s  call     tests/test_checkout.py::test_full_purchase_flow
8.21s  setup    tests/test_dashboard.py::test_widget_layout
...
```

```text
# --durations surfaces the actual bottlenecks instead of
# guessing — a slow `setup` phase usually means an expensive
# fixture (a UI login, a large seed) that could be scoped
# wider (session instead of function) or replaced with an
# API-based equivalent
```

## Sharding by duration, not by count

Splitting tests into equal-sized *groups* doesn't mean equal *runtime* if a
few tests are much slower than the rest. Balance shards by measured
duration:

```python
# scripts/balance_shards.py
import json

def balance_shards(durations: dict, num_shards: int) -> list[list[str]]:
    shards = [[] for _ in range(num_shards)]
    shard_totals = [0.0] * num_shards
    for test_id, duration in sorted(durations.items(), key=lambda x: -x[1]):
        target = shard_totals.index(min(shard_totals))
        shards[target].append(test_id)
        shard_totals[target] += duration
    return shards
```

```text
# a greedy "assign the next-slowest test to the currently
# least-loaded shard" approach approximates optimal balancing
# without needing to solve bin-packing exactly — durations come
# from a previous run's --durations or JSON reporter output
# (Level 4 module 03), refreshed periodically as the suite changes
```

## Test isolation at scale: why shared fixtures become a liability

```python
# risky at scale — a module-scoped browser context shared
# across many tests accumulates state (cookies, localStorage,
# open tabs) between them
@pytest.fixture(scope="module")
def shared_page(browser):
    context = browser.new_context()
    page = context.new_page()
    yield page
    context.close()

# safer at scale — function-scoped by default; pay the context
# creation cost, but eliminate an entire class of "only fails
# when run after test X" bugs
@pytest.fixture
def page(browser):
    context = browser.new_context()
    pg = context.new_page()
    yield pg
    context.close()
```

```text
# module/session-scoped page fixtures are a common shortcut to
# "make the suite faster," but at scale the debugging cost of
# occasional order-dependent failures usually exceeds the time
# saved — reserve wider scopes for read-only, side-effect-free
# setup (like the auth state file from Level 3), not for the
# page object itself
```

## Splitting the suite by ownership, not just by file

```text
tests/
├── checkout/        # owned by Checkout team
├── search/          # owned by Search team
├── account/         # owned by Account team
└── shared/          # cross-cutting smoke tests, owned by QA/platform
```

```yaml
# CI can then run only the affected directory on a PR that
# only touched that service, falling back to the full suite
# on main or on changes to shared code
jobs:
  detect-changes:
    outputs:
      checkout: ${{ steps.filter.outputs.checkout }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            checkout:
              - 'services/checkout/**'
              - 'tests/checkout/**'

  checkout-tests:
    needs: detect-changes
    if: needs.detect-changes.outputs.checkout == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: pytest tests/checkout tests/shared
```

```text
# path-based test selection means a PR touching only the
# search service doesn't wait on the full checkout suite —
# this only works safely once ownership boundaries in tests/
# actually match the codebase's real service boundaries
```

## Enforcing suite health with a budget

```python
# conftest.py — fail the CI run itself if total suite duration regresses
import pytest

MAX_TOTAL_DURATION_SECONDS = 600

def pytest_sessionfinish(session, exitstatus):
    total = sum(getattr(item, "_duration", 0) for item in session.items)
    if total > MAX_TOTAL_DURATION_SECONDS:
        print(f"::warning::Suite duration {total:.0f}s exceeds budget {MAX_TOTAL_DURATION_SECONDS}s")
```

```text
# treating suite runtime itself as a tracked, budgeted metric —
# not just individual test correctness — is what prevents slow
# growth from becoming a boiled-frog problem nobody notices
# until the suite takes 45 minutes
```

## Exercise

1. Run `pytest --durations=20` on a real suite and identify the top 3
   slowest tests; for each, note whether the time is in `setup` or `call`
   and what that implies about the fix.
2. Change one function-scoped fixture doing an expensive one-time UI login
   to session-scoped (reusing Level 3's storage-state pattern) and measure
   the total suite time before/after.
3. Implement the greedy `balance_shards` function above against a real or
   synthetic durations dict and confirm the resulting shard totals are more
   even than a naive alphabetical split.
4. Propose (in a comment or short doc) an ownership-based directory split
   for your own suite, and sketch the path-filter CI config that would let
   a PR only run the affected team's tests.
