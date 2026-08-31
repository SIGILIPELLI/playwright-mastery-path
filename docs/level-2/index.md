# Level 2 · Intermediate <span class="level-badge">Structure & Scale</span>

Goal: move from one-off scripts to a real test suite — organize page logic
with the Page Object Model, share setup/teardown with fixtures, handle the
messy realities of multi-tab and multi-frame apps, intercept and mock
network traffic, work with real file uploads/downloads, run tests in
parallel, manage configuration across environments, and debug a failing
test methodically instead of guessing.

## Modules

1. [Page Object Model](01-page-object-model.md)
2. [Fixtures & Test Hooks](02-fixtures-hooks.md)
3. [Multiple Tabs, Frames & Popups](03-tabs-frames-popups.md)
4. [Network Interception & Mocking](04-network-interception.md)
5. [File Upload & Download](05-file-upload-download.md)
6. [Parallel Execution & Sharding](06-parallel-sharding.md)
7. [Test Configuration & Projects](07-test-configuration.md)
8. [Environment & Test Data Management](08-env-test-data.md)
9. [Debugging Failing Tests](09-debugging-failing-tests.md)
10. [Project — POM Test Suite](10-project-pom-suite.md)

By the end of this level you'll be able to structure a multi-file pytest
suite with reusable page objects and fixtures, mock backend responses,
handle popups and iframes, and run the whole thing in parallel across
environments with clean, targeted debugging when something breaks.
