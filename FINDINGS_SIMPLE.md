# Findings so far (simple version), 2026-10-26

## Words used

* **Focal method:** the project method the test is generated for.
* **Generated test file:** one test class written by ChatUniTest (one per generation run).
* **Generated test method:** one `@Test` method inside a generated test file.
* **Developer test method:** one `@Test` method the project's developers wrote.

## Setup

| | Flaky side | Non-flaky side |
|---|---|---|
| Developer test methods we started from | 23 (flaky, from ReproFlake) | 93 (non-flaky) |
| Focal methods | 22 | 93 |
| Generated test files | 23 | 94 |
| Generated test methods | 48 | 175 |

## 1. The generated tests are not flaky themselves

| | Flaky side | Non-flaky side |
|---|---|---|
| Generated test methods that are flaky on their own (rerun, NonDex, NIO, iDFlakies) | 0 of 48 | 0 of 175 |

## 2. But some generated tests break the developers' own tests

| | Flaky side | Non-flaky side |
|---|---|---|
| Generated test files that break developer tests | **3 of 23** | **0 of 94** |
| Focal methods those files were generated for | 3 of 22 | 0 of 93 |
| Developer test methods they break | **35** | 0 |

Fisher's exact test (two-sided) on the generated test files, 3 of 23 vs 0 of 94: p = 0.007.

The 3 generated test files:

| Generated test file | Developer test methods it breaks |
|---|---|
| dubbo `ChangeTelnetHandler_telnet_0_1_Test` | 12 |
| dubbo `PortTelnetHandler_telnet_0_0_Test` | 12 |
| marineapi `AISMessageFactory_create_0_3_Test` | 11 |
| **Total** | **35** |

## 3. Why

All 3 do the same thing:
* they use **reflection** to replace a shared object (a singleton) with a fake or empty one;
* they **never put it back**;
* developer tests that run later get the fake one and fail.

ChatUniTest's prompt tells the model it may use reflection, but says nothing about cleaning up.

## 4. Restoring the object fixes it

* **The check:** I added an `@After` that puts the original object back, in copies of the 3 files.
* **The result:** all 35 developer test methods pass again.

**Details:** `OD_RESULTS.md` (full analysis), `RESULTS_TABLE.md` (all counts).
