# Debugging Spec Files

Debug Playwright `.spec.ts` test files by combining standard Playwright test runner debug tools with `playwright-cli`.

## Running a Spec File in Debug Mode

Use Playwright's built-in debug flag to pause execution and open the Inspector:

```bash
# Debug a specific spec file
npx playwright test tests/my-feature.spec.ts --debug

# Debug a specific test by title
npx playwright test tests/my-feature.spec.ts --debug -g "should submit form"

# Debug with headed browser (no auto-pause)
npx playwright test tests/my-feature.spec.ts --headed
```

## Using PWDEBUG Environment Variable

```bash
# Open Playwright Inspector for every test
PWDEBUG=1 npx playwright test tests/my-feature.spec.ts

# Run in headed mode with slow-motion
PWDEBUG=console npx playwright test tests/my-feature.spec.ts
```

## Debugging Workflow with playwright-cli

Use `playwright-cli` to explore the page interactively first, then translate findings into your spec file:

```bash
# Step 1: Explore the page with playwright-cli to understand element structure
playwright-cli open https://example.com/login
playwright-cli snapshot
# Output shows element refs: e1 [textbox "Email"], e2 [textbox "Password"]

# Step 2: Interact and collect generated code
playwright-cli fill e1 "user@example.com"
# Ran Playwright code: await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');
playwright-cli fill e2 "secret"
playwright-cli click e3

# Step 3: Capture the current state for comparison
playwright-cli screenshot --filename=expected-state.png
playwright-cli snapshot --filename=after-login.yaml

# Step 4: Put the generated code into your spec file
playwright-cli close
```

Then debug the spec file with the verified selectors:

```typescript
import { test, expect } from '@playwright/test';

test('login flow', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');
  await page.getByRole('textbox', { name: 'Password' }).fill('secret');
  await page.getByRole('button', { name: 'Sign In' }).click();
  await expect(page).toHaveURL(/.*dashboard/);
});
```

```bash
npx playwright test tests/login.spec.ts --debug
```

## Inspecting Failures with Tracing

When a spec file fails in CI, reproduce and debug it locally using traces:

```bash
# Step 1: Re-run the failing test with trace enabled
npx playwright test tests/my-feature.spec.ts --trace on

# Step 2: Open the trace viewer
npx playwright show-report
```

Or replay the failure interactively with `playwright-cli`:

```bash
# Reproduce the failing scenario manually to inspect state
playwright-cli open https://example.com
playwright-cli tracing-start
# ... reproduce the failing steps ...
playwright-cli tracing-stop
playwright-cli close
```

## Inserting page.pause() in Tests

Add `page.pause()` directly in your spec file to pause at a specific point:

```typescript
test('debug specific step', async ({ page }) => {
  await page.goto('https://example.com');
  await page.getByRole('button', { name: 'Open Menu' }).click();
  await page.pause(); // Execution pauses here, opens Playwright Inspector
  await page.getByRole('menuitem', { name: 'Settings' }).click();
});
```

Run the test in headed mode to trigger the pause:

```bash
npx playwright test tests/my-feature.spec.ts --headed
```

## Running a Single Test File Non-Interactively

```bash
# Run with retries disabled for cleaner debug output
npx playwright test tests/my-feature.spec.ts --retries=0

# Run with verbose output
npx playwright test tests/my-feature.spec.ts --reporter=line

# Run only tests matching a pattern
npx playwright test tests/my-feature.spec.ts -g "checkout"
```
