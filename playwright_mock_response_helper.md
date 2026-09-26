async mockResponse(
        method: string,
        urlPattern: string,
        options: { status: number; body: unknown; headers?: Record<string, string> }
    ): Promise<void> {
        const { status, body, headers } = options

        await this.page.route(urlPattern, (route) => {
            if (route.request().method().toUpperCase() === method.toUpperCase()) {
                route.fulfill({
                    status,
                    contentType: 'application/json',
                    body: JSON.stringify(body),
                    headers
                })
            } else {
                route.continue()
            }
        })

        this.registeredPatterns.push(urlPattern)
    }


# Generic Playwright API Mocking Helper --- Step-by-Step Breakdown

This code creates a generic, reusable helper function inside a test
automation framework (likely a Page Object Model class) to wrap
Playwright's `page.route()` method. It allows you to mock any API
response with just a single function call.

Here is the step-by-step breakdown of exactly how it works:

## 1. Function Declaration and Parameters

``` typescript
async mockResponse(
    method: string,
    urlPattern: string,
    options: { status: number; body: unknown; headers?: Record<string, string> }
): Promise<void>
```

**`method`:** The HTTP verb you want to target (e.g., `'GET'`, `'POST'`,
`'PUT'`).

**`urlPattern`:** The specific URL string, wildcard, or regex pattern of
the API endpoint you want to intercept (e.g.,
`'**/data/products.json'`).

**`options`:** An object containing the mock details:

-   **`status`:** The HTTP status code you want to return (e.g., `200`,
    `404`, `500`).
-   **`body`:** The fake data payload you want the API to return (can be
    an object, array, etc.).
-   **`headers`:** Optional custom HTTP headers (e.g., authorization
    tokens or custom cookies).

## 2. Destructuring the Options Object

``` typescript
const { status, body, headers } = options
```

This extracts the `status`, `body`, and `headers` variables from the
`options` object so they can be referenced cleanly in the code below
without writing `options.status` or `options.body` every time.

## 3. Registering the Interceptor with Playwright

``` typescript
await this.page.route(urlPattern, (route) => { ... })
```

`this.page.route()` tells Playwright's browser engine:

> "Keep an eye open for any network request that matches this
> `urlPattern`. When you see one, pause it and run the handler function
> inside these brackets."

## 4. Matching the HTTP Method

``` typescript
if (route.request().method().toUpperCase() === method.toUpperCase()) {
```

`urlPattern` only filters by the web address, not the action.

This line checks the actual HTTP method of the intercepted request
(`route.request().method()`), converts it to uppercase, and compares it
to the `method` argument you passed into the helper.

This ensures that if you are trying to mock a POST request to `/items`,
a GET request to the exact same URL won't accidentally get hijacked.

## 5. Fulfilling the Intercepted Request (The Fake Response)

``` typescript
route.fulfill({
    status,
    contentType: 'application/json',
    body: JSON.stringify(body),
    headers
})
```

If the URL and the HTTP method match perfectly, Playwright prevents the
request from ever leaving the browser to hit the real internet.

Instead, it short-circuits the request using `route.fulfill()`, sending
back the user's defined status, specifying that the data is an
application/json file type, converting the JavaScript body object into a
raw JSON text string, and attaching any custom headers.

## 6. Letting Unmatched Methods Pass Through

``` typescript
} else {
    route.continue()
}
```

If the URL matches but the HTTP method does not match (for example, the
page fired a GET request but you only targeted the POST method), it
enters this `else` block.

`route.continue()` tells Playwright to stop intercepting this specific
request and let it go out to the real live backend server normally.

## 7. Bookkeeping for Cleanup

``` typescript
this.registeredPatterns.push(urlPattern)
```

After setting up the routing rule, it pushes the `urlPattern` string
into a class array called `this.registeredPatterns`.

This is a best-practice tracking step. It allows the framework to easily
loop through and clean up/unregister all active mocks in an `afterEach`
hook using `this.page.unroute(pattern)` so that tests don't leak mock
behavior into each other.

------------------------------------------------------------------------

Would you like to see how to write a clean `afterEach` teardown method
that uses that `this.registeredPatterns` array to clear out the mocks
between tests?
