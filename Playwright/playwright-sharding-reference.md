# Playwright Test Sharding — Complete Reference

This document captures the Playwright sharding discussion and examples in one place for later reference.

## 1. What Is Playwright Sharding?

**Playwright Sharding** is a native feature that allows you to split a large test suite into smaller, independent parts (shards) and run them across multiple machines or CI/CD agents simultaneously.

While **workers** handle parallel execution on a single machine by utilizing multiple CPU cores, **sharding** provides horizontal scalability by distributing the workload across different machines.

The main idea is:

```text
One large test suite
        |
        +---- Shard 1 -> Machine / Runner 1
        |
        +---- Shard 2 -> Machine / Runner 2
        |
        +---- Shard 3 -> Machine / Runner 3
        |
        +---- Shard 4 -> Machine / Runner 4
```

The machines/runners execute their assigned shards at the same time.

---

## 2. Sharding vs Workers

| Feature | Workers (Vertical Parallelism) | Sharding (Horizontal Parallelism) |
|---|---|---|
| Execution | Runs multiple tests concurrently on **one machine**. | Splits and distributes tests across **multiple machines**. |
| Resource Sharing | Shares the same CPU, RAM, and hardware environment. | Each shard has its own dedicated runner/OS/hardware environment. |
| Best Used For | Maximizing a single runner's power. | Scaling large test suites that overload a single machine. |
| Typical Strategy | Increase workers on the runner. | Increase the number of shards/runners. |

A common approach is to combine both:

```text
4 remote machines
    |
    +-- Shard 1 + multiple workers
    +-- Shard 2 + multiple workers
    +-- Shard 3 + multiple workers
    +-- Shard 4 + multiple workers
```

So, **workers = parallelism inside a machine**, while **sharding = parallelism across machines**.

---

## 3. How Sharding Works

### Step 1 — Playwright identifies the test files

Playwright creates a deterministic, predictable list of test files.

### Step 2 — The files are partitioned

The test files are divided into shards based on the total number of shards configured.

The material captured here describes this as **file-level partitioning**, rather than splitting individual test cases across shards.

### Step 3 — Each shard runs independently

For example, with 3 shards:

```text
Total test files
       |
       +---- Shard 1/3
       +---- Shard 2/3
       +---- Shard 3/3
```

The deterministic calculation means a test file is assigned to one shard rather than being executed by multiple shards.

---

## 4. Core Sharding Command

Sharding is controlled using:

```bash
npx playwright test --shard=X/Y
```

Where:

- `X` = current shard index, starting from 1
- `Y` = total number of shards

### Example: 3 machines

Machine 1:

```bash
npx playwright test --shard=1/3
```

Machine 2:

```bash
npx playwright test --shard=2/3
```

Machine 3:

```bash
npx playwright test --shard=3/3
```

Each machine runs a different portion of the overall suite.

---

## 5. Local Execution Across Two Physical Machines

If you have two actual machines, for example:

```text
Machine A                Machine B
-----------              -----------
Same Git repository      Same Git repository
Same dependencies        Same dependencies
Shard 1/2                Shard 2/2
```

The machines do not need to communicate with each other during the test execution.

### Step 1 — Align the code

Both machines should have the same version of the repository.

For example:

```bash
git pull
npm install
```

The important point is that both machines should contain the same test files and compatible dependencies.

### Step 2 — Run the first shard

On Machine A:

```bash
npx playwright test --shard=1/2
```

### Step 3 — Run the second shard

On Machine B:

```bash
npx playwright test --shard=2/2
```

Both machines execute at the same time.

### Step 4 — Collect the results

Because the machines are independent, their reports are initially separate.

To combine them, use Playwright's **blob reporter**.

---

## 6. Blob Reporter

For sharded execution, each machine can generate a blob report.

You can configure the reporter in `playwright.config.ts`:

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  reporter: process.env.CI ? [['blob']] : [['html']],
});
```

Or specify it directly on the command line.

Machine A:

```bash
npx playwright test --shard=1/2 --reporter=blob
```

Machine B:

```bash
npx playwright test --shard=2/2 --reporter=blob
```

This produces a `blob-report/` directory on each machine.

---

## 7. Combining Local Shard Reports

Suppose Machine A produces:

```text
blob-report/
└── report-1.blob
```

and Machine B produces:

```text
blob-report/
└── report-2.blob
```

Copy the blob from Machine B to Machine A.

The final directory on Machine A can look like:

```text
your-project/
└── blob-report/
    ├── report-1.blob
    └── report-2.blob
```

Then run:

```bash
npx playwright merge-reports --reporter=html ./blob-report
```

This combines the individual shard reports into a single HTML report.

---

## 8. If You Only Have One Machine

You can simulate two shards on a single machine.

Open two terminal windows.

Terminal 1:

```bash
npx playwright test --shard=1/2 --reporter=blob
```

Terminal 2:

```bash
npx playwright test --shard=2/2 --reporter=blob
```

This is useful for verifying that your sharding configuration works before moving it to CI/CD.

However, this does **not** provide the same benefit as two remote machines because both executions still compete for the resources of the same physical machine.

---

# 9. Sharding in GitHub Actions / GitLab CI

This is where sharding becomes particularly useful.

You do **not** need to physically own multiple machines.

GitHub Actions or GitLab CI can provision separate remote runners for the jobs.

For example:

```text
                         CI Pipeline
                             |
                 +-----------+-----------+
                 |           |           |
                 v           v           v
              Runner 1    Runner 2    Runner 3
              Shard 1/3   Shard 2/3   Shard 3/3
                 |           |           |
                 +-----------+-----------+
                             |
                             v
                       Merge Reports
```

The runners execute the shards simultaneously.

---

## 10. Why Remote Sharding Reduces Execution Time

Suppose your complete suite takes:

```text
60 minutes
```

on one machine.

If the workload were perfectly balanced across 4 shards:

```text
60 minutes / 4 machines
≈ 15 minutes
```

The overall pipeline can therefore be dramatically shorter.

The actual time will not always be exactly one-quarter because of:

- Uneven test distribution
- Test setup/teardown
- Runner startup time
- Dependency installation
- Browser installation
- Infrastructure overhead
- A large test file becoming a bottleneck

The important concept is that the shards run **in parallel**, rather than waiting for one machine to finish before another starts.

---

# 11. GitHub Actions — Complete Example

GitHub Actions can use a **matrix strategy** to create separate runners for each shard.

Example:

```yaml
name: Playwright Sharded Tests

on: [push, pull_request]

jobs:
  playwright-tests:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        # Define 4 parallel shards
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}

      - name: Upload Shard Blob Report
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: all-blob-reports-${{ matrix.shardIndex }}
          path: blob-report/
          retention-days: 1

  merge-reports:
    if: ${{ !cancelled() }}
    needs: playwright-tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Download all Blob Reports
        uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: all-blob-reports-*
          merge-multiple: true

      - name: Merge Reports into HTML
        run: npx playwright merge-reports --reporter=html ./all-blob-reports

      - name: Upload Final HTML Report
        uses: actions/upload-artifact@v4
        with:
          name: final-html-report
          path: playwright-report/
          retention-days: 7
```

---

## 12. What the GitHub Matrix Is Doing

This section is important.

This:

```yaml
matrix:
  shardIndex: [1, 2, 3, 4]
  shardTotal: [4]
```

causes GitHub Actions to create four parallel job instances.

Conceptually:

```text
Job 1:
npx playwright test --shard=1/4

Job 2:
npx playwright test --shard=2/4

Job 3:
npx playwright test --shard=3/4

Job 4:
npx playwright test --shard=4/4
```

The GitHub-hosted runners are separate machines/environments.

Therefore:

```text
Runner 1 -> Shard 1/4
Runner 2 -> Shard 2/4
Runner 3 -> Shard 3/4
Runner 4 -> Shard 4/4
```

The jobs execute concurrently.

---

# 13. GitLab CI Example

GitLab provides the `parallel` keyword for creating multiple instances of a job.

Example:

```yaml
stages:
  - test
  - report

playwright-shards:
  stage: test
  image: mcr.microsoft.com/playwright
  parallel: 4

  script:
    - npm ci
    - npx playwright test --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL --reporter=blob

  artifacts:
    when: always
    paths:
      - blob-report/
    expire_in: 1 day
```

The important variables are:

```text
CI_NODE_INDEX
CI_NODE_TOTAL
```

Conceptually, with:

```yaml
parallel: 4
```

GitLab creates jobs equivalent to:

```text
Job 1 -> --shard=1/4
Job 2 -> --shard=2/4
Job 3 -> --shard=3/4
Job 4 -> --shard=4/4
```

---

# 14. Merging Reports in CI/CD

When shards run on different machines, each machine generates its own results.

You therefore need a final job that:

1. Waits for all shard jobs.
2. Downloads the blob reports.
3. Places them together.
4. Runs `merge-reports`.
5. Produces the final HTML report.

The central command is:

```bash
npx playwright merge-reports --reporter=html ./blob-report
```

For GitHub Actions, the merge job can use:

```yaml
needs: playwright-tests
```

This means the merge job waits for the shard jobs to complete.

---

# 15. End-to-End CI Flow

The complete flow is:

```text
Developer pushes code
        |
        v
CI Pipeline starts
        |
        v
Matrix / Parallel jobs created
        |
        +------------------+------------------+------------------+
        |                  |                  |
        v                  v                  v
    Runner 1           Runner 2           Runner 3
    Shard 1/3          Shard 2/3          Shard 3/3
        |                  |                  |
        +------------------+------------------+
                           |
                           v
                   Blob reports collected
                           |
                           v
                     Merge job starts
                           |
                           v
              npx playwright merge-reports
                           |
                           v
                   Unified HTML report
```

---

# 16. Important Limitation — The Bottleneck File

One important caveat is that sharding is based on **test files**, not test execution duration.

Imagine:

```text
Test file 1 -> 1 minute
Test file 2 -> 1 minute
Test file 3 -> 1 minute
...
Test file 30 -> 1 minute
Huge test file -> 15 minutes
```

With four shards, most shards may finish quickly while the shard containing the huge file continues running.

For example:

```text
Shard 1 -> 8 minutes
Shard 2 -> 8 minutes
Shard 3 -> 8 minutes
Shard 4 -> 15 minutes
```

The pipeline effectively waits for the slowest shard.

Therefore, very large test files can become a bottleneck.

A practical approach is to keep test files reasonably small and balanced so that the workload is distributed more evenly.

---

# 17. Test Independence

Because shards execute independently, tests should not depend on another shard completing first.

Avoid assumptions such as:

```text
Test A must run before Test B
```

if Test A and Test B can be placed on different shards.

The tests should be sufficiently independent that each shard can execute its assigned tests without relying on another shard's execution state.

---

# 18. Don't Over-Parallelize Too Early

Sharding is primarily useful when the test suite is large enough that a single runner has become a significant execution bottleneck.

For a relatively small suite, increasing workers on a single runner may already provide enough parallelism.

Sharding becomes more valuable when the suite is large and CI execution time is a significant concern.

---

# 19. The Most Important Concept

The easiest way to remember the difference is:

```text
WORKERS
=======

One machine
     |
     +-- Worker 1
     +-- Worker 2
     +-- Worker 3
     +-- Worker 4


SHARDING
========

Multiple machines
     |
     +-- Machine 1 -> Shard 1
     +-- Machine 2 -> Shard 2
     +-- Machine 3 -> Shard 3
     +-- Machine 4 -> Shard 4
```

And you can combine them:

```text
Machine 1
  ├── Shard 1
  │    ├── Worker 1
  │    ├── Worker 2
  │    └── Worker 3
  │
Machine 2
  ├── Shard 2
  │    ├── Worker 1
  │    ├── Worker 2
  │    └── Worker 3
  │
Machine 3
  ├── Shard 3
  │    ├── Worker 1
  │    ├── Worker 2
  │    └── Worker 3
```

So:

**Sharding gives you multiple machines.**

**Workers give you parallel execution within each machine.**

---

# 20. Quick Reference Commands

### One shard out of two

```bash
npx playwright test --shard=1/2
```

```bash
npx playwright test --shard=2/2
```

### Three shards

```bash
npx playwright test --shard=1/3
npx playwright test --shard=2/3
npx playwright test --shard=3/3
```

### Four shards with blob reports

```bash
npx playwright test --shard=1/4 --reporter=blob
npx playwright test --shard=2/4 --reporter=blob
npx playwright test --shard=3/4 --reporter=blob
npx playwright test --shard=4/4 --reporter=blob
```

### Merge reports

```bash
npx playwright merge-reports --reporter=html ./blob-report
```

---

# 21. Key Takeaway

If your Playwright suite takes a long time on one machine, sharding allows the suite to be divided across multiple CI runners.

For example:

```text
Without sharding:

One runner
    |
    +-- All tests
    |
    +-- 60 minutes


With 4-way sharding:

Runner 1 -> 25% of tests
Runner 2 -> 25% of tests
Runner 3 -> 25% of tests
Runner 4 -> 25% of tests

All run simultaneously
    |
    v
Potentially much shorter total execution time
```

The CI/CD platform is responsible for providing the multiple remote runners. Playwright is responsible for deciding which shard executes which portion of the test suite.

The final reporting step collects the individual shard results and creates one consolidated HTML report.
