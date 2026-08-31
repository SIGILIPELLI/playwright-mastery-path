# 02 · Component Testing

Full E2E tests are slow and only exercise a component through whatever paths
the rest of the app happens to expose. Playwright's component testing runners
(`@playwright/experimental-ct-react`, `-vue`, `-svelte`) mount a single
component in a real browser, in isolation, letting you test its props and
states directly. Note: component testing is JS/TS-only — this module covers
it because Python E2E suites in a full-stack team frequently sit alongside a
JS/TS component-test layer, and understanding the split matters for Level 4
architecture decisions even when you personally write the Python side.

## Why component tests exist alongside E2E, not instead of it

```text
Unit test:        pure function, no DOM at all — fastest, narrowest
Component test:   one component, real browser DOM, isolated from the app —
                   fast, catches rendering/interaction bugs directly
E2E test:         the whole app, real backend or a realistic fake —
                   slowest, but the only tier that catches integration bugs
                   between systems (routing, auth, real API contracts)
```

A component test can't tell you if a component wired into the real page
works — only that it works given the exact props you passed it. That's a
feature: it isolates the failure to the component itself when the assertion
fails, rather than to "somewhere in the checkout flow."

## Setup (React example)

```bash
npm init playwright@latest -- --ct
```

```ts
// playwright-ct.config.ts
import { defineConfig, devices } from "@playwright/experimental-ct-react";

export default defineConfig({
  testDir: "./src",
  snapshotDir: "./__snapshots__",
  use: { trace: "on-first-retry" },
  projects: [{ name: "chromium", use: devices["Desktop Chrome"] }],
});
```

## A basic component test

```tsx
// Button.ct.tsx
import { test, expect } from "@playwright/experimental-ct-react";
import { Button } from "./Button";

test("renders label and fires onClick", async ({ mount }) => {
  let clicked = false;
  const component = await mount(
    <Button label="Save" onClick={() => (clicked = true)} />
  );
  await expect(component).toHaveText("Save");
  await component.click();
  expect(clicked).toBe(true);
});
```

```text
# mount() renders the component into a real, isolated browser
# page — not jsdom, an actual Chromium instance — so real CSS,
# real event dispatch, and real layout all apply, unlike a
# pure unit test with a simulated DOM
```

## Testing every visual state of a component directly

```tsx
test("disabled button does not fire onClick", async ({ mount }) => {
  let clicked = false;
  const component = await mount(
    <Button label="Save" disabled onClick={() => (clicked = true)} />
  );
  await component.click({ force: true }); // bypass actionability to prove it's truly inert
  expect(clicked).toBe(false);
});

test("loading state shows a spinner instead of the label", async ({ mount }) => {
  const component = await mount(<Button label="Save" loading />);
  await expect(component.getByRole("status")).toBeVisible();
  await expect(component).not.toContainText("Save");
});
```

```text
# hitting every prop combination (disabled, loading, error)
# directly is far cheaper here than finding an E2E scenario
# that happens to put the real app into each of those states —
# some states (a specific error prop) might not even be
# reachable through the live app at all
```

## Mocking props that would otherwise require a real backend

```tsx
test("data table renders rows from props, no API involved", async ({ mount }) => {
  const rows = [
    { id: 1, name: "Widget", price: 9.99 },
    { id: 2, name: "Gadget", price: 19.99 },
  ];
  const component = await mount(<DataTable rows={rows} />);
  await expect(component.getByRole("row")).toHaveCount(3); // 2 + header
  await expect(component.getByText("$19.99")).toBeVisible();
});
```

## When to still write the E2E test anyway

```text
Skip E2E, keep only component test:
  - pure presentational components (a Badge, a Tooltip, a Card)
  - components whose only inputs are props, no routing/context/API

Keep both:
  - a form component: component test covers validation states,
    E2E covers "this form's submit actually persists to the backend
    and the resulting page reflects it"
  - anything wired into routing or global app state
```

## Full worked example: form component test suite

```tsx
// LoginForm.ct.tsx
import { test, expect } from "@playwright/experimental-ct-react";
import { LoginForm } from "./LoginForm";

test("shows validation error for invalid email", async ({ mount }) => {
  const component = await mount(<LoginForm onSubmit={() => {}} />);
  await component.getByLabel("Email").fill("not-an-email");
  await component.getByRole("button", { name: "Sign in" }).click();
  await expect(component.getByText("Enter a valid email")).toBeVisible();
});

test("calls onSubmit with form values when valid", async ({ mount }) => {
  let submitted: any = null;
  const component = await mount(
    <LoginForm onSubmit={(values) => (submitted = values)} />
  );
  await component.getByLabel("Email").fill("user@example.com");
  await component.getByLabel("Password").fill("hunter2");
  await component.getByRole("button", { name: "Sign in" }).click();
  expect(submitted).toEqual({ email: "user@example.com", password: "hunter2" });
});
```

## Exercise

1. Scaffold `@playwright/experimental-ct-react` (or the Vue/Svelte
   equivalent your stack uses) in a sample project and mount a single
   simple component.
2. Write component tests covering at least three prop-driven states of one
   component (default, disabled, loading, or error) without touching a real
   backend.
3. Write a test that passes a mock `onSubmit`/`onChange` callback as a prop
   and asserts it was called with the expected arguments.
4. Pick one component currently only covered by an E2E test and write a
   component test for its validation logic; note in a comment which parts
   of the original E2E test remain necessary anyway (e.g. real persistence)
   and which are now redundant.
