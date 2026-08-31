# 04 · CI Integration (GitHub Actions)

A test suite that only runs on someone's laptop protects nobody. This module
wires a Playwright/pytest suite into GitHub Actions so every pull request
runs the full suite automatically, with browsers cached, artifacts uploaded
on failure, and results visible without anyone needing local setup.

## A minimal workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install -r requirements.txt

      - run: playwright install --with-deps chromium

      - run: pytest tests/ --junitxml=results.xml
```

```text
# `playwright install --with-deps` installs both the browser
# binaries AND the OS-level shared libraries (fonts, codecs,
# etc.) Chromium needs to run headless on a bare Ubuntu runner —
# skipping --with-deps is the #1 cause of "works locally, fails
# in CI" for a fresh CI setup
```

## Caching browser binaries between runs

Downloading Chromium/Firefox/WebKit on every run wastes minutes. Cache the
Playwright browser directory keyed on the installed version:

```yaml
      - name: Get Playwright version
        id: pw-version
        run: echo "version=$(pip show playwright | grep Version | cut -d' ' -f2)" >> "$GITHUB_OUTPUT"

      - uses: actions/cache@v4
        id: playwright-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ steps.pw-version.outputs.version }}

      - run: playwright install --with-deps chromium
        if: steps.playwright-cache.outputs.cache-hit != 'true'

      - run: playwright install-deps chromium
        if: steps.playwright-cache.outputs.cache-hit == 'true'
```

```text
# on a cache hit, the browser binary itself is restored from
# cache (skipping the download), but install-deps still runs
# because the cache only holds ~/.cache/ms-playwright, not the
# apt-installed OS libraries the runner image may not persist
```

## Uploading traces and screenshots on failure

The whole point of tracing (Level 2/4) is worthless if the trace file
disappears when the CI job ends. Upload it as a build artifact:

```yaml
      - run: pytest tests/ --tracing=retain-on-failure --screenshot=only-on-failure

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces
          path: test-results/
          retention-days: 7
```

```text
# `if: failure()` means this step only runs when a prior step
# failed, so passing runs don't waste storage — a teammate can
# download the artifact zip and run
# `playwright show-trace trace.zip` locally, seeing exactly
# what the CI browser saw
```

## Sharding across parallel jobs

For suites too large for one runner, split the test IDs across a matrix and
merge results, rather than accepting a 40-minute serial run:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && playwright install --with-deps chromium
      - run: pytest tests/ --shard=${{ matrix.shard }}/4 --junitxml=results-${{ matrix.shard }}.xml
      - uses: actions/upload-artifact@v4
        with:
          name: results-${{ matrix.shard }}
          path: results-${{ matrix.shard }}.xml
```

```text
# `--shard=N/M` is provided by pytest-playwright: each of the
# 4 matrix jobs runs a disjoint quarter of the collected tests
# in parallel, cutting a 40-minute suite to roughly 10 minutes
# wall-clock (fail-fast: false so one shard failing doesn't
# cancel the others mid-run)
```

## Required status check on pull requests

```yaml
# repository settings, not YAML — but the practical effect:
# Settings > Branches > Branch protection rule for `main`
#   -> Require status checks to pass before merging
#   -> select the "test" job from e2e.yml
```

```text
# once configured, GitHub blocks the "Merge" button on any PR
# until this workflow's job reports success — turning "please
# run the tests before merging" into something enforced rather
# than a norm people can forget
```

## Full worked example: complete workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - uses: actions/cache@v4
        id: pw-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
      - run: playwright install --with-deps chromium
        if: steps.pw-cache.outputs.cache-hit != 'true'
      - run: playwright install-deps chromium
        if: steps.pw-cache.outputs.cache-hit == 'true'
      - run: >
          pytest tests/ --shard=${{ matrix.shard }}/2
          --tracing=retain-on-failure
          --junitxml=results-${{ matrix.shard }}.xml
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: traces-shard-${{ matrix.shard }}
          path: test-results/
```

## Exercise

1. Add a GitHub Actions workflow to a real (or sample) repository that
   installs dependencies, installs Chromium with `--with-deps`, and runs
   `pytest`.
2. Add browser-binary caching keyed on the Playwright version and confirm a
   second run shows a cache hit in the Actions log.
3. Add `--tracing=retain-on-failure` and an `upload-artifact` step gated on
   `if: failure()`; deliberately break a test, push, and download the
   resulting trace artifact to open locally with `playwright show-trace`.
4. Split the suite across a 2-job shard matrix using `--shard=N/M` and
   confirm both shards together still run every test exactly once (compare
   total test counts before and after sharding).
