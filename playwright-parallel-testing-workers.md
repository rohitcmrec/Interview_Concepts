# Playwright Parallel Testing: Workers and `fullyParallel`

## 1. Overview

Parallel testing in Playwright is achieved by running multiple independent **OS and Node.js processes**, called **workers**, simultaneously.

This allows different parts of the test suite to execute at the same time without sharing memory.

---

## 2. What Is a Playwright Worker?

A Playwright worker is a completely separate, independent **Node.js/OS process** managed by the Playwright Test Runner.

### Key characteristics

- **Complete Isolation:** Each worker has its own memory heap. Variables, states, or modifications made in one worker cannot leak into another.
- **Dedicated Browser Instance:** Every worker spins up and controls its own browser instance.
- **No Direct Communication:** Workers run independently and cannot directly communicate or exchange messages during execution.

---

## 3. How Parallelism Works

Playwright distributes the test workload across workers using a task-scheduling model.

The important concept is:

> **Workers are the execution units, while Playwright's scheduler decides which test work each worker receives.**

The way tests are distributed depends on the `fullyParallel` setting.

---

# 4. Default Distribution — `fullyParallel: false`

By default, Playwright treats **test files (`.spec.ts`)** as the unit of work.

For example:

```text
login.spec.ts
  ├── Test 1
  ├── Test 2
  └── Test 3

checkout.spec.ts
  ├── Test 4
  ├── Test 5
  └── Test 6
```

With:

```typescript
workers: 3,
fullyParallel: false
```

Playwright can distribute files like this:

```text
Worker 1 → login.spec.ts
Worker 2 → checkout.spec.ts
Worker 3 → another.spec.ts
```

### Important behavior

- A worker receives an entire test file.
- Tests inside that file run sequentially.
- When a worker finishes its assigned file, it picks up the next available file.
- Up to `3` files can execute concurrently when `workers: 3`.

### Simple mental model

```text
                 Test Files
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Worker 1   Worker 2   Worker 3
          |          |          |
          v          v          v
       File A     File B     File C
       T1 → T2    T4 → T5    T7 → T8
          ↓          ↓          ↓
       sequential sequential sequential
```

---

# 5. Fully Parallel Mode — `fullyParallel: true`

When you enable:

```typescript
fullyParallel: true
```

Playwright changes the distribution model.

Instead of treating the **file** as the unit of parallel work, individual **tests** can be distributed across workers.

For example:

```text
login.spec.ts
  ├── T1
  ├── T2
  └── T3

checkout.spec.ts
  ├── T4
  ├── T5
  └── T6
```

Playwright can distribute individual tests across workers:

```text
Worker 1 → T1
Worker 2 → T2
Worker 3 → T3
```

Then, as workers become available:

```text
Worker 2 → T4
Worker 1 → T5
Worker 3 → T6
```

---

# 6. `workers: 3` + `fullyParallel: true`

This is the most important scenario to understand.

Assume we have:

```text
File 1: login.spec.ts

T1
T2
T3

File 2: checkout.spec.ts

T4
T5
T6
```

Total:

```text
6 tests
```

Configuration:

```typescript
export default defineConfig({
  workers: 3,
  fullyParallel: true,
});
```

Conceptually, Playwright now has individual test work available to execute concurrently.

```text
Global Test Queue

[T1] [T2] [T3] [T4] [T5] [T6]
  |    |    |
  v    v    v
+----+ +----+ +----+
| W1 | | W2 | | W3 |
+----+ +----+ +----+
```

## Round 1 — Initial Execution

The first three available tests are assigned:

```text
Worker 1 → T1
Worker 2 → T2
Worker 3 → T3
```

All three execute concurrently.

```text
W1: T1 ────────────────>
W2: T2 ───────>
W3: T3 ───────────────>
```

---

## Round 2 — Dynamic Takeover

Suppose `T2` finishes first.

Worker 2 does **not** wait for Worker 1 or Worker 3.

Instead:

```text
Worker 2 → T4
```

while the other workers continue:

```text
W1: T1 ───────────────────>
W2: T2 ──> T4 ────────────>
W3: T3 ───────────────>
```

If `T4` finishes next:

```text
Worker 2 → T5
```

This continues until the queue is empty.

---

# 7. Important Point: `workers: 3` Does NOT Mean 3 Fixed Tests

A common misunderstanding is:

> "`workers: 3` means Worker 1 always runs Test 1, Worker 2 always runs Test 2, and Worker 3 always runs Test 3."

That is **not** how the scheduling model should be understood.

Instead:

> `workers: 3` means Playwright can have **up to 3 worker processes executing test work concurrently**.

The scheduler dynamically assigns available work to workers.

For example:

```text
Initial:

W1 → T1
W2 → T2
W3 → T3


T2 finishes:

W1 → T1
W2 → T4
W3 → T3


T1 finishes:

W1 → T5
W2 → T4
W3 → T3
```

The exact order depends on scheduling and test execution.

---

# 8. Worker Isolation

Each worker is an independent process.

Conceptually:

```text
             Playwright Test Runner
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Worker 1   Worker 2   Worker 3
          |          |          |
          v          v          v
       Browser    Browser    Browser
          |          |          |
       Memory A   Memory B   Memory C
```

Therefore:

```text
Worker 1 memory ≠ Worker 2 memory ≠ Worker 3 memory
```

A variable created inside Worker 1 cannot simply be accessed by Worker 2.

---

# 9. Important Catch — `beforeAll` / `afterAll`

This is one of the most important differences when using `fullyParallel: true`.

When tests from the same file are distributed across different workers, the workers do not share the same process or setup state.

Therefore, `beforeAll` / `afterAll` behavior needs careful consideration.

For example:

```typescript
test.beforeAll(async () => {
  // setup
});

test('Test 1', async () => {});
test('Test 2', async () => {});
test('Test 3', async () => {});
```

With fully parallel execution, tests from the same file can be executed by different workers.

Conceptually:

```text
Worker 1
  ├── beforeAll
  └── Test 1

Worker 2
  ├── beforeAll
  └── Test 2

Worker 3
  ├── beforeAll
  └── Test 3
```

The important lesson is:

> Do not assume that `beforeAll` state created in one worker is available to another worker.

Each worker has its own process and setup context.

---

# 10. Test Isolation Requirement

With fully parallel execution, tests should ideally be **independent and isolated**.

### Bad example

```text
Test 1
  ↓
Creates user "Rohit"

Test 2
  ↓
Uses the same user "Rohit"

Test 3
  ↓
Deletes user "Rohit"
```

If these tests execute concurrently:

```text
Worker 1 → Create user
Worker 2 → Use user
Worker 3 → Delete user
```

the tests can interfere with each other.

Possible result:

```text
Test 2 → FAIL
```

because Worker 3 may delete the user while Worker 2 is using it.

### Better approach

Give each test independent data:

```text
Worker 1 → User A
Worker 2 → User B
Worker 3 → User C
```

Now the tests do not interfere.

---

# 11. Worker Count Configuration

You can configure the number of workers in `playwright.config.ts`.

## Configuration file

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  workers: 3,
});
```

Or combine it with fully parallel execution:

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  workers: 3,
  fullyParallel: true,
});
```

## Command line

You can also override the worker count from the CLI:

```bash
npx playwright test --workers=3
```

---

# 12. `fullyParallel: false` vs `true`

| Feature | `fullyParallel: false` | `fullyParallel: true` |
|---|---|---|
| Distribution unit | Test file | Individual test |
| Tests in same file | Sequential | Can run concurrently |
| Multiple workers | Different files can run concurrently | Individual tests can run concurrently |
| Isolation requirement | Important | Very important |
| Potential test-data collisions | Lower | Higher if tests share state |
| `beforeAll` / `afterAll` | File-level execution model | May execute separately in different workers |
| Best suited for | Suites where file-level isolation is sufficient | Highly independent/stateless tests |

---

# 13. Example: 6 Tests + 3 Workers

Suppose:

```text
login.spec.ts
  T1
  T2
  T3

checkout.spec.ts
  T4
  T5
  T6
```

Configuration:

```typescript
workers: 3,
fullyParallel: true
```

A simplified execution model:

```text
                Test Queue
     +-----------------------------+
     | T1 | T2 | T3 | T4 | T5 | T6 |
     +-----------------------------+
        |    |    |
        v    v    v
       +----+----+----+
       | W1 | W2 | W3 |
       +----+----+----+

Initial:
W1 → T1
W2 → T2
W3 → T3

Suppose T2 finishes:

W1 → T1
W2 → T4
W3 → T3

Suppose T1 finishes:

W1 → T5
W2 → T4
W3 → T3

Eventually:

W1 → T6
W2 → available
W3 → available
```

The exact ordering is scheduler-dependent; the key concept is that **available workers pull available test work until the suite is complete**.

---

# 14. `workers` vs `fullyParallel`

These two settings answer different questions.

### `workers`

Answers:

> **How many worker processes can execute concurrently?**

Example:

```typescript
workers: 3
```

Means:

```text
Maximum concurrent workers = 3
```

### `fullyParallel`

Answers:

> **Can Playwright distribute individual tests across workers, including tests from the same file?**

Example:

```typescript
fullyParallel: true
```

Means individual tests can become units of parallel execution.

---

# 15. Easy Way to Remember

Think of a restaurant kitchen.

### `workers`

Number of chefs:

```text
workers: 3

Chef 1
Chef 2
Chef 3
```

### `fullyParallel: false`

Each chef gets a **whole order/file** and works through its items sequentially.

```text
Chef 1 → Order A
          Item 1 → Item 2 → Item 3

Chef 2 → Order B
          Item 4 → Item 5 → Item 6

Chef 3 → Order C
          Item 7 → Item 8 → Item 9
```

### `fullyParallel: true`

Individual items/tests can be distributed dynamically:

```text
Chef 1 → Test 1
Chef 2 → Test 2
Chef 3 → Test 3

Test 2 finishes

Chef 2 → Test 4

Test 1 finishes

Chef 1 → Test 5
```

This is a useful mental model for understanding Playwright's parallel execution.

---

# 16. Key Takeaways

1. **Worker = independent Node.js/OS process.**
2. Each worker has its own memory/process isolation.
3. Each worker controls its own browser instance.
4. `workers: 3` allows up to **3 workers** to execute concurrently.
5. With `fullyParallel: false`, Playwright primarily distributes **test files** across workers.
6. With `fullyParallel: true`, individual **tests can be distributed across workers**, including tests from the same file.
7. Workers dynamically pick up available work as they become free.
8. Do not assume workers share variables, browser state, fixtures, or setup state.
9. Fully parallel execution requires strong test-data and state isolation.
10. Shared database records, users, files, or external resources can cause race conditions and flaky tests.

---

## Quick Reference

```typescript
export default defineConfig({
  workers: 3,
  fullyParallel: true,
});
```

Think:

```text
              Playwright Scheduler
                       |
                Test Work Queue
                       |
           +-----------+-----------+
           |           |           |
           v           v           v
        Worker 1    Worker 2    Worker 3
           |           |           |
           v           v           v
        Test A      Test B      Test C
           |           |           |
           +-----------+-----------+
                       |
                 More Tests...
                       |
              Until Queue Empty
```

> **One-line interview answer:**  
> With `workers: 3` and `fullyParallel: true`, Playwright can run up to three independent worker processes concurrently, and individual tests can be dynamically distributed among those workers rather than keeping all tests from a file on a single worker.
