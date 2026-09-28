# Playwright Interview Questions & Answers

> Converted from the uploaded **Playwright_Interview_QA.pdf**.
>
> The source is an image-based PDF, so the Markdown below preserves the questions, explanations, and code examples as closely as possible to the source.

---

## 1. What is Playwright and how does it differ from Selenium?

### Answer

Playwright is a modern end-to-end testing framework developed by Microsoft that supports Chromium, Firefox, and WebKit browsers.

### Key differences from Selenium

- Auto-waits for elements to be actionable before performing actions.
- Built-in support for modern web features such as iframes, shadow DOM, and web components.
- Native support for multiple browser contexts and pages.
- Better handling of dynamic content with built-in wait mechanisms.
- Single API for all browsers without needing browser-specific drivers.

---

## 2. How do you handle multiple browser contexts in Playwright?

```typescript
import { test, chromium } from '@playwright/test';

test('multiple contexts example', async () => {
  const browser = await chromium.launch();

  // Create two independent browser contexts
  const context1 = await browser.newContext();
  const context2 = await browser.newContext();

  const page1 = await context1.newPage();
  const page2 = await context2.newPage();

  // Each context has independent cookies/storage
  await page1.goto('https://example.com');
  await page2.goto('https://example.com');

  await context1.close();
  await context2.close();

  await browser.close();
});
```

Each browser context is isolated and can have its own cookies, local storage, session state, and pages.

---

## 3. Explain Playwright's auto-waiting mechanism

### Answer

Playwright automatically waits for elements to be actionable before performing actions.

It checks that:

- Element is attached to the DOM.
- Element is visible.
- Element is stable (not animating).
- Element receives events (not obscured).
- Element is enabled for interactive actions.

This eliminates most explicit waits and reduces flakiness in tests.

---

## 4. How do you handle iframes in Playwright?

### Method 1: Using `frameLocator()`

```typescript
await page
  .frameLocator('iframe#myframe')
  .locator('button')
  .click();
```

### Method 2: Using `frame()` for named frames

```typescript
const frame = page.frame({ name: 'frameName' });
await frame.locator('button').click();
```

### Method 3: Using `contentFrame()` for iframe elements

```typescript
const iframeElement = page.locator('iframe#myframe');
const frame = await iframeElement.contentFrame();

await frame.locator('button').click();
```

---

## 5. What are locator strategies in Playwright and which is recommended?

Playwright supports multiple locator strategies:

```typescript
page.getByRole()
page.getByText()
page.getByLabel()
page.getByPlaceholder()
page.getByTestId()
page.locator()
```

### Recommended priority

**Role → Label → Placeholder → Text → TestId → CSS/XPath**

`page.locator()` with CSS/XPath is generally treated as a last resort in the source material.

---

## 6. How do you handle file uploads in Playwright?

### Method 1: Using `setInputFiles()`

```typescript
await page
  .locator('input[type="file"]')
  .setInputFiles('path/to/file.pdf');
```

### Method 2: Multiple files

```typescript
await page
  .locator('input[type="file"]')
  .setInputFiles(['file1.pdf', 'file2.pdf']);
```

### Method 3: Using the file chooser event

```typescript
const [fileChooser] = await Promise.all([
  page.waitForEvent('filechooser'),
  page.locator('button').click()
]);

await fileChooser.setFiles('path/to/file.pdf');
```

### Method 4: Buffer upload

```typescript
await page.locator('input[type="file"]').setInputFiles({
  name: 'file.txt',
  mimeType: 'text/plain',
  buffer: Buffer.from('file content')
});
```

---

## 7. Explain Page Object Model (POM) implementation in Playwright

### `pages/LoginPage.ts`

```typescript
export class LoginPage {
  constructor(private page: Page) {}

  private usernameInput = () => this.page.getByLabel('Username');
  private passwordInput = () => this.page.getByLabel('Password');
  private loginButton = () =>
    this.page.getByRole('button', { name: 'Login' });

  async login(username: string, password: string) {
    await this.usernameInput().fill(username);
    await this.passwordInput().fill(password);
    await this.loginButton().click();
  }

  async goto() {
    await this.page.goto('/login');
  }
}
```

### Test file

```typescript
test('login test', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');
});
```

---

## 8. How do you handle API testing in Playwright?

Playwright provides an API request fixture for API testing.

```typescript
import { test, expect } from '@playwright/test';

test('API testing example', async ({ request }) => {
  // GET request
  const response = await request.get(
    'https://api.example.com/users'
  );

  expect(response.ok()).toBeTruthy();

  const users = await response.json();

  // POST request
  const createResponse = await request.post(
    'https://api.example.com/users',
    {
      data: {
        name: 'John Doe',
        email: 'john@example.com'
      },
      headers: {
        Authorization: 'Bearer token123'
      }
    }
  );

  expect(createResponse.status()).toBe(201);

  // PUT/DELETE can follow a similar pattern.
});
```

---

## 9. How do you implement custom fixtures in Playwright?

### `fixtures.ts`

```typescript
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

type MyFixtures = {
  authenticatedPage: Page;
  loginPage: LoginPage;
};

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);

    await loginPage.goto();
    await loginPage.login('user@example.com', 'password123');

    await use(page);
  }
});
```

### Usage

```typescript
import { test, expect } from './fixtures';

test('dashboard visible', async ({ authenticatedPage }) => {
  await authenticatedPage.goto('/dashboard');

  await expect(
    authenticatedPage.getByRole('heading', { name: 'Dashboard' })
  ).toBeVisible();
});
```

---

## 10. How do you handle network interception and mocking?

### Mock an API response

```typescript
test('mock API response', async ({ page }) => {
  await page.route('**/api/users', route => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Mock User' }
      ])
    });
  });

  await page.goto('/users');
});
```

### Abort specific requests

```typescript
await page.route(
  '**/*.{png,jpg,jpeg}',
  route => route.abort()
);
```

### Continue with modifications

```typescript
await page.route('**/api/**', route => {
  route.continue({
    headers: {
      ...route.request().headers(),
      Authorization: 'Bearer mock-token'
    }
  });
});
```

### What it does

- Mocks API responses for consistent testing.
- Aborts specific requests such as images.
- Modifies request headers.
- Helps test different scenarios without relying on the real backend.

---

## 11. What are the different types of waits in Playwright?

### 1. Auto-wait

```typescript
await page.click('button');
```

Playwright automatically waits for the actionability conditions.

### 2. Wait for element state

```typescript
await page.locator('button').waitFor({ state: 'visible' });
await page.locator('button').waitFor({ state: 'hidden' });
```

### 3. Wait for navigation / URL

```typescript
await Promise.all([
  page.waitForURL('/dashboard'),
  page.click('a[href="/dashboard"]')
]);
```

### 4. Wait for load state

```typescript
await page.goto('https://example.com');
await page.waitForLoadState('networkidle');
```

### 5. Wait for selector

```typescript
await page.waitForSelector('.dynamic-content');
```

### 6. Wait for function / condition

```typescript
await page.waitForFunction(() => {
  return document.querySelectorAll('.items').length > 5;
});
```

### 7. Wait for timeout

```typescript
await page.waitForTimeout(3000);
```

This is generally not recommended unless there is a specific reason.

### 8. Wait for event

```typescript
await page.waitForEvent('dialog');
```

### Best practice

Prefer auto-waiting and specific `waitFor*` methods. Avoid unnecessary hard waits using `waitForTimeout()`.

---

## 12. How do you handle authentication and session storage?

### Method 1: Save and reuse authentication state

```typescript
test('login once', async ({ page }) => {
  await page.goto('/login');

  await page.fill('#username', 'user');
  await page.fill('#password', 'pass');

  await page.click('button[type="submit"]');

  // Save storage state
  await page.context().storageState({
    path: 'auth.json'
  });
});
```

### Use the saved state

```typescript
// playwright.config.ts

export default defineConfig({
  use: {
    storageState: 'auth.json'
  }
});
```

### What gets saved in `storageState`?

- Cookies
- Local storage
- Session-related authentication state
- IndexedDB support is also referenced in the source material.

### Benefits

- Login once and reuse across tests.
- Faster test execution.
- Consistent authenticated state.

### Method 2: Setup project for authentication

```typescript
export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/
    },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'auth.json'
      },
      dependencies: ['setup']
    }
  ]
});
```

The setup project performs login and saves the authentication state. Other projects use the saved authenticated state.

---

## 13. How do you perform visual regression testing?

```typescript
test('visual regression test', async ({ page }) => {
  await page.goto('https://example.com');

  // Full page screenshot comparison
  await expect(page).toHaveScreenshot('homepage.png');

  // Element screenshot comparison
  await expect(
    page.locator('.header')
  ).toHaveScreenshot('header.png');

  // With custom threshold
  await expect(page).toHaveScreenshot('page.png', {
    maxDiffPixels: 100,
    threshold: 0.2
  });

  // Mask dynamic content
  await expect(page).toHaveScreenshot({
    mask: [page.locator('.timestamp')],
    fullPage: true
  });
});
```

### Key points

- Compares current screenshots with baseline images.
- Detects unintended UI changes.
- Masks can ignore dynamic content such as timestamps.
- `maxDiffPixels` and `threshold` control tolerance.

---

## 14. How do you handle browser contexts for parallel test isolation?

Each Playwright test gets a fresh browser context automatically.

```typescript
import { test } from '@playwright/test';

test('test 1', async ({ page, context }) => {
  await page.goto('https://example.com');
});

test('test 2', async ({ page, context }) => {
  await page.goto('https://example.com');
});
```

### Custom context configuration

```typescript
test.use({
  contextOptions: {
    viewport: { width: 1920, height: 1080 },
    geolocation: {
      latitude: 40.7128,
      longitude: -74.0060
    },
    permissions: ['geolocation'],
    locale: 'en-US',
    timezoneId: 'America/New_York'
  }
});
```

### Benefits

- Each context has separate cookies, local storage, and session state.
- Ensures test isolation for parallel execution.
- Prevents test data contamination.
- Supports reliable parallel testing and independent tests.

---

## 15. How do you debug Playwright tests?

### Headed mode

```bash
npx playwright test --headed
```

### Debug mode with Inspector

```bash
npx playwright test --debug
```

### Pause execution

```typescript
test('debug test', async ({ page }) => {
  await page.goto('https://example.com');
  await page.pause();
});
```

### Slow motion

```typescript
test.use({
  launchOptions: {
    slowMo: 1000
  }
});
```

### Screenshots on failure

```typescript
use: {
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
  trace: 'on-first-retry'
}
```

### Console logs

```typescript
page.on('console', msg => {
  console.log(msg.text());
});
```

### Playwright Inspector

```bash
PWDEBUG=1 npx playwright test
```

### Trace Viewer

```bash
npx playwright show-trace trace.zip
```

---

## 16. How do you handle dynamic content and Shadow DOM?

### Shadow DOM

```typescript
// Method 1: Using a piercing selector
await page.locator('pierce=shadow-button').click();

// Method 2: Navigate shadow root explicitly
const shadowHost = page.locator('#shadow-host');

const shadowRoot = await shadowHost.evaluateHandle(
  el => el.shadowRoot
);

await shadowRoot.locator('button').click();
```

### Shadow DOM notes

- Playwright can work with Shadow DOM.
- For complex scenarios, navigate to the shadow root and continue.
- The source notes open Shadow DOM scenarios.

### Dynamic content

```typescript
// Auto-wait for element to appear
await page.locator('.dynamic-element').click();

// Wait for a specific element
await page.locator('.list-item').nth(4).waitFor();

// Wait for element to have text
await expect(page.locator('.status'))
  .toHaveText('Complete', { timeout: 10000 });
```

### Common strategy

Use locator-based strategies, conditions, and `expect()` assertions rather than fixed waits.

---

## 17. How do you implement data-driven testing?

### Method 1: Using a test-data array

```typescript
const testData = [
  { username: 'user1', password: 'pass1' },
  { username: 'user2', password: 'pass2' },
  { username: 'user3', password: 'pass3' }
];

testData.forEach(data => {
  test(`login with ${data.username}`, async ({ page }) => {
    await page.goto('/login');

    await page.fill('#username', data.username);
    await page.fill('#password', data.password);

    await page.click('button[type="submit"]');
  });
});
```

### Method 2: External JSON data

```typescript
import testCases from './testdata.json';

for (const testCase of testCases) {
  test(`test case: ${testCase.name}`, async ({ page }) => {
    // Test implementation using testCase data
  });
}
```

### Method 3: CSV data

```typescript
import csv from 'csv-parser';
import fs from 'fs';

const users: any[] = [];

fs.createReadStream('users.csv')
  .pipe(csv())
  .on('data', row => users.push(row));
```

### Possible data sources

- JSON files
- CSV / Excel files
- Databases
- APIs

### Why data-driven testing?

- Run the same test with multiple data sets.
- Increase test coverage efficiently.
- Make test data easier to add or update.
- Help find edge cases.

---

## 18. How do you handle alerts, dialogs, and popups?

### Alert

```typescript
page.on('dialog', async dialog => {
  console.log(dialog.message());
  await dialog.accept();
  // or dialog.dismiss()
});
```

### Confirm dialog

```typescript
page.on('dialog', async dialog => {
  expect(dialog.type()).toBe('confirm');
  await dialog.accept();
  // or dialog.dismiss()
});
```

### Prompt dialog

```typescript
page.on('dialog', async dialog => {
  await dialog.accept('My input text');
});
```

### New window / tab popup

```typescript
const [popup] = await Promise.all([
  page.waitForEvent('popup'),
  page.click('a[target="_blank"]')
]);

await popup.waitForLoadState();
await popup.locator('h1').textContent();
```

### Key points

- Use `page.on('dialog')` to handle alerts, confirms, and prompts.
- Always `accept()` or `dismiss()` dialogs so the test does not remain blocked.
- Use `Promise.all()` when handling a new page/tab popup.
- Wait for the popup to load before interacting with it.
- Validate popup content when required.

---

## 19. How do you implement retry logic and test stability?

### Built-in retries

```typescript
// playwright.config.ts

export default defineConfig({
  retries: process.env.CI ? 2 : 0,

  use: {
    actionTimeout: 10000,
    navigationTimeout: 30000
  },

  expect: {
    timeout: 5000
  }
});
```

### Custom retry logic

```typescript
test('with custom retry', async ({ page }) => {
  let attempts = 0;
  const maxAttempts = 3;

  while (attempts < maxAttempts) {
    try {
      await page.goto('https://flaky-site.com');
      await expect(page.locator('h1')).toBeVisible();

      break;
    } catch (error) {
      attempts++;

      if (attempts === maxAttempts) {
        throw error;
      }

      console.log(
        `Retry attempt ${attempts} failed. Retrying.`
      );

      await page.waitForTimeout(1000);
    }
  }
});
```

### Test stability practices

- Use auto-wait for elements.
- Use locator-based strategies.
- Avoid hard waits such as `waitForTimeout()`.
- Use automatic retries for assertions.
- Keep tests independent and isolated.
- Retry only for known flaky scenarios.
- Do not overuse retries because they may hide real issues.
- Fix the root cause of flakiness whenever possible.

---

## 20. What are soft assertions in Playwright?

```typescript
import { test, expect } from '@playwright/test';

test('soft assertions', async ({ page }) => {
  await page.goto('https://example.com');

  const softExpect = expect.soft;

  softExpect(page.locator('.title'))
    .toHaveText('Expected');

  softExpect(page.locator('.subtitle'))
    .toBeVisible();

  softExpect(page.locator('.badge'))
    .toHaveText('NEW');
});
```

Soft assertions allow the test to continue even when an assertion fails. Failures are collected and reported at the end.

### Flow

```text
Run all assertions
       ↓
Collect all failures
       ↓
Report all failures together
       ↓
Test continues and ends
```

---

## 21. How do you optimize Playwright test execution?

### Example configuration

```typescript
// playwright.config.ts

export default defineConfig({
  workers: process.env.CI ? 4 : undefined,

  shard: {
    total: 4,
    current: 1
  },

  fullyParallel: true,

  use: {
    trace: 'on-first-retry',
    video: 'retain-on-failure',
    waitUntil: 'domcontentloaded'
  },

  timeout: 30000,

  projects: [
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome']
      }
    }
  ]
});
```

### Other optimization techniques

```typescript
test.describe.configure({ mode: 'parallel' });

// Skip unnecessary tests
test.skip(({ browserName }) => browserName !== 'chromium');

// Use API calls instead of UI interactions for setup
test.beforeEach(async ({ request }) => {
  await request.post('/api/setup', {
    data: { user: 'test' }
  });
});
```

### Ways to optimize

- Run tests in parallel using workers.
- Shard tests across multiple machines.
- Use `fullyParallel`.
- Reduce trace/video overhead.
- Use faster navigation with `waitUntil: 'domcontentloaded'`.
- Group tests using projects.
- Use API requests for setup instead of unnecessary UI interactions.

### Benefits

- Faster test execution.
- Lower resource usage.
- Scalable CI/CD execution.
- More stable and reliable runs.

---

## 22. What is Playwright Codegen and how do you use it?

Playwright Codegen is a test generator tool that records user interactions and generates test code.

### Basic Codegen

```bash
npx playwright codegen https://example.com
```

### Specific browser

```bash
npx playwright codegen --browser=firefox https://example.com
```

### Custom viewport

```bash
npx playwright codegen \
  --viewport-size=1280,720 \
  https://example.com
```

### Device emulation

```bash
npx playwright codegen \
  --device="iPhone 12" \
  https://example.com
```

### Authentication

```bash
npx playwright codegen \
  --load-storage=auth.json \
  https://example.com
```

### Save generated test

```bash
npx playwright codegen \
  --output=tests/generated.spec.ts \
  https://example.com
```

### Specific language

```bash
npx playwright codegen \
  --target=python \
  https://example.com
```

Targets mentioned in the source include JavaScript, Python, Java, and C#.

### Other example

```bash
npx playwright codegen \
  --user-agent="Custom Bot" \
  https://example.com
```

```bash
npx playwright codegen \
  --lang=es-ES \
  https://example.com
```

### Key features

- Records clicks, fills, and navigation.
- Generates locators using best practices.
- Shows a live preview of generated code.
- Supports assertion generation.
- Can resume recording on existing tests.

---

## 23. Explain Playwright UI Mode and its features

### Launch UI Mode

```bash
npx playwright test --ui
```

### UI Mode with specific test

```bash
npx playwright test tests/login.spec.ts --ui
```

### UI Mode with headed browser

```bash
npx playwright test --ui --headed
```

### UI Mode with project

```bash
npx playwright test --ui --project=chromium
```

### UI Mode with grep

```bash
npx playwright test --ui --grep="login"
```

### Specific project

```bash
npx playwright test --project=chromium --ui
```

### UI Mode features

1. Watch Mode — automatically runs tests on file changes.
2. Time Travel — steps through test execution with DOM snapshots.
3. Pick Locator — interactive locator picker.
4. Network Tab — view network requests.
5. Console Logs — view browser console output.
6. Screenshots — view screenshots at each step.
7. Trace Viewer — integrated trace viewing.
8. Filtering — filter tests by name, status, and project.
9. Debugging — set breakpoints and step through code.

### Why use UI Mode?

- Interactive debugging experience.
- Visual test execution step-by-step.
- Faster troubleshooting.
- Useful for development and exploration.

---

## 24. How do you configure NPM scripts for Playwright in `package.json`?

### Example

```json
{
  "name": "playwright-project",
  "version": "1.0.0",
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed",
    "test:ui": "playwright test --ui",
    "test:debug": "playwright test --debug",
    "test:chrome": "playwright test --project=chromium",
    "test:firefox": "playwright test --project=firefox",
    "test:webkit": "playwright test --project=webkit",
    "test:all-browsers": "playwright test --project=chromium --project=firefox --project=webkit",
    "test:smoke": "playwright test --grep @smoke",
    "test:regression": "playwright test --grep @regression",
    "test:api": "playwright test tests/api",
    "test:e2e": "playwright test tests/e2e",
    "test:parallel": "playwright test --workers=4",
    "test:serial": "playwright test --workers=1",
    "test:shard": "playwright test --shard=1/4",
    "test:retry": "playwright test --retries=2",
    "test:reporter": "playwright test --reporter=html",
    "test:report": "playwright show-report",
    "test:trace": "playwright show-trace",
    "test:ci": "playwright test --reporter=blob",
    "test:local": "playwright test --headed --workers=1",
    "codegen": "playwright codegen",
    "codegen:auth": "playwright codegen --load-storage=auth.json",
    "install:browsers": "playwright install",
    "install:chrome": "playwright install chromium",
    "install:deps": "playwright install-deps",
    "test:update-snapshots": "playwright test --update-snapshots",
    "test:list": "playwright test --list",
    "test:specific": "playwright test tests/login.spec.ts",
    "test:grep": "playwright test --grep 'login'",
    "test:grep-invert": "playwright test --grep-invert 'slow'",
    "pretest": "npm run lint",
    "posttest": "npm run test:report"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.3"
  }
}
```

### What these scripts do

- Generate and view HTML reports.
- Generate a CI-oriented blob report.
- Run tests in headed mode locally.
- Generate tests using Codegen.
- Update and list tests.
- Run specific tests or tests matching grep patterns.
- Run tests in different browsers/projects.
- Run tests in parallel or with sharding.
- Retry failed tests.
- Run pre/post test tasks.
- Pin development dependency versions.

---

## 25. What are the essential Playwright dependencies and their purposes?

### Example dependencies

```json
{
  "devDependencies": {
    "@playwright/test": "^1.48.0",
    "typescript": "^5.3.3",
    "@types/node": "^20.10.0",
    "playwright-html-reporter": "^1.0.0",
    "allure-playwright": "^2.15.0",
    "dotenv": "^16.3.1",
    "faker": "^6.6.6",
    "axios": "^1.6.0",
    "xlsx": "^0.18.5",
    "csv-parser": "^3.0.0"
  },
  "dependencies": {}
}
```

### Dependency purposes

| Dependency | Purpose |
|---|---|
| `@playwright/test` | Core testing library, fixtures, assertions, and test runner |
| `typescript` | Type safety and better development experience |
| `@types/node` | Node.js type definitions for TypeScript |
| `playwright-html-reporter` | HTML test reports |
| `allure-playwright` | Allure reporting integration |
| `dotenv` | Environment variable management |
| `faker` | Test data generation |
| `axios` | API testing helpers |
| `xlsx` | Reading/writing Excel files |
| `csv-parser` | Reading CSV files |

### Example usage

```typescript
import { test, expect } from '@playwright/test';

import * as fs from 'fs';
import * as path from 'path';

import 'dotenv/config';

const API_URL = process.env.API_URL;
```

### Faker

```typescript
import { faker } from '@faker-js/faker';

const email = faker.internet.email();
```

### Installation commands

```bash
npm install -D @playwright/test
npx playwright install
npx playwright install-deps
npx playwright install chromium
npm install -D @playwright/test typescript @types/node
```

### Notes

- Use `-D` for development dependencies.
- `playwright install` downloads Chromium, Firefox, and WebKit.
- `install-deps` installs required OS-level dependencies.
- Install a specific browser when only one browser is needed.

---

## 26. What are the most important Playwright terminal commands?

### Test execution commands

| Command | Purpose |
|---|---|
| `npx playwright test` | Run all tests |
| `npx playwright test tests/login.spec.ts` | Run a specific test file |
| `npx playwright test --ui` | Open Playwright UI Mode |
| `npx playwright test --headed` | Run tests in headed browser mode |
| `npx playwright test --debug` | Run tests in debug mode |
| `npx playwright test --project=chromium` | Run tests in a specific project/browser |
| `npx playwright test --list` | List all tests |
| `npx playwright codegen` | Record/generate tests |
| `npx playwright show-report` | Open HTML report |
| `npx playwright show-trace trace.zip` | Open a trace |

### Utility commands

```bash
npx playwright --help
npx playwright --version
npx playwright install
npx playwright install chromium
npx playwright install-deps
```

### Useful filtering/options

```bash
npx playwright test --grep "login"
npx playwright test --workers=4
npx playwright test --reporter=html
npx playwright test --headed
```

---

## 27. How do you configure different environments using NPM scripts?

### `package.json`

```json
{
  "scripts": {
    "test:dev": "BASE_URL=https://dev.example.com playwright test",
    "test:staging": "BASE_URL=https://staging.example.com playwright test",
    "test:prod": "BASE_URL=https://example.com playwright test",

    "test:dev:ui": "BASE_URL=https://dev.example.com playwright test --ui",

    "test:staging:smoke": "BASE_URL=https://staging.example.com playwright test --grep @smoke",

    "test:ci": "playwright test --reporter=blob,html",

    "test:ci:chrome": "playwright test --project=chromium --reporter=blob",

    "test:mobile": "playwright test --config=playwright.mobile.config.ts",
    "test:desktop": "playwright test --config=playwright.desktop.config.ts",

    "test:parallel:dev": "BASE_URL=https://dev.example.com playwright test --workers=4",
    "test:serial:prod": "BASE_URL=https://example.com playwright test --workers=1"
  }
}
```

### `playwright.config.ts`

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    baseURL:
      process.env.BASE_URL || 'http://localhost:3000',

    extraHTTPHeaders: {
      Authorization:
        process.env.AUTH_TOKEN || ''
    }
  }
});
```

### `.env` examples

#### `.env.dev`

```text
BASE_URL=https://dev.example.com
API_KEY=dev-key-123
```

#### `.env.staging`

```text
BASE_URL=https://staging.example.com
API_KEY=staging-key-456
```

### Best practices

- Store environment-specific values in `.env` files.
- Do not commit sensitive data.
- Use `dotenv` to load environment variables.
- Use different configs for different devices or browsers.
- Use NPM scripts and environment variables to switch between environments.

---

## 28. How do you configure and use Playwright CLI for test filtering and execution control?

### Filter by test title

```bash
npx playwright test -g "login"
```

### Multiple grep patterns

```bash
npx playwright test --grep "login|signup"
```

### Exclude tests

```bash
npx playwright test --grep-invert "slow"
```

Equivalent short option:

```bash
npx playwright test -gv "slow"
```

### Combine grep and invert

```bash
npx playwright test \
  --grep "@smoke" \
  --grep-invert "@skip"
```

### Filter by file path

```bash
npx playwright test tests/auth/
```

```bash
npx playwright test tests/auth/login.spec.ts
```

### Filter by project

```bash
npx playwright test \
  --project=chromium \
  --project=firefox
```

### Global timeout

```bash
npx playwright test --global-timeout=3600000
```

### Test timeout

```bash
npx playwright test --timeout=30000
```

### Repeat tests

```bash
npx playwright test --repeat-each=3
```

### Forbid `only`

```bash
npx playwright test --forbid-only
```

### Pass when there are no tests

```bash
npx playwright test --pass-with-no-tests
```

### Ignore snapshots

```bash
npx playwright test --ignore-snapshots
```

### Common CLI options

| Option | What it does |
|---|---|
| `-g`, `--grep` | Run tests matching a title/description pattern |
| `--grep-invert`, `-gv` | Exclude tests matching a pattern |
| File path | Run tests in a specific file/folder |
| `--project` | Run tests in one or more projects/browsers |
| `--global-timeout` | Set maximum time for the entire test run |
| `--timeout` | Set test timeout |
| `--repeat-each` | Repeat each test N times |
| `--forbid-only` | Fail the run if `test.only` is present |
| `--pass-with-no-tests` | Exit successfully even if no tests are found |
| `--ignore-snapshots` | Ignore snapshot comparisons |

---

# Playwright Reporters

## Single reporter

```bash
npx playwright test --reporter=list
npx playwright test --reporter=line
npx playwright test --reporter=dot
npx playwright test --reporter=html
npx playwright test --reporter=json
npx playwright test --reporter=junit
```

## Multiple reporters

```bash
npx playwright test --reporter=list,html,junit
```

You can also configure reporters in `playwright.config.ts`:

```typescript
export default defineConfig({
  reporter: [
    ['list'],
    ['html', { open: 'never' }],
    ['junit', { outputFile: 'results.xml' }]
  ]
});
```

## Common built-in reporters

| Reporter | Description | Typical use |
|---|---|---|
| `list` | Detailed list of tests with status | CI logs and quick overview |
| `line` | One line per test | Compact CI output |
| `dot` | Progress dots | Long-running suites |
| `html` | Rich HTML report | Detailed local investigation |
| `json` | JSON output | Custom processing/integration |
| `junit` | JUnit XML report | CI tools such as Jenkins/GitLab |

Using multiple reporters can provide both console output and a rich report.

---

# Quick Reference

## Common Playwright concepts

- **Browser** — launches a browser instance.
- **BrowserContext** — an isolated browser session.
- **Page** — a browser tab/page.
- **Locator** — identifies elements.
- **Expect** — provides assertions.
- **Fixtures** — reusable test setup/context.
- **Storage State** — saves authentication/session state.
- **Route** — intercepts network requests.
- **Codegen** — records interactions and generates test code.
- **UI Mode** — interactive test execution and debugging.
- **Trace Viewer** — helps investigate test execution.
- **Reporter** — produces test execution results.

## Important commands

```bash
npx playwright test
npx playwright test --headed
npx playwright test --debug
npx playwright test --ui
npx playwright codegen
npx playwright show-report
npx playwright show-trace trace.zip
npx playwright install
```

---

## Source

Converted from the uploaded **Playwright Interview Questions & Answers** PDF, which contains 18 image-based pages covering 28 Playwright interview questions and related examples.
