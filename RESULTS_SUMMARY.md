# ChatUniTest on ReproFlake: results so far

Status as of 2026-10-22. Everything below is in `/home/srafi/only_scripts`.

## 1. The overall numbers

| Stage | Count |
|---|---|
| Flaky tests in `result_matched.csv` | 144 |
| Focal method identified (rest ambiguous or unresolved in `prepare`) | 53 |
| Module built (both graylog2-server rows fail) | 51 |
| ChatUniTest generated a test that compiled and passed its own check | **47** |
| Pass fully with plain `mvn test` (module's own setup) | **22** |
| ...plus pass fully only with the Mockito fix (eco-schema-catalog row 12) | **1** → **23** |
| Some test methods in the file pass, some fail | 10 |
| All test methods in the file fail | 12 |
| No tests ran (dubbo-common surefire setup issue) | 2 |

Caveats among the 23 passing files:
- **YARN-9405 (part 1, row 141)** passes but tests a different overload than the one the flaky test calls.
- **CAMEL-12025 (part 2, row 121)** passes but which overload it targets is ambiguous.

So there are about **21 clear-cut cases** where a generated test passes under plain Maven and targets the right method. Several come from the same module (ormlite has 5 passing rows), so the number of distinct focal methods is somewhat lower.

## 2. Generation (ChatUniTest)

Two runs: part 1 (27 rows, `out_part1/`) and part 2 (26 rows, `out_part2/`).

| | Part 1 | Part 2 | Total |
|---|---|---|---|
| generated | 25 | 22 | **47** |
| no_tests (built fine, ChatUniTest kept nothing) | 1 | 3 | 4 |
| build_failed | 1 | 1 | 2 |

- **build_failed:** graylog2-server, both copies (036bdb5 and 27269f2). The frontend `yarn run build` step (frontend-maven-plugin) fails in `graylog-web-plugin`.
- **no_tests:** fury-core (part 1, row 15), blade-core (part 2, row 13), shenyu-admin (part 2, row 20), elasticjob-tracing-rdb (part 2, row 77). They used up all repair rounds without a passing version. No API errors.

### How ChatUniTest validates
- `MaxRounds 5`, `Stop when success true`. Each round it compiles and runs the test; on failure it sends the error back to the model for a fix.
- 41 rows ended with "compile and execute successfully".
- 6 rows (part 1: 55, 59, 73; part 2: 45, 47, 57) ended with "Processed test … generated successfully". This is most likely ChatUniTest removing still-failing test methods and keeping the passing ones (not confirmed in its source).

### Overload matching
- 2 rows target the wrong overload (`no_test_for_called_overload`): ormlite (part 1, row 68) and YARN-9405 (part 1, row 141).
- 3 rows are `ambiguous`: dubbo-common (part 1, row 11) and CAMEL-12025 (part 2, rows 121 and 130).
- 6 rows reach the focal method only through reflection: crane4j, apollo-openapi, dubbo-common (83c466e), nacos, elasticjob294, HADOOP-12588.

### Setup details
- **Patched ChatUniTest core:** `~/.m2/repository/io/github/zju-aces-ise/chatunitest-core/2.1.1/` (installed Oct 21). The patch is `/home/srafi/chatunitest-junit4/chatunitest-core-junit4.patch`.
- **JUnit 4 prompts:** `/home/srafi/chatunitest-junit4/prompts-junit4`. All 32 JUnit 4 rows used them (each `maven.log` shows the prompt path), and every generated JUnit 4 file uses `org.junit.Test` with no JUnit 5 imports. The other 15 rows ran in JUnit 5 mode with the stock prompts.
- **Prompt wording:** JUnit 5 says "create a complete unit test … ensuring to cover all branches". JUnit 4 says "create a whole unit test using JUnit 4, ensuring optimal branch and line coverage". So each file holds several `@Test` methods for the same focal method.
- **Typos in `prompts-junit4/initial_system.ftl`:** "Signatures of **r** methods" (other) and "**followotheing**" (following).
- **Mockito:** the prompts ask the model to use Mockito, and the patched core's `JUnitFlavor.addTestImports` inserts Mockito imports into every generated file. All 49 files import Mockito; 39 actually use it. The generator's `ensure_mockito` adds test-scoped `mockito-core` 4.11.0 to the module pom when it's missing (19 of the 47 rows). The original pom is kept as `pom.xml.batch_orig`.
- **File names:** `<Class>_<method>_<N>_<M>_Test.java`, where N is the overload number of the focal method. Rows with 2 files had several overloads of the same name.

## 3. Replaying the generated tests with plain `mvn test`

Script: `replay_generated.py`. For each generated row it:
1. copies the test into the module's `src/test/java/`, refusing to overwrite an existing file;
2. runs `mvn test -pl <module> -Dtest=<generated class>` with the row's JDK and Maven without the KebabPizza `lib/ext` extension;
3. if compilation or dependency resolution fails, merges the archive's `Flakym2/` into `~/.m2` (adds missing files only) and retries once;
4. removes the copied test and its compiled classes, and checks the pom is byte-for-byte unchanged (a backup is in `rows/<id>/pom.xml.before`).

Nothing was left behind in any checkout and no pom changed. One side effect: for dubbo-common row 11, the Flakym2 fallback added 1,796 files to `~/.m2`, without replacing anything.

### Run 1: module's own setup (`genreplay_out/`)

| Result | Rows |
|---|---|
| passed (all methods) | 22 |
| partly_failed | 7 |
| failed | 16 |
| no_tests_ran | 2 |

Passing rows: part 1 rows 3, 6, 17, 32, 40, 46, 49, 53, 55, 59, 70, 71, 72, 73, 141; part 2 rows 16, 43, 45, 47, 57, 119, 121.

### The Mockito clash
About 14 of the failures were two Mockito setups colliding, not bad tests:
- **Old byte-buddy pinned by a parent or Spring Boot**, together with the added Mockito 4.11, gives `Could not initialize plugin: MockMaker`. Affected: eco-schema-catalog (2 rows), apollo-openapi, wildfly-naming (5 rows).
- **`mockito-all` 1.x on the classpath ahead of Mockito 4.11** gives `NoSuchMethodError: RunnerFactory.createStrict`. Affected: CAMEL-12025 (4 rows, mockito-all 1.10.19) and HADOOP-12588 (mockito-all 1.8.5).
- ormlite row 69: `MockMaker … does not support`.

### Run 2: with `--mockito-fix` (`genreplay_fix_out/`, the 25 non-passing rows only)
For the run only, the pom gets Mockito 4.11, `mockito-inline` 4.11 (so final classes can be mocked), byte-buddy 1.12.19 and objenesis 3.3 declared first in `<dependencies>`. The pom is restored and checked against its hash afterwards.

| Result | Rows |
|---|---|
| passed | 1 (eco-schema-catalog row 12: 0/3 → 3/3) |
| partly_failed | 9 |
| failed | 12 |
| no_tests_ran | 3 |

- **Improved:** eco-schema-catalog row 25 (0/3 → 1/3), wildfly rows 66 and 67 (0/3 → 1/3).
- **Worse:** commons-lang row 1 went from 2/3 to "no tests ran". Adding `mockito-junit-jupiter` 4.11 probably brings a newer JUnit 5 API than the project's surefire and engine handle. Use its original 2/3 result.
- Note: these rows ran with a slightly different test classpath than the project's own, so keep them separate from the 22 that passed with the unchanged setup.

## 4. Partly passing files (10)

A row is one generated test file. Each file has several `@Test` methods, and "partly" means some of them pass and some fail.

| Row | Module | Passed | Why the failing methods fail |
|---|---|---|---|
| part 1 row 1 | commons-lang | 2/3 (normal setup) | expected `NullPointerException`, got a different exception |
| part 1 row 25 | eco-schema-catalog | 1/3 (fix) | expected `IllegalArgumentException`, got a reflection exception |
| part 1 row 41 | wikidata-toolkit | 7/8 | expected `IOException`, got a Mockito exception |
| part 1 row 52 | elastic-job-lite-core | 1/2 | `ClassCastException` (Integer cast to String) |
| part 1 row 63 | wikidata-toolkit | 2/3 | expected `IOException`, got a Mockito exception |
| part 1 row 123 | CAMEL-17188 | 2/3 | failed assertion |
| part 2 row 34 | nacos | 5/10 | expected `IndexOutOfBoundsException`, got a reflection exception |
| part 2 row 66 | wildfly-naming | 1/3 (fix) | expected `NameNotFoundException`; `WrongTypeOfReturnValue` (Mockito misuse) |
| part 2 row 67 | wildfly-naming | 1/3 (fix) | failed assertion; NPE |
| part 2 row 130 | CAMEL-12025 | 1/2 | "No consumers attached to the stream" |

Example (part 1, row 41): `DirectoryManagerFactory_createDirectoryManager_1_2_Test` has 8 tests. Only `testCreateDirectoryManagerInvocationTargetExceptionWithIOException` fails.

## 5. Files where every test fails (12)

All of these are problems in the test itself, not the environment (results after the Mockito fix).

**1. The test doesn't set up the object properly, so the real code hits a null (5)**
- wildfly part 2 row 50 (`rebind`): crashes in `buildServiceName`, because the service name was never set.
- HADOOP-12588 part 2 row 137 (`publishMetricsNow`): crashes in `snapshotMetrics`, because the metrics system wasn't started or configured.
- elasticjob003 part 2 row 79 (`addJobStatusTraceEvent`): NPE in the test's own `setUp`.
- CAMEL-12025 part 2 row 117 (`to`): NPE inside the focal method.
- apollo part 2 row 5 (`findItemsByNamespace`): NPE in the main test (its other tests also hit groups 2 and 4).

**2. Reflection on private methods that don't exist in this version (2)**
- CAMEL-12025 part 2 row 122: `setSubscribers(ConcurrentMap)` doesn't exist.
- apollo part 2 row 5: `checkNotEmpty(String, String)` doesn't exist.

**3. The test misunderstands what the code does (4)**
- wildfly part 2 rows 54 and 60 (`bind`): the store is read-only unless set up as writable ("WFLYNAM0043: Naming context is read-only"). Row 54 also calls `when()` on a non-mock (`MissingMethodInvocationException`).
- CAMEL-12025 part 2 row 128: "No consumers attached to the stream".
- elasticjob294 part 2 row 76 (`registerTypeAdapter`): failed assertion.

**4. The test hides the real error (2)**
- ormlite part 1 rows 68 and 69 (`dropSchema`): the test catches the exception and fails with "Reflection issues: null" or "Exception during test setup: null". Row 68 also targets the wrong overload.

**Open question:** these passed ChatUniTest's own check but fail completely under Maven. ChatUniTest's runner probably set things up differently (classpath, or static state left over from earlier repair rounds in the same process). Running one (e.g. wildfly row 60) several times would show whether it ever passes, which would itself be a sign of flakiness.

## 6. No tests ran (2)

dubbo-common part 1 row 11 and part 2 row 23 (both JUnit 5 mode): surefire 3.0.0-M5 prints `NOOP` and runs 0 tests. This is a surefire or JUnit 5 setup issue in dubbo-common, not a test result.

## 7. Non-flaky test collection (`collect_nonflaky.py`)

Total: **72** (project, commit, module) units from the 144 rows of `result_matched.csv` (80 distinct archives).

| | Modules |
|---|---|
| Non-flaky list collected | **26** |
| Archive on disk but build fails (graylog2-server) | 1 |
| No archive on disk yet | **45** |

- **About 13,850 non-flaky tests so far:** `nonflaky_out/` (from `out_part1/extracted`) has 8,308 and `nonflaky_out2/` (from `out_part2/extracted`) has 6,880. Three modules ran in both (elastic-job-lite-core, dubbo-common, dubbo-rpc-dubbo); for those, only tests both runs agree on are counted.
- **dubbo-rpc-dubbo differs between runs:** 23 not-passed in run 1, 1 in run 2 (different archive copies). Worth checking.
- **Why 45 modules have no archive:** the generation runs only downloaded the 32 archives needed for the 53 rows with a resolved focal method, and `--delete-zips` removed the zips after unpacking. The other 48 archives were never downloaded. Neither non-flaky run used `--download`.
- **The two output folders are not merged yet.**
- **No LLM is used in this step:** it only runs `mvn test`, reads the surefire reports and drops tests listed as flaky in `result_matched.csv` or IDoFT, or that didn't pass cleanly.

Command for the remaining modules (downloads from Zenodo; expect hours):
```
python3 collect_nonflaky.py --csv result_matched.csv \
  --also-look-in out_part1/extracted --also-look-in out_part2/extracted \
  --out ./nonflaky_out --archives ./zips --download --delete-zips \
  2>&1 | tee nonflaky_step3.log
```

## 8. Suggested next step for the flakiness question

Keep ChatUniTest's output as it is (several tests per file) and measure flakiness **per test method**:
1. Use the tests that pass under plain `mvn test`.
2. Rerun each class 50–100 times unchanged; a method that sometimes passes and sometimes fails is non-order-dependent (NOD) flaky.
3. Shuffle method order (`-Dsurefire.runOrder=random` or iDFlakies); failing only in some orders means order-dependent.
4. Run NonDex, since many original flaky tests are `ID` (implementation-dependent) with NonDex seeds.
5. Optionally run each method on its own (`-Dtest=Class#method`) to separate order-dependent failures.

## 9. Where things are

| Path | What |
|---|---|
| `out_part1/generation_results.csv`, `out_part2/generation_results.csv` | generation result per row |
| `out_part*/runs/row_NNN/` | ChatUniTest logs (`maven.log`) and generated tests (`tests/`) |
| `genreplay_out/results.csv`, `methods.csv` | replay with the module's own setup |
| `genreplay_fix_out/results.csv`, `methods.csv` | replay with `--mockito-fix` |
| `genreplay_out*/rows/<id>/` | logs, surefire XML, pom backup per row |
| `nonflaky_out/`, `nonflaky_out2/` | non-flaky collection (`nonflaky.csv`, `units.csv`, `all_tests.csv`) |
| `replay_generated.py` | replay script |
| `collect_nonflaky.py` | non-flaky collection script |
