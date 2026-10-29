# Meeting summary: flakiness of LLM-generated unit tests (2026-10-29)

ChatUniTest (gpt-4o) and an LLM agent generate unit tests for focal methods of real Java projects. We check whether the generated tests are flaky themselves, or make the project's existing (developer) tests flaky. We also compare their quality with the developer tests.

**Checks (all experiments use the same settings):**
- rerun: 100 runs of each method alone;
- NonDex: 100 seeds;
- NIO: 100 runs;
- iDFlakies: 20 rounds of the whole module suite with the generated file added;
- every iDFlakies flag is then confirmed with `od_confirm` (M8 surefire, exact order), plus a method-by-method re-check (`od_recheck`).

## Headlines

1. **The generated tests are never flaky by themselves.** Rerun, NonDex and NIO found 0 flaky tests across all experiments.
2. **The problem is order-dependence: generated tests break developer tests.** In the main study, 3 of 18 flaky-side files that iDFlakies could check are polluters, against 0 of 78 on the non-flaky side (p = 0.006).
3. **One root cause covers every polluter.** The test replaces a static singleton or static field with a mock or an empty object, often by reflection, and never restores it. None has an `@After`. Adding an `@After` that restores the field fixed 35 of 35 victims.
4. **A prompt that asks for cleanup removes it.** Current prompt: 4 of 8 files are polluters, breaking 507 developer tests. Cleanup prompt: 0 of 12 (Fisher p = 0.014).
5. **Most of the broken developer tests are new.** Of the 507 broken by the current prompt, 418 are not in IDoFT's list of known flaky tests.
6. **An agent that reads the existing tests first pollutes less, but copies their weaknesses.** Agent A (no reading): 2 marineapi polluters. Agent F (reads tests first): 0 polluters, but its 3 marineapi files become victims of an existing developer polluter, because F copied the setup of a test that's already a known victim.
7. **Quality:** for the same focal method, the generated tests usually match the developer tests on coverage and mutation score. When they differ, the developer tests are more often better. Line coverage is 86% vs 94% (Wilcoxon p = 0.004). Generated tests on the flaky side are much weaker: line 66%, mutation 39%.

## The reflection issue, in short

**What reflection is:** a Java feature that lets code reach into a class at runtime and read or change things it normally can't, such as private or static fields. For example:

```java
Field f = DubboProtocol.class.getDeclaredField("INSTANCE");
f.setAccessible(true);      // ignore "private"
f.set(null, mockProtocol);  // replace the shared singleton with a mock
```

**Why it matters here:**
- ChatUniTest's own prompt tells the model to "use reflection to invoke private methods or fields if needed". So the generated tests often swap a shared singleton or static field for a mock or an empty object, and never put it back.
- That leftover state breaks the developer tests that run later, which covers 8 of the 10 polluter files.
- The cleanup prompt still allows reflection, but asks for an `@After` that restores the field.

## Research questions and results so far

### RQ1. To what extent do AI-based test generation tools generate flaky tests?

**Answer:** the generated tests are almost never flaky on their own. The flakiness they cause is order-dependence: they **pollute shared state and break developer tests**. That happens far more for focal methods of flaky tests than of non-flaky ones.

**All experiments, generated tests only** (each row is one experiment; P = polluter, S = state-setter, V = victim, B = brittle):

| Experiment | What it is | Started with | Built | Tests generated | Pass alone (files) | OD-checked so far (files) | P | S | V | B | Flaky alone (rerun / NonDex / NIO) | Developer tests broken | ...known OD tests in IDoFT |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Main, flaky side | ChatUniTest, focal methods of flaky tests | 144 flaky tests → 53 focal methods, 32 modules | 51 | 47 | 23 | 18 | **3** | 1 | 0 | 0 | 0 | **35** | 11 |
| Main, non-flaky side | ChatUniTest, focal methods of non-flaky tests | 156 focal methods, 13 modules | 141 | 132 | 94 | 78 | 0 | 1 | 0 | 0 | 0 | 0 | - |
| Prompt, current | ChatUniTest's own prompt, 5 focal methods ×3 | 15 runs, 5 modules | 15 | 13 | 8 | 8 | **4** | 2 | 0 | 0 | 0 | **507** | 70 |
| Prompt, cleanup | + "restore shared state" | 15 runs, 5 modules | 15 | 15 | 12 | 12 | 0 | 1 | 0 | 0 | 0 | 0 | - |
| Agent A | agent, doesn't read existing tests | 15 runs, 5 modules | 15 | 15 | 13 | 8 (dubbo running) | **2** | 3 | 0 | 0 | 0 | **11** | 11 |
| Agent F | agent, reads existing tests first | 15 runs, 5 modules | 15 | 14 | 11 | 6 (dubbo, scimono on machine B) | 0 | 3 | **3** | 0 | 0 | 0 | - |

**What the columns mean:**
- **Tests generated:** the generator produced a test file.
- **Pass alone:** the file passes when run alone with plain `mvn test`. Only these files are checked further.
- **OD-checked:** iDFlakies ran 20 rounds and `od_confirm` confirmed what it flagged. On the main study, iDFlakies couldn't run on 5 + 16 files (mixed JUnit 4/5, its runner crashed, a JVM didn't exit).
- **P, polluter:** the generated file makes developer tests fail.
- **S, state-setter:** the generated file makes a brittle developer test pass. It's harmless; all are in ormlite.
- **V, victim:** the generated file itself fails after a developer test. Only agent F's 3 marineapi files.
- **B, brittle:** the generated file fails alone and passes after another test. None; files that fail alone are excluded anyway.
- **Developer tests broken:** distinct developer tests that fail after a polluter.
- **...known OD tests in IDoFT:** of those, how many IDoFT's `pr-data.csv` already lists as order-dependent (`OD-Vic`).
  - Some more are listed under other categories: main study 12 (NDOD/NOD), prompt current 19 (ID/NDOD/NOD).
  - The rest are new: main study 12 of 35, prompt current 418 of 507, agent A 0 of 11.

**Modules per experiment** (module = one project at one commit, as a ReproFlake archive):

| Experiment | Started with | Focal method found | Built | ≥1 test generated | ≥1 file passes alone | OD-checked | With a polluter |
|---|---|---|---|---|---|---|---|
| Main, flaky side | 80 modules (44 projects, 144 flaky tests) | 32 | 30 | 27 | 15 | 11 | 3 (dubbo ×2, marineapi) |
| Main, non-flaky side | the 13 modules where the flaky side generated tests | 13 | 12 | 12 | 11 | 8 | 0 |
| Prompt, current | 5 focal methods chosen from the main study (2): dubbo ×2, marineapi, ormlite, scimono | 5 | 5 | 5 | 4 (no marineapi) | 4 | 3 (dubbo ×2, ormlite) |
| Prompt, cleanup | same 5 | 5 | 5 | 5 | 5 | 5 | 0 |
| Agent A | same 5 | 5 | 5 | 5 | 5 | 3 (dubbo running) | 1 (marineapi) |
| Agent F | same 5 | 5 | 5 | 5 | 5 | 2 (dubbo, scimono on machine B) | 0 |

(2) **How the 5 were chosen:**
- From the flaky side, the 3 confirmed polluters: dubbo aa9f16e `ChangeTelnetHandler#telnet`, dubbo 628ad77 `PortTelnetHandler#telnet`, marineapi `AISMessageFactory#create`.
- From the non-flaky side, 2 files that write fields by reflection: ormlite `LoggerFactory#setLogBackendType` (a static field, not restored) and scimono `GroupRequest#createGroup`.

**Where modules drop out:**
- 80 → 32: no single focal method could be identified for the flaky test.
- 32 → 30: graylog2-server (2 commits) doesn't build.
- 30 → 27: ChatUniTest kept no test.
- 27 → 15: no generated file passes alone.
- 15 → 11: iDFlakies can't run (mixed JUnit 4/5, its runner crashed, a JVM didn't exit).

**In short:** across all experiments, 9 generated files are polluters. They break 35 + 507 + 11 developer tests, and most of those weren't known as order-dependent before. The main-study difference, 3 of 18 vs 0 of 78 polluter files, is significant (Fisher p = 0.006).

### RQ1.1. How does the context available to the generator affect flakiness? (reworded)

ChatUniTest is a fixed pipeline that sees only the focal class. Agent A explores the project but isn't told to read the existing tests. Agent F is told to read the existing tests first. Same 5 focal methods; 3 runs per method for the agents.

| | ChatUniTest (current prompt) | Agent A | Agent F |
|---|---|---|---|
| Polluter files | 4 of 8 (2 of 3 without dubbo) | 2 of 8 (dubbo pending) | 0 of 6 (dubbo, scimono pending) |
| Generated victim files | 0 | 0 | 3 of 6 (all marineapi) |
| Files with an `@After` that has code | 0% | 0% | 36% |
| marineapi focal-method line coverage | 0% (main-study file; the current-prompt files fail alone) | 18% | 82% |

**Did they actually read the tests?** Both agents have the same tools (shell, so they can read any file).
- **A read 0 existing test files in all 15 runs.**
- **F read them in 11 of 15 runs:** all 9 marineapi, ormlite and scimono runs, but only 2 of the 6 dubbo runs. Its other 4 dubbo runs wrote a test straight away and ignored the instruction.

**Answer:** reading the existing tests stops the agent from polluting, but makes it copy their weaknesses.
- F copied the singleton setup of `AISMessageFactoryTest`, which is already a known IDoFT victim, so F's tests became victims of the same developer polluter.
- So the flakiness moves from polluter to victim instead of disappearing.

### RQ2. How do the causes of flakiness differ between AI-generated and developer-written tests?

**Developer flaky tests have 4 kinds of causes; generated tests show only 1, order-dependence.**

| Flakiness category (ReproFlake) | Developer flaky tests we started from (53 used, of 144) | Generated tests |
|---|---|---|
| **OD**, order-dependent (polluter / victim) | 23 | **all of it**: 9 polluter files, 3 victim files |
| **ID**, implementation-dependent (e.g. HashMap order; found by NonDex) | 17 | 0 (NonDex, 100 seeds) |
| **TD**, time-dependent | 9 | 0 (no sleep or clock use in any generated file) |
| **NIO**, not idempotent (fails when run twice in a row) | 4 | 0 (NIO, 100 runs) |

**One mechanism behind all the generated cases:**
- A shared singleton or static field is replaced with a mock or an empty object, and never restored. None of the polluters has an `@After`.
- In 8 of the 10 polluter files this is done by reflection; in the other 2 (ormlite), by the class's own static setter.

**Why the generated tests differ** (static counts):
- They are small and mock-heavy: about 2 test methods per file; 66–100% of files use mocks, against 29% of the developer classes.
- They avoid the things behind TD and ID flakiness: no sleep, clock, randomness or file I/O. Developer classes on the flaky side use sleep in 18% and the clock in 6%.
- But they write internal state by reflection in 43–100% of files; developer classes in 0%.

**New or reproduced problems?** Both.
- **New:** 518 distinct developer tests are broken across experiments, and most of them aren't known as OD in IDoFT. Main study: 11 of 35 are known OD. Prompt current: 70 of 507.
- **Reproduced:** the same focal methods polluted again whenever they were regenerated (dubbo, marineapi, ormlite). And agent F copied the setup of a developer test that's already a known victim, so its own tests became victims.

### RQ3. To what extent can prompt modifications reduce flakiness?

**A one-line cleanup instruction removed all pollution in our sample.** It says: restore any static field, singleton or reflection-set field in `@After`.

| | Current prompt | Cleanup prompt |
|---|---|---|
| Polluter files | 4 of 8 | **0 of 12** (Fisher p = 0.014) |
| Developer tests broken | 507 | **0** |
| Files with an `@After` that has code | 0% | 92% |
| Line coverage of the focal method (marineapi / ormlite / scimono) | 0% / 100% / 100% (1) | 0% / 67% / 100% |

(1) The current prompt's marineapi files fail alone, so the main study's marineapi file is used.

- **The mechanism is confirmed:** adding a restoring `@After` by hand to the main study's 3 polluters made all 35 of their victims pass.
- **Cost:** coverage stayed the same, except ormlite (100% → 67%).
- **Agent variant:** telling the agent to read the existing tests first (F) also brought polluters to 0. But it copied an existing weakness: 3 victim files (RQ1.1).
- **Caveat:** 5 focal methods with 3 runs each, so treat this as case evidence.

## 1. Main study: ChatUniTest on ReproFlake

**Generation (flaky side):**
- 144 flaky tests;
- 53 had a focal method identified;
- 47 generated files;
- **23 pass with plain `mvn test`** (48 methods).

**Non-flaky side:** focal methods of non-flaky developer tests, **94 files** (175 methods).

| | Flaky side (23 files) | Non-flaky side (94 files) | Fisher (two-sided) |
|---|---|---|---|
| Flaky alone (rerun / NonDex / NIO) | 0 | 0 | |
| Files iDFlakies could check (1) | 18 | 78 | |
| Files with a real **polluter** | **3 of 18 (17%)** | **0 of 78** | p = 0.006 |
| Files with a polluter or state-setter | 4 of 18 (22%) | 1 of 78 (1%) | p = 0.004 |
| Generated methods that pollute or set state | 6/48 (12.5%) | 1/175 (0.6%) | p = 0.0004 |
| Developer tests broken | **35** (23 in IDoFT, 12 new) | 0 | |
| Generated tests that are themselves OD | 0 | 0 | |

(1) iDFlakies didn't fully run on 5 flaky-side and 16 non-flaky-side files: mixed JUnit 4/5 (not supported by iDFlakies), its test runner crashed, a test JVM didn't exit, or a timeout. Earlier tables counted these as clean (3 of 23 vs 0 of 94, p = 0.007).

**Funnel:**
- iDFlakies flagged 1,531 suspects on the flaky side and 1,325 on the non-flaky side.
- Confirmation left 37 and 3. One timing-dependent elastic-job developer test was then removed from each side as not OD, leaving 36 and 2.
- So iDFlakies alone overcounts a lot.

**Hadoop** (machine B): rerun, NonDex and NIO on 17 of its 23 methods found 0 flaky. iDFlakies wasn't run, because it's too slow on hadoop.

## 2. Root causes: every confirmed case

| Experiment | Subject | Generated test | What it does | Restores? | Developer tests broken |
|---|---|---|---|---|---|
| Main | dubbo aa9f16e | `ChangeTelnetHandler_telnet_0_1_Test` | `@Before`: reflection sets `DubboProtocol.INSTANCE` to a Mockito mock | no | 12 |
| Main | dubbo 628ad77 | `PortTelnetHandler_telnet_0_0_Test` | same | no | 12 |
| Main | marineapi | `AISMessageFactory_create_0_3_Test` | reflection sets the singleton's `parsers` map to an empty map | no | 11 |
| Prompt, current | dubbo aa9f16e (row 2) | `ChangeTelnetHandler_telnet_0_1_Test` | `DubboProtocol.INSTANCE` set to a mock | no | 14 |
| Prompt, current | dubbo 628ad77 (row 4) | `PortTelnetHandler_telnet_0_0_Test` | same | no | 15 |
| Prompt, current | ormlite (row 11) | `LoggerFactory_setLogBackendType_6_2_Test` | static log backend set to a mock; hidden by JUnit's method order | no | 478 |
| Prompt, current | ormlite (row 12) | `LoggerFactory_setLogBackendType_6_0_Test` | static log backend set to a mock | no | 478 (the same tests) |
| Agent A | marineapi (rows 7, 8) | `AISMessageFactory_create_0_0_Test` | `@Before`: `parsers` set to an empty map | no | 11 each |

- **State-setters (harmless):** all in ormlite. They make the brittle developer test `LoggerTest` pass, because loading `LoggerFactory` initialises a global log level. That's a defect in the developer test.
- **Victim:** agent F's marineapi tests fail after the developer test `SentenceFactoryTest#testRegisterParserWithAlternativeBeginChar`.
- Details are in `ROOT_CAUSES.md`.

## 3. Prompt experiment: current vs cleanup prompt (5 focal methods)

Both prompts were run on the same 5 focal methods: dubbo ×2, marineapi, ormlite, scimono. The cleanup prompt adds: restore any shared state you change, in `@After`.

| | Current prompt | Cleanup prompt |
|---|---|---|
| Files that pass alone (checked) | 8 | 12 |
| Flaky alone (rerun / NonDex / NIO; 4,800 runs each) | 0 | 0 |
| **Confirmed polluters** | **4 of 8** (dubbo rows 2 and 4, ormlite rows 11 and 12) | **0 of 12** |
| Developer tests broken | **507** (89 in IDoFT, 418 new) | 0 |
| Files with an `@After` that has code | 0% | 92% |

Fisher, 4/8 vs 0/12: **p = 0.014**.

**Note on row 11:** `od_confirm` first missed it, because JUnit ran the cleaning method last. A method-by-method re-check confirmed it. The same re-check over about 1,800 rejected cases elsewhere found nothing else.

## 4. Agent experiment (gpt-4o agent, step limit 40, 3 runs per focal method)

- **A** = ChatUniTest's prompt plus "how to work" steps.
- **F** = A plus "read the project's existing tests first".

| Subject | A | F |
|---|---|---|
| dubbo ×2 (rows 1-6) | 5 of 6 generated files pass. iDFlakies flags rows 1-4 as polluters; **confirmation is still running** (done tonight or tomorrow). | 4 pass; row 3 made no tests. **Running on machine B** (about 1 day). |
| marineapi (rows 7-9) | **2 polluters** (11 developer tests each, all known in IDoFT). Row 9 breaks the same 11 but fails alone, so it's left out. | **0 polluters**, but **all 3 files are victims** of a developer polluter. F copied the singleton setup of a developer test that's an IDoFT known victim. |
| ormlite (rows 10-12) | 0 polluters; 3 harmless state-setters | 0 polluters; 3 harmless state-setters, and its `@AfterEach` restores the state |
| scimono (rows 13-15) | clean | machine B |
| Flaky alone (rerun / NonDex / NIO) | 0 | 0 |
| Cost (15 rows) | $3.88 | $3.85 |

**Takeaway:** reading the existing tests transfers their patterns, the bad ones as well as the good ones. Sample is small (A vs F polluters 2 of 8 vs 0 of 6, p = 0.47).

## 5. Test quality: coverage and mutation of the focal method

**How it's measured:**
- Each test class is run alone, and only the focal method is measured: JaCoCo line and branch coverage, and PIT mutation score (the share of the focal method's mutants the class kills).
- Generated = one generated file. Developer = the developer test class the focal method came from, which must pass 3 times alone.
- **PIT mutators:** STRONGER is PIT's standard set, a few mutants per method. ALL is every operator PIT has, more mutants, so a steadier score.

**Conclusion:**
1. **Usually equal:** in 58 of 72 focal methods, the generated class covers the same share of the focal method's lines as the developer class. Mutation score is equal in 42 of 67.
2. **When they differ, the developer tests are usually better:**
   - line coverage 86% vs 94% (Wilcoxon p = 0.004);
   - mutation score 76% vs 82% (p = 0.09, not significant), so there's no clear mutation difference overall.
3. **The flaky side is weakest:** generated tests for focal methods of flaky tests reach 66% line coverage and 39% mutation score, against 90% and 79% on the non-flaky side (p < 0.001). The polluting files are in the weak group, so pollution doesn't come with better tests.
4. **Context helps quality:** on the 3 measured focal methods, agent F matched or beat the developer tests (marineapi 82% / 89% vs 64% / 67%), while ChatUniTest got 0% on marineapi. That's a case result: 5 focal methods, dubbo not measured yet.
5. **Cleanup costs little:** coverage stays the same, except ormlite (100% → 67%).

### 5.1 Main study: generated vs developer (JaCoCo + PIT; 184 of 189 items measured)

| Tests | Files | Line | Branch | Mutation STRONGER | Mutation ALL |
|---|---|---|---|---|---|
| Generated, flaky side | 19 | 66% | 55% | 39% | 49% |
| Generated, non-flaky side | 94 | 90% | 81% | 79% | 82% |
| Developer (class of the source test) | 74 | 94% | 87% | 82% | 87% |

**Paired: the same focal method (generated mean vs developer class)**

| Metric | Pairs | Generated | Developer | Generated higher / same / developer higher | Wilcoxon p |
|---|---|---|---|---|---|
| Line | 72 | 86% | 94% | 3 / 58 / 11 | **0.004** |
| Branch | 36 | 81% | 87% | 4 / 26 / 6 | 0.17 |
| Mutation STRONGER | 67 | 76% | 82% | 11 / 42 / 14 | 0.09 |
| Mutation ALL | 69 | 79% | 87% | 13 / 40 / 16 | 0.05 |

Generated tests are weaker on the flaky side than on the non-flaky side: line p = 0.002, mutation p < 0.001 (Mann-Whitney).

### 5.2 The 5 focal methods (prompt and agent experiments)

| Subject | Developer | ChatUniTest current | ChatUniTest cleanup | Agent A | Agent F |
|---|---|---|---|---|---|
| marineapi: line / mutation | 64% / 67% | 0% / 0% | 0% / 0% | 18% / 17% | **82% / 89%** |
| ormlite: line / mutation | 100% / 50% | 100% / 100% | 67% / 67% | 100% / 100% | 100% / 100% |
| scimono: line / mutation | 100% / 100% | 100% / 100% | 100% / 100% | 100% / 100% | 100% / 100% |

Mutation here is STRONGER. Dubbo is still pending.

## 6. What the generated tests contain (static counts, share of files)

| Group | Files | Polluter | Mocks | Writes a field by reflection | @After with code | Test methods / file | Lines / file |
|---|---|---|---|---|---|---|---|
| Developer (main study) | 79 | - | 29% | 0% | 21% | 17 | 243 |
| ChatUniTest main, flaky side | 23 | 3 of 18 checked | 78% | 43% | 4% | 2.1 | 73 |
| ChatUniTest main, non-flaky side | 94 | 0 of 78 checked | 66% | 43% | 0% | 1.9 | 75 |
| Prompt, current | 8 | 4 | 100% | 62% | 0% | 2.9 | 59 |
| Prompt, cleanup | 12 | 0 | 75% | 100% | **92%** | 2.1 | 61 |
| Agent A | 13 | 2 (dubbo pending) | 77% | 85% | 0% | 2.8 | 50 |
| Agent F | 11 | 0 (dubbo pending) | 45% | 55% | 36% | 3.5 | 51 |

- **The pattern:** generated tests mock a lot and write fields by reflection; developer tests never do. They almost never clean up, except with the cleanup prompt or agent F.
- The developer row combines the flaky and non-flaky sides, weighted by file count.
- Full table: `quality/STATIC_FEATURES.md`.

## 7. Still running or open

- **Agent A dubbo:** iDFlakies is on row 5 of 6 on this machine. Then confirmation, re-check and the other 3 checks follow.
- **Agent F dubbo and scimono:** running on machine B, about 1 day.
- **Dubbo coverage and mutation** (main study and the 5-subject set): runs after the agent dubbo check (port clash).
- **Caveats:**
  - small samples in the prompt and agent experiments (5 focal methods);
  - one LLM (gpt-4o);
  - 23 of 47 flaky-side files pass, so the polluter rate is computed on the files that pass.

## Where the data is

| Topic | File |
|---|---|
| Main study OD | `OD_RESULTS.md` |
| Root causes | `ROOT_CAUSES.md` |
| Prompt experiment | `prompt_experiment/IDFLAKIES_RESULTS.md` |
| Agent F | `agent_experiment/FLAKY_F_RESULTS.md`, `agent_experiment/F_VICTIMS_ANALYSIS.md` |
| Coverage and mutation | `quality/SUMMARY_MAIN.md`, `quality/SUMMARY.md` |
| Static features | `quality/STATIC_FEATURES.md` |
| Log of everything | `EXPERIMENTS.md` |
