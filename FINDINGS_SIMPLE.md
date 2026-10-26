# Findings so far (simple version), 2026-10-26

**Setup.** ChatUniTest (gpt-4o) generated unit tests for methods covered by
* **flaky** developer tests (from ReproFlake): 23 generated test files;
* **non-flaky** developer tests: 94 generated test files.

## 1. The generated tests are not flaky themselves

| | Flaky side | Non-flaky side |
|---|---|---|
| Generated test files | 23 | 94 |
| Flaky on their own (rerun, NonDex, NIO, iDFlakies) | 0 | 0 |

## 2. But some generated tests break the developers' own tests

| | Flaky side | Non-flaky side |
|---|---|---|
| Generated test files that break developer tests | **3 of 23** | **0 of 94** |
| Developer tests they break | **35** | 0 |

The difference is statistically significant (Fisher's exact test, p = 0.007).

The 3 files:

| Generated test file | Developer tests it breaks |
|---|---|
| dubbo `ChangeTelnetHandler_telnet_0_1_Test` | 12 |
| dubbo `PortTelnetHandler_telnet_0_0_Test` | 12 |
| marineapi `AISMessageFactory_create_0_3_Test` | 11 |

## 3. Why

All 3 do the same thing:
* they use **reflection** to replace a shared object (a singleton) with a fake or empty one;
* they **never put it back**;
* developer tests that run later get the fake one and fail.

ChatUniTest's prompt tells the model it may use reflection, but says nothing about cleaning up.

## 4. Restoring the object fixes it

* **The check:** I added an `@After` that puts the original object back, in copies of the 3 files.
* **The result:** all 35 developer tests pass again.

## 5. Other things we found

* **Two ormlite generated tests make a broken developer test pass.** The developer test `LoggerTest`
  fails on its own; the generated tests happen to switch logging on, so it then passes. This is not
  the generated tests' fault.
* **One developer test is a timing flake, not pollution.** elastic-job `JobEventRdbSearchTest` failed
  after many different tests. A 5 ms sleep made the failures disappear.
* **iDFlakies needs confirming.** It gave about 2,850 suspects; real runs confirmed only the cases above.
* **We fixed a bug in our own script.** It mixed up iDFlakies' passing and failing orders, and missed
  24 broken developer tests until it was fixed.

## 6. Running now

* **Prompt experiment:** we generate tests again for the 3 polluting methods (plus 2 JUnit 5 ones),
  with the current prompt and with a prompt that adds "restore anything you change in @After".
* **Question:** does the new prompt stop the pollution?
* **Results:** expected around 13:00-13:30.

**Details:** `OD_RESULTS.md` (full analysis), `RESULTS_TABLE.md` (all counts).
