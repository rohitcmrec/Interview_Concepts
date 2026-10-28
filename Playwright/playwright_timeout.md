# Playwright Timeouts

Playwright has different timeout settings for different operations. The important thing is to understand **what is being timed**.

## 1. Timeout Types

| Timeout                | Controls                                                   | Example                          |
| ---------------------- | ---------------------------------------------------------- | -------------------------------- |
| **Test Timeout**       | Maximum time allowed for the **entire test**               | `test.setTimeout(1000)`          |
| **Action Timeout**     | Maximum time for actions such as `click()`, `fill()`, etc. | `click({ timeout: 5000 })`       |
| **Navigation Timeout** | Maximum time for page navigation/load operations           | `goto(url, { timeout: 2000 })`   |
| **Assertion Timeout**  | Maximum time for `expect()` assertions to pass             | `toBeVisible({ timeout: 5000 })` |

### Important

A timeout specified directly on an operation applies **only to that operation**.

```ts
await page.goto(url, { timeout: 2000 })       // Navigation: 2 sec

await page.getByRole('heading').click({
  timeout: 5000
})                                             // Action: 5 sec

await expect(page.locator('.heavyLoader'))
  .toBeVisible({ timeout: 5000 })              // Assertion: 5 sec
```

---

## 2. Test Timeout

Controls the **maximum total duration of the test**.

```ts
test('timeout', async ({ page }) => {
    test.setTimeout(1000)

    // Entire test must finish within 1 second
})
```

If the test exceeds this limit, Playwright fails the test even if an individual action or assertion has a larger timeout.

**Default:** `30 seconds`

---

## 3. Action Timeout

Controls how long Playwright waits for actions such as:

* `click()`
* `fill()`
* `check()`
* `selectOption()`

Example:

```ts
await page.getByRole('heading', { name: 'Email' })
    .click({ timeout: 5000 })
```

Here, Playwright can wait up to **5 seconds for the click to succeed**.

**Default:** `0` — no explicit action timeout; the operation is still subject to the overall test timeout.

### Configure globally

```ts
use: {
    actionTimeout: 10000
}
```

This gives actions a default timeout of **10 seconds**.

---

## 4. Navigation Timeout

Controls how long Playwright waits for navigation operations such as:

```ts
await page.goto(url)
await page.reload()
await page.goBack()
await page.goForward()
```

Example:

```ts
await page.goto(url, { timeout: 2000 })
```

The navigation is allowed to wait up to **2 seconds**.

### Configure globally

```ts
use: {
    navigationTimeout: 15000
}
```

This sets the navigation timeout to **15 seconds**.

---

## 5. Assertion Timeout

Controls how long Playwright's `expect()` waits for an assertion to become true.

Example:

```ts
await expect(page.locator('.heavyLoader'))
    .toBeVisible({ timeout: 5000 })
```

Playwright keeps retrying the assertion for up to **5 seconds**.

### Configure globally

```ts
expect: {
    timeout: 8000
}
```

This gives all assertions a default timeout of **8 seconds**.

---

## 6. Example `playwright.config.ts`

A typical configuration can look like this:

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
    timeout: 45000,              // Entire test: 45 sec

    expect: {
        timeout: 8000,           // Assertions: 8 sec
    },

    use: {
        actionTimeout: 10000,    // Actions: 10 sec
        navigationTimeout: 15000 // Navigation: 15 sec
    }
});
```

## 7. Easy Way to Remember

Think of the timeouts as **nested limits**:

```text
Test Timeout
│
├── Navigation Timeout
│   └── page.goto()
│
├── Action Timeout
│   └── click(), fill(), check(), etc.
│
└── Assertion Timeout
    └── expect(...)
```

* **Test Timeout** → How long can the **whole test** run?
* **Navigation Timeout** → How long can **page navigation** take?
* **Action Timeout** → How long can an **action** wait?
* **Assertion Timeout** → How long can an **assertion** retry?

### Key Point

A larger operation-level timeout does **not** override a smaller test timeout.

For example:

```ts
test.setTimeout(1000);

await expect(locator).toBeVisible({ timeout: 5000 });
```

The assertion cannot actually use the full 5 seconds if the **1-second test timeout** is reached first.
