# 02 · Installation & Project Setup

## Creating the project

```bash
mkdir playwright-practice && cd playwright-practice
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
```

## Installing Playwright

There are two separate installs, and skipping the second one is the single
most common "why doesn't this work" moment for newcomers:

```bash
pip install pytest-playwright
playwright install
```

```text
Collecting pytest-playwright
  ...
Successfully installed playwright-1.4x.0 pytest-playwright-0.5.x

Downloading Chromium 1xx.x (playwright build vXXXX) ...
Downloading Firefox 1xx.x (playwright build vXXXX) ...
Downloading Webkit 1x.x (playwright build vXXXX) ...
Playwright Host validation warnings:
...
```

`pip install` gets you the Python **library** — the API you import and call.
`playwright install` downloads the actual **browser binaries** Playwright
drives, into a local cache (`~/.cache/ms-playwright` on Linux/macOS,
`%USERPROFILE%\AppData\Local\ms-playwright` on Windows). Those binaries are
versioned and tested together with the Playwright release you installed —
this is why Playwright doesn't just reuse whatever Chrome you already have
on your machine: it guarantees the automation protocol and the browser build
are compatible.

If you only need one engine (common in CI to save time):

```bash
playwright install chromium
```

### System dependencies (Linux)

On a fresh Linux machine (including most CI images) the browsers also need
OS-level shared libraries:

```bash
playwright install --with-deps chromium
```

This runs the platform's package manager under the hood to install what's
missing. macOS and Windows generally don't need this step.

## Verifying the install

```bash
playwright --version
```

```text
Version 1.4x.0
```

```bash
python3 -c "from playwright.sync_api import sync_playwright; \
p = sync_playwright().start(); b = p.chromium.launch(); \
print('OK:', b.version); b.close(); p.stop()"
```

```text
OK: 1xx.0.xxxx.xx
```

If that prints `OK` and a Chromium version, the library and the binary are
both correctly installed and can talk to each other.

## Project layout

A conventional `pytest-playwright` project looks like this:

```text
playwright-practice/
├── .venv/
├── pytest.ini
├── conftest.py
├── requirements.txt
└── tests/
    ├── test_homepage.py
    └── test_login.py
```

`requirements.txt`:

```text
pytest-playwright==0.5.*
```

Pin it — a Playwright minor version bump can ship browser binary updates
that change rendering just enough to break a visual regression suite
(Level 3), so reproducible installs matter more here than in most Python
projects.

`pytest.ini`:

```ini
[pytest]
testpaths = tests
addopts = --browser chromium
```

`--browser` is a flag `pytest-playwright` adds on top of plain pytest. It
controls which engine the `page` fixture launches (Module 3 introduces that
fixture). Without it, Chromium is the default anyway, but being explicit in
`pytest.ini` means the whole team runs the same engine by default and only
overrides it deliberately.

## Editor tooling (optional but worth it)

```bash
playwright codegen https://example.com
```

This opens a real browser window alongside an "Inspector" window. Every
click, type, and navigation you perform in the browser is recorded live as
Playwright Python code in the Inspector. You won't ship generated code
as-is — Module 4 explains why hand-picked locators beat generated ones — but
`codegen` is genuinely useful for two things: discovering what locator
Playwright would pick for an element you're pointing at, and quickly
prototyping a flow before you clean it up by hand.

```bash
playwright codegen --target python-pytest https://example.com
```

```text
# pass --target python-pytest to get pytest-style output directly,
# instead of the default sync_api script format
```

## Updating

```bash
pip install --upgrade pytest-playwright
playwright install
```

Always re-run `playwright install` after upgrading the library — an updated
Python package expects updated browser binaries, and running mismatched
versions is a common source of confusing failures that have nothing to do
with your test code.

## Exercise

1. Create the `playwright-practice` project exactly as shown above, with a
   virtual environment.
2. Run `pip install pytest-playwright` and then `playwright install
   chromium firefox` (skip webkit for now to save disk space/time).
3. Run the verification one-liner for both Chromium and Firefox (change
   `p.chromium` to `p.firefox`) and confirm both print a version.
4. Create `pytest.ini` with `testpaths = tests` and `addopts = --browser
   chromium`.
5. Run `playwright codegen https://example.com`, click around the page for a
   few seconds, then close both windows and look at the code it generated in
   your terminal. Note anything it picked that looks like a CSS class or
   auto-generated ID — you'll revisit why that's fragile in Module 4.
