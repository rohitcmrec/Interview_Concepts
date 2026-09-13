# Scenario-Based Challenges



## 1. The Pesticide Paradox: When Your Tests Stop Finding Bugs

Boris Beizer coined the term in 1990, and the analogy is perfect: just as insects develop resistance to pesticides used repeatedly, software develops a kind of "resistance" to test cases run repeatedly. The same tests, run against an evolving codebase, find fewer and fewer bugs over time — not because the software is getting better, but because **your tests are only looking in places they've already looked**.

This is one of the most insidious long-term quality risks in manual testing, precisely because it's invisible. The test suite keeps passing. Reports look green. Everyone feels confident. But the coverage is quietly calcifying while the software keeps changing around it.

### Why It Happens

When a test case is written, it captures a specific understanding of the system at a specific point in time. It tests a particular input, a particular path, a particular expected output. Over months and years, the system evolves — new code is added, old code is refactored, integrations change, user behaviour shifts — but the test case doesn't evolve with it. It keeps testing the same narrow slice it always tested, while the new risk surface grows untouched around it.

The paradox deepens because passing tests create psychological comfort. Teams stop questioning whether their tests are still the *right* tests because the tests are all passing, and passing feels like quality.

### Strategy 1: Scheduled Test Case Retirement and Renewal

Treat your test suite like a garden, not a library. A library accumulates everything. A garden requires regular pruning, replanting, and cultivation.

Establish a deliberate cadence — quarterly works well for most long-running projects — where you review your existing test cases not for correctness but for **continued relevance**. Ask for each test case: Is this area of the system still active and unchanged? Is this scenario still a realistic user path? Is this risk still meaningful? Has the underlying business rule this test was based on changed?

Test cases that no longer reflect the system's actual behaviour should be retired or rewritten. Test cases covering stable, low-risk areas that have passed without incident for two years might be moved to a lighter execution schedule or converted to a quick sanity check rather than a full scripted run. The freed capacity gets redirected to newer, higher-risk areas.

This isn't reducing testing rigour — it's reallocating it intelligently.

### Strategy 2: Rotating Testers Across Modules

A tester who has tested the same module for 18 months has developed a mental model of that module that is simultaneously deep and narrow. They know exactly how it behaves, which means they've also stopped noticing its abnormalities. They've normalised the quirks.

A fresh tester brings fresh assumptions — and fresh assumptions generate fresh test ideas. When someone encounters a module without the accumulated familiarity of the original tester, they naturally probe it differently: they try the things the original tester learned not to try because "that's just how it works."

Build deliberate rotation into your team's testing assignments. When you rotate onto a new module, your first task before running existing scripts is to spend time in exploratory testing — forming your own mental model of how the system behaves, independent of the existing test suite. This often surfaces scenarios the original tester never documented.

### Strategy 3: Exploratory Testing as a Formal Complement to Scripted Testing

Scripted test cases are, by design, deterministic. They test what you already thought to test. Exploratory testing is non-deterministic — it tests what you think of in the moment, guided by curiosity, anomalies, and risk intuition.

On long-running projects, the ratio of scripted to exploratory testing should shift over time. Early in a project, scripted testing dominates because the system is new and you need comprehensive coverage documentation. Two years in, the high-value scripted tests are running and passing reliably. The incremental value of running those same scripts again is low. The incremental value of a skilled tester spending two hours exploring a module they haven't examined closely in months is high.

Schedule formal exploratory sessions — time-boxed charters — as a recurring activity, not an ad hoc one. These sessions aren't improvised; they have a defined focus area, a time limit, and a note-taking structure. But within those constraints, the tester follows their instincts and the system's behaviour.

### Strategy 4: Using Defect History to Redirect Test Focus

Your historical defect data is a map of where the system's risk actually lives. Pull a defect report for the past 12 months and look at the distribution: which modules generated the most defects? Which types of defects recurred? Which areas that had zero defects in testing later produced production issues?

High historical defect density is a strong signal that your test cases for that area may have been insufficient even when they were passing — or that the area is inherently prone to regression. Either way, it warrants deeper, more varied testing than a test case written three years ago can provide.

Zero historical defects in an area can mean two things: the area is genuinely stable, or your tests aren't good enough to find the defects that are there. Distinguish between these by periodically applying exploratory testing to "clean" areas. If you find bugs in an area that has never had a defect, your test suite has a blind spot — not a clean record.

### Strategy 5: Introducing Variability Into Existing Tests

Where you can't replace tests, vary them. For a test case that tests "user submits a form with valid data," the scripted test probably uses the same test data every time. Vary the data: use different character sets, different length values, different data combinations. The test case stays the same structurally, but the inputs change, which slightly expands the coverage without requiring entirely new test design.

Similarly, vary the sequence in which you execute related test cases. The order of operations sometimes matters in ways the original test design didn't anticipate, and varying execution order occasionally reveals state-dependency bugs that sequential execution misses.



## 2. Legacy Systems: Regression Testing Without Institutional Memory

This is arguably the most challenging scenario in all of manual testing. You're handed a system that is large, old, incompletely documented, and staffed by people who joined after the original builders left. Nobody fully understands it. Parts of it haven't been touched in years. And your job is to make sure changes to it don't break anything.

The brutal reality of legacy system regression testing is that **you will never have complete coverage**. The goal is not comprehensive coverage — it's *intelligent* coverage, built systematically from everything you can learn about a system that was never designed to be easy to test.

### Phase 1: Archaeological Investigation

Before writing a single test case, you spend time becoming an archaeologist. You're excavating everything that was left behind.

**Existing test artefacts** — find every test case, test plan, test script, or testing spreadsheet that exists, regardless of how outdated it looks. Even a test suite from eight years ago tells you what the original testers thought was important to test. That's valuable signal. Don't run it yet — evaluate it first. Is this area still active? Has the underlying feature changed? Is the expected behaviour described still the expected behaviour?

**Defect history** — pull every bug that has ever been logged against this system, as far back as records exist. You're looking for patterns: which modules have historically been defect-prone? Which integration points have failed repeatedly? Which types of changes reliably caused regressions? Defect history is the closest thing to institutional memory that gets written down.

**Release notes and change logs** — even sparse release notes tell you what changed when. Cross-reference these with defect history: when a release touched Module X, did defects follow? This builds a risk map of which modules are sensitive to change and which are genuinely stable.

**Code history** — if you have access to version control (Git logs, for example), look at commit frequency by module. Code that changes frequently is higher risk for regression than code that hasn't been touched in four years. You don't need to read the code — just the activity pattern.

**Support and incident tickets** — customer support logs and production incident records are a goldmine of real-world failure modes. These are the scenarios that scripted tests missed and real users found. Mine them for both specific scenarios to test and categories of failure that suggest systematic test gaps.

### Phase 2: Building a Functional Coverage Map

With institutional knowledge gone, you reconstruct system understanding through direct exploration. This means spending structured time in the system, navigating every accessible area, documenting what you find.

Create a **functional map**: a structured inventory of every feature, workflow, and integration point in the system. This doesn't need to be detailed — at this stage you're building breadth, not depth. You want to know that the system has these 47 functional areas before you start deciding how to prioritise them.

For each area in your map, make a rough assessment of: How critical is this to core business operation? How frequently do users interact with it? How often does this code change? Has it historically produced defects? These four questions give you a preliminary risk rating for each area, which drives your coverage prioritisation.

### Phase 3: Identifying the Critical Paths

On a legacy system without full documentation, you can't test everything — and you shouldn't try. Focus first on identifying the **critical user journeys**: the end-to-end flows that represent the core value the system delivers.

For a legacy financial system, this might be: process a transaction, generate a report, reconcile end-of-day balances, produce a regulatory export. For a legacy CRM: create a customer record, log an interaction, generate a quote, close an opportunity. These critical paths are your regression testing spine. If everything else is uncertain, these must pass.

Interview current users and business stakeholders, not to get technical documentation, but to answer one question: *"If this system were to break in a way that stopped your work for a day, what would that look like?"* Their answers tell you what they depend on most, which is a better guide to critical paths than any technical document.

### Phase 4: Building the Regression Suite Incrementally

Don't try to build a complete regression suite before you start testing. Build it incrementally, sprint by sprint, change by change.

Every time a change is made to the system, document what you tested to verify it, what adjacent areas you checked for regression, and what you found. Over time, this accumulates into a regression library that reflects both the system's actual behaviour and the real-world risk profile of each area.

When a production defect occurs, immediately convert the failure scenario into a regression test case. These are the most valuable test cases in a legacy system — they represent real failure modes that the system is historically capable of producing.

### Phase 5: Managing the Uncertainty Honestly

The most professional thing you can do with a legacy system is be transparent about the limits of your coverage. Maintain a living document that maps your test coverage against your functional map — showing not just what you test, but explicitly what you don't test and why.

When a release goes out, communicate the coverage position to stakeholders: *"We have strong regression coverage for core transaction processing and reporting. Coverage for the legacy batch import module is limited to basic happy-path scenarios due to incomplete documentation and no historical test artefacts. We recommend enhanced production monitoring for that area post-release."*

This isn't admitting defeat. It's intelligent risk communication — which is exactly what a legacy system environment demands.



## 3. Root Cause Analysis: When the Same Module Keeps Failing

A single defect in a module is a bug. Two defects in the same module is a coincidence worth noting. Three or more defects in the same module is a **pattern** — and patterns don't point to the code, they point to the process.

When you see repeated defects clustering in one area, the bug reports are symptoms. Your job is to find the disease.

### The Mindset Shift: From Bug Fixing to Process Fixing

The default organisational response to repeated defects in a module is to fix each bug individually and add test cases to cover each failure. This is treating symptoms. The bugs keep coming back — in slightly different forms, through slightly different paths — because the conditions that produce bugs haven't changed.

Genuine root cause analysis asks not *"what went wrong in the code?"* but *"what went wrong in the process that allowed this code to be written, reviewed, and released without the defect being caught?"* The answer to that question leads to a systemic fix that prevents the entire category of defect, not just the specific instance.

### Step 1: Aggregate and Pattern-Analyse the Defect Data

Pull all defects logged against the module in question — not just recent ones, as far back as records go. For each defect, note the type of defect (logic error, data handling, integration failure, UI inconsistency), when it was introduced (which sprint, which developer, following which type of change), when it was found (in development, in testing, in production), and what triggered it (specific user action, data state, environment condition).

Look for patterns across these dimensions. Are defects introduced predominantly after a specific type of change — refactoring, performance optimisation, new feature additions? Are they found predominantly at a specific test stage — or worse, predominantly in production? Are they concentrated in a specific type of functionality — data transformation, state management, API calls?

The pattern tells you where to look for the root cause.

### Step 2: Apply the 5 Whys

The 5 Whys is a deceptively simple technique: you take a problem statement and ask "why" repeatedly until you reach a root cause rather than a symptom. The standard wisdom is five iterations, but the real rule is to keep going until you reach something actionable at a process level.

Applied to a defect pattern, it might look like this:

*The payment calculation module has produced five defects in three months.* Why? Because edge cases in discount stacking logic are consistently incorrect. Why? Because the discount stacking rules are complex and the code implementing them is difficult to understand. Why? Because the business rules for discount stacking were never formally documented — they exist in the original developer's head and in a scattered email chain from two years ago. Why? Because requirement documentation for this module was always treated as informal given the original developer's deep domain knowledge. Why? Because the team never established a standard for requirement formalisation for complex business rules.

The root cause isn't the code. It's the absence of formalised documentation for complex business logic. Fixing that — establishing a documentation standard and backfilling the discount rule documentation — prevents an entire category of future defects, not just the five you've already seen.

### Step 3: Categorise the Root Cause

Root causes in software development cluster into recognisable categories, and identifying the category helps you design the right systemic fix:

**Requirement ambiguity** — the expected behaviour was never clearly defined, leading to different interpretations by different developers at different times. Fix: strengthen requirement documentation, acceptance criteria, and Three Amigos review for this module's domain.

**Knowledge concentration** — the module's logic is understood by one person (or was understood by a person who has since left). Fix: knowledge transfer sessions, documentation sprints, pair programming on changes to this module.

**Inadequate test coverage** — the defect types that keep occurring represent scenarios that the test suite consistently doesn't cover. Fix: targeted test design work to address the specific scenario category, not just the specific bug.

**Complexity without safeguards** — the module is inherently complex and prone to regression when changed, but there are no automated checks, no code review standards, and no testing gates proportionate to that complexity. Fix: establish module-specific review and testing requirements calibrated to its risk level.

**Technical debt** — the module's internal structure makes it fragile, where changes in one place unpredictably break behaviour in another. Fix: this is a development conversation about refactoring, but the tester's role is to make the business case for it using defect frequency data.

### Step 4: Bring the Analysis to the Team, Not Just the Fix

A root cause analysis conducted in isolation and handed to the team as a verdict produces defensiveness. A root cause analysis conducted collaboratively produces ownership.

Bring your aggregated defect data and your preliminary pattern analysis to a retrospective or dedicated quality review session. Present what you've observed — the defect frequency, the pattern, the hypothesis about root cause — and invite the team to challenge and enrich it. Developers who worked on the module may have context that changes your analysis. Product owners may recognise that a specific requirement gap is responsible. The shared investigation produces a more accurate root cause and, crucially, a team that owns the fix.

Frame the conversation around the process, explicitly and consistently. Not *"why does this module have so many bugs"* — which people hear as *"whose fault is this"* — but *"what in our process makes it hard to get this module right, and what would make it easier?"*

### Step 5: Measure Whether the Fix Worked

A root cause analysis without measurement is just a retrospective conversation. After implementing the process fix, track defect rates in the affected module for the next three to six months. Did frequency decrease? Did the specific defect type that was recurring disappear? Did a new type emerge that suggests the real root cause was something adjacent to what you identified?

If defect rates decrease, you have evidence that the fix worked — and a case study to apply the same analysis approach to other problem areas. If they don't, your root cause identification was incomplete and you need to go deeper.



The common thread across all three of these scenarios is a consistent orientation: **look past the immediate problem to the systemic condition that produced it**. The Pesticide Paradox is a symptom of test suite stagnation. Legacy system regression risk is a symptom of knowledge loss and coverage ambiguity. Repeated defects are a symptom of a process that makes certain types of mistakes easy to make. In each case, the tester who creates lasting value is the one who fixes the condition, not just the instance.
