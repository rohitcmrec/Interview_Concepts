# Playwright API Mocking — Plain English Guide

Playwright lets you intercept network requests your browser makes during a test, so you can control what the server "responds" with — without needing a real server running.

---

## 1. Fully Mocking an API Call

```ts
test('mocks a fruit and does not call api', async ({ page }) => {
  await page.route('*/**/api/v1/fruits', async (route) => {
    const json = [{ name: 'Strawberry', id: 21 }];
    await route.fulfill({ json });
  });

  await page.goto('https://demo.playwright.dev/api-mocking');
  await expect(page.getByText('Strawberry')).toBeVisible();
});
```

**What's happening:**

- `page.route(...)` intercepts any request that matches the URL pattern `*/**/api/v1/fruits`.
- Instead of hitting the real API, Playwright returns your fake JSON: `[{ name: 'Strawberry', id: 21 }]`.
- The page never actually calls the server — it just receives the mocked response.
- The test then checks that "Strawberry" appears on screen.

**When to use:** When you want complete control over the response — no network needed.

---

## 2. Intercepting and Modifying the Real Response

```ts
test('gets the json from api and adds a new fruit', async ({ page }) => {
  await page.route('*/**/api/v1/fruits', async (route) => {
    const response = await route.fetch();     // Call the real API
    const json = await response.json();       // Get the real data
    json.push({ name: 'Playwright', id: 100 }); // Add a new item
    await route.fulfill({ response, json }); // Send the modified data back
  });

  await page.goto('https://demo.playwright.dev/api-mocking');
  await expect(page.getByText('Playwright', { exact: true })).toBeVisible();
});
```

**What's happening:**

- Playwright intercepts the request, but this time it *still calls* the real API using `route.fetch()`.
- It gets back the real list of fruits, then pushes a new item (`Playwright`) into the array.
- The modified response is sent to the page — the page thinks the server returned all of it.
- The test checks that the added fruit "Playwright" appears.

**When to use:** When you want real data *plus* some extras — great for testing edge cases.

---

## 3. Mocking with HAR Files

A **HAR file** (HTTP Archive) is a recorded snapshot of real network requests and responses. Think of it like a saved "replay" of what the server said.

### Recording / Updating a HAR

```ts
test('records or updates the HAR file', async ({ page }) => {
  await page.routeFromHAR('./hars/fruits.har', {
    url: '*/**/api/v1/fruits',
    update: true,  // Hit the real server and save/overwrite the HAR
  });

  await page.goto('https://demo.playwright.dev/api-mocking');
  await expect(page.getByText('Strawberry')).toBeVisible();
});
```

- `update: true` means: go to the real server, record the response, and save it in `fruits.har`.
- Run this test when you want to refresh your saved responses.

### Playing Back a HAR

```ts
test('gets the json from HAR and checks the new fruit has been added', async ({ page }) => {
  await page.routeFromHAR('./hars/fruits.har', {
    url: '*/**/api/v1/fruits',
    update: false,  // Use the saved HAR, don't call the real server
  });

  await page.goto('https://demo.playwright.dev/api-mocking');
  await expect(page.getByText('Strawberry')).toBeVisible();
});
```

- `update: false` means: use the already-saved HAR file. No real network call is made.
- If the request doesn't match anything in the HAR, it gets aborted.

**When to use HAR files:** When you want stable, reproducible tests that don't depend on a live server — for CI pipelines or offline testing.

---

## Quick Summary

| Approach | Calls Real Server? | Best For |
|---|---|---|
| `route.fulfill(...)` | No | Fully fake responses |
| `route.fetch()` + `fulfill` | Yes | Real data + modifications |
| `routeFromHAR` (`update: true`) | Yes | Recording responses for later |
| `routeFromHAR` (`update: false`) | No | Replaying saved responses |
