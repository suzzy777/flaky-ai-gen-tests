# Results per RQ, in short (2026-11-01)

Numbers as they stand in the paper now (paper/sections/results.tex).

## Main study: filtering steps (paper/sections/setup.tex)

**Words used below:**
- **Focal method:** the production method a test is generated for.
- **File:** one generated test class. ChatUniTest writes one file for the focal method of each developer test.
- **Test method:** one `@Test` method inside a file. A file usually has 1-5 of them.

**Main study funnel** (flaky and non-flaky sources together):

| Step | What is counted | Count |
|---|---|---|
| Start | developer tests we start from: 144 flaky (ReproFlake) + 156 non-flaky | 300 |
| Focal method identified | developer tests with one focal method | 209 |
| Project builds | developer tests with one focal method | 209 |
| ChatUniTest generates a file | files | 192 |
| File passes alone | files (test methods inside them) | 126 (240) |
| iDFlakies can check the file | files | 96 |
| Polluter | files (test methods that pollute) | 3 (10) |
| State-setter | files (test methods that set the state) | 2 (3) |

All 300 come from 80 project archives (44 projects). The non-flaky focal methods are sampled from the 13 modules where a file for a flaky test passes alone (up to 15 per module, fixed seed), so they add no new projects.

**Flaky tests by type** (types exist only for the flaky tests):

| Step | OD | ID | TD | NIO | Total |
|---|---|---|---|---|---|
| Start (ReproFlake) | 36 | 36 | 36 | 36 | 144 |
| Focal method identified | 23 | 17 | 9 | 4 | 53 |
| Builds | 23 | 17 | 9 | 4 | 53 |
| ChatUniTest generates a test | 23 | 13 | 9 | 3 | 48 |
| Generated file passes alone | 13 | 6 | 3 | 1 | 23 |
| iDFlakies can check | 13 | 4 | 0 | 1 | 18 |
| Polluter | 3 | 0 | 0 | 0 | 3 |
| State-setter | 1 | 0 | 0 | 0 | 1 |

**Why tests drop out at each step:**
1. **Focal method:** both techniques (Jaccard name similarity and the LLM) must pick the same single method. This removes tests of a whole component (e.g. wildfly's subsystem test), tests of a framework mechanism that call no production method directly (junit-quickcheck), and tests whose name points to no method (fastjson `Issue1480#test_for_issue`). 80 → 32 archives. NIO and TD tests drop most here.
2. **Build:** everything builds. hadoop (YARN-9405) needs protoc 2.5.0: our first non-flaky generation ran on a machine with a newer protoc, so its 15 hadoop focal methods were generated again on a machine with the right one (12 files). graylog2-server builds once its JavaScript frontend build is skipped (the Java tests don't need it).
3. **Generation:** ChatUniTest keeps no test for 5 flaky tests. For graylog2-server, the time limit was raised from 30 minutes to 3 hours, because just parsing the project takes over an hour. When the focal method has overloads, only tests for the overload that the developer test calls are kept.
4. **Pass alone:** the file must compile and pass alone; 13 archives have no such file. Example: the graylog test reads private fields by reflection, which graylog's own build check forbids in tests.
5. **iDFlakies:** it can't run on 4 archives (mixed JUnit 4 and 5, its runner crashes, or a test JVM doesn't exit). One of them is hadoop: every iDFlakies round runs the whole YARN services suite with mini clusters and hit the 6 h limit with 0 of 20 rounds. So the hadoop files (1 flaky side, 9 non-flaky side) are checked with reruns, NonDex, NIO and the stress test, but not for order-dependence.
6. **TD check, FlakeSync** (Hana): ran on every file that passes alone except hadoop, because FlakeSync's default blacklist excludes `org.apache.hadoop` from instrumentation. **Stress test** (Devanshi, stress-ng CPU/memory): ran on the main study and rows 1-15, hadoop included.

## The experiments and what changes between them

Same for all: gpt-4o (`gpt-4o-2024-08-06`), temperature 0.5. The agent is mini-swe-agent with a 40-step and $3 limit per run.

Each condition adds text to the one before it. The exact text that is added (for JUnit 5 tests, `@Before`/`@After` read `@BeforeEach`/`@AfterEach`):

| Experiment | Generator | Exact text added to the prompt |
|---|---|---|
| Main study, Prompt default | ChatUniTest | nothing: ChatUniTest's own prompt (it already says: "Compile without errors, and use reflection to invoke private methods.") |
| Prompt cleanup | ChatUniTest | **System message:** "The tests may run in any order together with the project's other tests in the same JVM, so they must not change state that other tests see: if a test changes a static field, a singleton (e.g. an object returned by getInstance()), a system property, or any field set by reflection, save the original value before changing it (in @Before) and restore it in an @After method." <br> **Repair step:** "If the test changes a static field, a singleton or a field set by reflection, save the original value and restore it in an @After method." |
| Agent A | agent | ChatUniTest's first prompt, then: "You are in the project's repository: <path> (JDK <n>, Maven offline). Every bash command runs in a new shell starting in that directory. You can read any file of the project (e.g. with cat, sed -n, grep) to find what you need." <br> "1. Write the JUnit test class `<Class>` (package `<pkg>`) to <path>. Do not change any other file of the project. 2. Compile and run it with: <mvn command>. Run only your own test class. 3. Fix it until it compiles and its tests pass. 4. Then finish by running `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` as its own command." |
| Agent F | agent | A + a new first step: "1. Read the project's existing tests first. They are in <src/test/java>; read the ones you need. Your test class will be added to the project's test suite next to them." <br> + in the "Write the test class" step: "Text in your reply is not saved: write the test file with a bash command (e.g. `cat > <path> << 'EOF'`)." |
| Agent FC | agent | F + the same two **cleanup** sentences as Prompt cleanup (the first in the system message, the second in the "Fix it until it compiles and its tests pass" step) |
| Agent FCV | agent | FC + **victim sentence** in the system message: "Each test must pass no matter which tests ran before it: do not assume that shared state (static fields, singletons, system properties) is in its default state; set up the state your test needs in @Before." |

**The two extra sentences, in plain words:**

| | Cleanup sentence | Victim sentence |
|---|---|---|
| Protects against | the generated test being a **polluter**: it changes shared state and leaves it changed, so *later* tests fail | the generated test being a **victim**: it fails because an *earlier* test left shared state changed |
| What it asks (our short summary, not the prompt's words) | Clean up after yourself: if you change a static field, a singleton (e.g. from `getInstance()`), a system property or a field set by reflection, save the old value in `@Before` and put it back in `@After` | Don't trust what others left behind: don't assume shared state is in its default state; set up the state your test needs yourself in `@Before` |
| Used in | Prompt cleanup, Agent FC, Agent FCV | Agent FCV only |
| Did it work? | yes: polluters 4 of 8 -> 0 of 12 files (ChatUniTest), 3 -> 1 run (agent F -> FC) | no: the marine-api victims stay (FCV 2 of 2 usable files) |

Exact wording in the prompt (the whole sentence that is added, nothing else):
- **Cleanup:** "The tests may run in any order together with the project's other tests in the same JVM, so they must not change state that other tests see: if a test changes a static field, a singleton (e.g. an object returned by getInstance()), a system property, or any field set by reflection, save the original value before changing it (in @Before) and restore it in an @After method."
- **Victim:** "Each test must pass no matter which tests ran before it: do not assume that shared state (static fields, singletons, system properties) is in its default state; set up the state your test needs in @Before."

The prompt and agent experiments use 10 focal methods, 3 runs each: rows 1-15 (the 3 focal methods whose main-study tests are polluters + 2 that write fields by reflection) and rows 16-30 (5 more chosen where flakiness is likely).

**Results per experiment** (files; P = polluter, S = state-setter, V = victim):

| Experiment | Generated | Pass alone | OD-checked | P | S | V | Developer tests broken (not in IDoFT) |
|---|---|---|---|---|---|---|---|
| Main study (flaky + non-flaky sources) | 192 | 126 | 96 | **3** | 2 | 0 | 35 (12) |
| Rows 1-15, ChatUniTest default | 13 | 8 | 8 | **4** | 2 | 0 | 507 (418) |
| Rows 1-15, ChatUniTest cleanup | 15 | 12 | 12 | 0 | 1 | 0 | 0 |
| Rows 1-15, Agent A | 15 | 13 | 13 | **6** | 3 | 0 | 41 (14) |
| Rows 1-15, Agent F | 14 | 11 | 11 | **3** | 3 | 3 | 27 (12) |
| Rows 1-15, Agent FC | 15 | 12 | 12 | **1** | 3 | 2 | 14 |
| Rows 1-15, Agent FCV | 15 | 9 | 9 | 0 | 3 | 2 | 0 |
| Rows 16-30, ChatUniTest default | 14 | 12 | 12 | 0 | 1 | 0 | 0 |
| Rows 16-30, ChatUniTest cleanup | 15 | 9 | 9 | 0 | 1 | 0 | 0 |
| Rows 16-30, Agent A | 14 | 12 | 12 | 0 | 2 | 0 | 0 |
| Rows 16-30, Agent F | 15 | 12 | 12 | 0 | 0 | 0 | 0 |
| Rows 16-30, Agent FC | 15 | 10 | 10 | 0 | 0 | 0 | 0 |
| Rows 16-30, Agent FCV | 15 | 12 | 12 | 0 | 0 | 0 | 0 |

No file is flaky alone in any row (reruns, NonDex and NIO everywhere; stress on the main study and rows 1-15; FlakeSync everywhere except hadoop). P, S and V add up to the totals in RQ1: 17 polluters, 21 state-setters, 7 victims.

## RQ1: How often are generated tests flaky?
- **All flakiness checks, one per type:**

  | Check | Flakiness type | What it does | Checked | Result |
  |---|---|---|---|---|
  | 100 reruns | non-deterministic (NOD) | run each method alone 100 times | all methods that pass alone | 0 flaky |
  | NonDex | implementation-dependent (ID) | shuffle unordered Java APIs (100 seeds) | all methods that pass alone | 0 flaky |
  | NIO | non-idempotent (NIO) | run each method twice in one JVM, 100 times | all methods that pass alone | 0 flaky |
  | FlakeSync | timing / concurrency (TD) | find methods that run concurrently, then delay them (100 ms up to 12.8 s) | 510 methods (all experiments, no hadoop) | 0 flaky (concurrency found in only 1 method, a dubbo agent test; no delay makes it fail) |
  | Stress (stress-ng) | resource-dependent | each method alone 10x under full CPU load and 10x under memory pressure | 421 methods (main study + rows 1-15) | 0 flaky |
  | iDFlakies + confirmation | order-dependent (OD) | 20 random orders of the whole module test suite with the generated file, then confirm each case in a fixed order | 228 files (main study 96, rows 1-15: 65, rows 16-30: 67) | **17 polluter files, 21 state-setter files, 7 victim files** (iDFlakies finds 4 of the 7 victims; all 7 by direct check after the known developer polluter) |

- **Flaky alone:** so no generated test method is flaky by itself (main study: 0 of 240 methods). All the flakiness is order-dependence.
- **Polluters:** generated tests often break developer tests.
  - Main study (ChatUniTest; 126 files pass alone, 96 of them checkable by iDFlakies): **3 of 96 files are polluters**, and **10 of the 240 test methods in the 126 files pollute** (each method run alone before a victim, twice). Together they break 35 developer tests.
  - Across all experiments: **17 polluter files, from just 4 focal methods**; **54 of the 67 test methods in these files pollute** (in 12 files every method does). They break **over 500 developer tests**, 478 of them through one ormlite log setting. Most of these tests weren't known as flaky.
- **State-setters:** 21 files, all harmless (main study: 2 files, 3 methods). They make one brittle developer test (`LoggerTest`) pass.

## RQ2: Why does it happen?
- **What polluters do:** every polluter replaces a shared static field or singleton with a mock or empty object, and never puts it back.
  - 15 of the 17 do this by reflection.
  - 16 of the 17 have no `@After` at all.
- **Fix confirmed:** adding an `@After` that restores the field makes all 35 main-study victims pass.
- **Why only 4 focal methods:** those methods themselves depend on shared state (`DubboProtocol.INSTANCE`, the `parsers` map, the static log backend).
- **Generated victims:** these come from the agent copying the setup of a developer test that is already order-dependent (marine-api `SentenceFactory`).

## RQ3: Does reading the existing tests help?
- **Fewer polluters:** 6 runs (A) → 3 (F) of 15, though not significant (p = 0.42).
- **More cleanup:** 36% of F's files have an `@After` with code, vs 0% of A's.
- **But it copies weaknesses:** every usable marine-api test from an agent that read the tests is a victim (F 3/3, FC 2/2, FCV 2/2; A 0/2). So reading the tests turns polluters into victims.

## RQ4: Do prompt instructions help?
- **Cleanup sentence, ChatUniTest:** polluters drop from **4 of 8 → 0 of 12** files (p = 0.014).
- **Cleanup sentence, agent:** polluters drop from 3 runs (F) → 1 (FC).
- **Victim sentence (FCV):** no effect.
  - FCV's 0 polluters comes from the cleanup sentence plus unusable runs, not from the victim sentence.
  - The marine-api victims remain.
- **Instructions make generation harder:**
  - step-limit runs: 4 (A) → 8 (FCV) of 30;
  - usable runs: 25 → 21 of 30.
  - Neither difference is significant.

## RQ5: Quality and cost
- **Quality:** generated tests are close to developer tests for the same focal method.
  - Line coverage 85% vs 93% (p = 0.002); mutation 74% vs 82% (p = 0.049, borderline).
  - Most pairs are identical.
- **Agents that read the tests write the strongest tests:** mutation F 90%, FCV 88%, vs ChatUniTest 59–68% (rows 1–15).
- **Cost per run:**
  - ChatUniTest ≤ $0.10; agent $0.26–0.47.
  - The cleanup sentence almost doubles the agent's cost ($0.26 → $0.46).

## One-line takeaway
Generated tests aren't flaky by themselves, but they break developer tests by changing shared state and not restoring it. One cleanup sentence mostly fixes that, but it makes generation harder and more expensive.
