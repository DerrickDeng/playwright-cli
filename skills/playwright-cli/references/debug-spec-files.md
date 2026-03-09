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

## Connecting playwright-cli to a Running Spec (Automated)

The most powerful workflow: run your spec in debug mode with a fixed CDP port, then connect `playwright-cli` directly to that browser. This lets you inspect live state, take snapshots, and run commands while the test is paused.

**Step 1: Expose a fixed CDP port in `playwright.config.ts`**

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    launchOptions: {
      args: ['--remote-debugging-port=9222'],
    },
  },
});
```

**Step 2: Run your spec in debug mode** (test will pause at first action or `page.pause()`)

```bash
npx playwright test tests/my-feature.spec.ts --debug
```

**Step 3: In another terminal, connect playwright-cli to the same browser**

```bash
PLAYWRIGHT_MCP_CDP_ENDPOINT=http://localhost:9222 playwright-cli snapshot
PLAYWRIGHT_MCP_CDP_ENDPOINT=http://localhost:9222 playwright-cli screenshot
```

Or set it once as an environment variable for the session:

```bash
export PLAYWRIGHT_MCP_CDP_ENDPOINT=http://localhost:9222

playwright-cli snapshot        # inspect live page state
playwright-cli screenshot      # capture the current frame
playwright-cli eval "document.title"
```

playwright-cli reads `PLAYWRIGHT_MCP_CDP_ENDPOINT` and connects to the browser running your spec, sharing the same page. You can inspect the DOM, take screenshots, and run eval expressions while the test is paused at a breakpoint or `page.pause()`.

---

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
