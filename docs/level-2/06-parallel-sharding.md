# 06 · Parallel Execution & Sharding

## Why parallelize

A suite of 500 E2E tests at ~2 seconds each is nearly 17 minutes run
serially. Since most Playwright tests spend most of their wall-clock time
waiting on the browser/network rather than the CPU, running many at once
(across processes, and often across multiple CI machines) cuts wall-clock
time dramatically without needing more powerful hardware per-test.

## `pytest-xdist` for local/CI parallelism

```bash
pip install pytest-xdist
pytest -n auto
```

```text
gw0 [500] / gw1 [500] / gw2 [500] / gw3 [500]
................................................ [100%]
500 passed in 187.32s (0:03:07)
```

`-n auto` spawns one worker process per CPU core; `-n 4` pins it to a
specific count. Each worker gets its own Python interpreter and, with
`pytest-playwright`'s default fixtures, its own browser/context per test —
so tests running concurrently never share mutable state unless you've
explicitly built a fixture that does (a shared file, a shared test-account
login), which is the most common cause of "passes alone, fails with
`-n auto`."

## Test isolation requirements for safe parallelism

```python
# BAD: all workers hit the same test account and could race
def test_updates_profile_name(page):
    login_as(page, "shared_test_user@example.com")
    ...

# GOOD: each test gets a fresh, uniquely-named account or resource
import uuid

def test_updates_profile_name(page):
    email = f"test_{uuid.uuid4().hex[:8]}@example.com"
    register_new_account(page, email)
    ...
```

```text
# no output — the difference only shows up as flaky, hard-to-
# reproduce failures under -n auto, since two workers editing the
# same shared_test_user's profile concurrently silently clobber
# each other's changes
```

The rule: any two tests must be safe to run at the exact same instant,
against the same backend, without affecting each other's outcome. Shared
read-only fixtures (a public catalog page) are fine; shared *mutable*
state (one login used everywhere, one row in a database everyone edits)
is the top cause of "flaky under parallel, fine alone."

## Built-in parallelism via `playwright.config` (JS) vs. pytest workers (Python)

Playwright's JS/TS test runner parallelizes natively with a `workers`
config option; the Python `pytest-playwright` plugin instead delegates
parallelism entirely to `pytest-xdist`, since that's the general-purpose
Python test parallelism tool the ecosystem already uses. Functionally the
outcome is equivalent — N worker processes running the suite concurrently
— but Python projects reach for `-n auto` rather than a Playwright-specific
setting.

```ini
# pytest.ini
[pytest]
addopts = -n auto --dist loadscope
```

```text
# no output — addopts bakes the flag into every `pytest` invocation
# so CI and local runs stay consistent without retyping flags
```

`--dist loadscope` groups tests by module (or class) onto the same worker,
useful when a `conftest.py` fixture with `scope="module"` does expensive
one-time setup you don't want repeated per-test across workers.

## Sharding across multiple CI machines

Parallelism inside one machine has a ceiling (CPU cores, memory for N
browser instances). Sharding splits the *whole suite* across separate CI
job runners, each handling a fraction of the tests.

```yaml
# .github/workflows/e2e.yml (excerpt)
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: pytest --shard-id=${{ matrix.shard }} --num-shards=4
```

```python
# conftest.py — a minimal manual sharding implementation
def pytest_addoption(parser):
    parser.addoption("--shard-id", type=int, default=1)
    parser.addoption("--num-shards", type=int, default=1)

def pytest_collection_modifyitems(config, items):
    shard_id = config.getoption("--shard-id")
    num_shards = config.getoption("--num-shards")
    items[:] = [
        item for i, item in enumerate(items)
        if i % num_shards == (shard_id - 1)
    ]
```

```text
# no output — pytest_collection_modifyitems runs after test
# collection but before execution, so each of the 4 CI jobs only
# executes roughly a quarter of the total collected tests
```

In practice, `pytest-split` is the standard plugin for this (it also
supports duration-based balancing so shards finish around the same time,
rather than a naive modulo split where one shard happens to get all the
slow tests) — the hand-rolled version above shows the mechanism plainly.

```bash
pip install pytest-split
pytest --splits 4 --group 1     # on CI runner 1
pytest --splits 4 --group 2     # on CI runner 2
```

```text
Running group 1/4 (127 tests)
...
```

## Combining both

A common CI layout: 4 shards (matrix jobs), each shard itself running with
`-n auto` internally — sharding gives you horizontal scale across
machines, `-n auto` gives you vertical scale within each machine.

```bash
pytest --splits 4 --group ${{ matrix.shard }} -n auto
```

```text
# no output — flags compose: pytest-split selects this shard's
# subset of tests first, then pytest-xdist parallelizes that subset
```

## Exercise

1. Take an existing small suite (5-10 tests) and time a serial run with
   `pytest` alone, then with `pytest -n auto`; record both durations.
2. Deliberately introduce a shared-state bug: two tests that both log into
   the *same* hardcoded account and mutate a profile field with opposing
   values. Run with `-n 2` several times and observe intermittent failures
   that don't happen with `-n 1`.
3. Fix it using a `uuid`-suffixed unique account/resource per test, and
   confirm `-n 2` is now consistently green across several runs.
4. Install `pytest-split`, split your suite into 3 groups, and run each
   group's command separately, confirming the union of all three groups'
   test counts equals your total test count with none run twice.
