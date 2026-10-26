# Results table (2026-10-26)

ChatUniTest (gpt-4o), one generation per focal method.
* **Flaky side:** focal methods of ReproFlake flaky tests.
* **Non-flaky side:** focal methods of non-flaky developer tests.

Every cell counts the units named in its column: generated **test files** (classes), **focal methods**,
generated **test methods**. A file or focal method is counted when at least one of its test methods is.

## Table 1: generated tests

| | Flaky: files | Flaky: focal methods | Flaky: test methods | Non-flaky: files | Non-flaky: focal methods | Non-flaky: test methods |
|---|---|---|---|---|---|---|
| Generated | 23 | 22 | 48 | 94 | 93 | 175 |
| Rerun (100×): flaky | 0 | 0 | 0 | 0 | 0 | 0 |
| NonDex: flaky | 0 | 0 | 0 | 0 | 0 | 0 |
| NIO: flaky | 0 | 0 | 0 | 0 | 0 | 0 |
| NIO: inconclusive | 0 | 0 | 0 | 1 | 1 | 2 |
| iDFlakies: completed 20 rounds | 18 | 17 | 40 | 78 | 77 | 152 |
| iDFlakies: not completed (not run / incomplete / timeout) | 5 | 5 | 8 | 16 | 16 | 23 |
| iDFlakies: flagged the generated test as OD | 5 | 5 | 5 | 2 | 2 | 2 |
| Confirmed OD with M8 | **0** | **0** | **0** | **0** | **0** | **0** |
| iDFlakies guess: suspected polluter | 12 | 12 | 25 | 10 | 10 | 23 |
| **Confirmed polluter or state-setter (M8)** | **4** | **4** | **6** | **1** | **1** | **1** |
| ...polluter (makes developer tests fail) | 3 | 3 | 5 | 0 | 0 | 0 |
| ...state-setter (makes a brittle developer test pass) | 1 | 1 | 1 | 1 | 1 | 1 |

## Table 2: developer tests

| | Flaky: test classes | Flaky: test methods | Non-flaky: test classes | Non-flaky: test methods |
|---|---|---|---|---|
| Source tests the focal methods came from | 17 | 23 | 66 | 93 |
| **Victims: fail after a generated polluter** | **10** | **35** | **0** | **0** |
| Brittle: pass after a generated state-setter | 1 | 1 | 1 | 2 |
| Removed: timing flake, not OD (`JobEventRdbSearchTest`) | 1 | 1 | 1 | 1 |

A developer test is counted per project version. The same test name in two dubbo commits counts twice.

## Which generated tests

| Side | Generated test file | Test methods confirmed | Role | Victim / brittle developer test methods |
|---|---|---|---|---|
| flaky | dubbo aa9f16e `ChangeTelnetHandler_telnet_0_1_Test` | 3 | polluter | 12 |
| flaky | dubbo 628ad77 `PortTelnetHandler_telnet_0_0_Test` | 1 | polluter | 12 |
| flaky | marineapi `AISMessageFactory_create_0_3_Test` | 1 | polluter | 11 |
| flaky | ormlite `SchemaUtils_createSchema_3_0_Test` | 1 | state-setter | 1 |
| non-flaky | ormlite `LoggerFactory_setLogBackendType_6_0_Test` | 1 | state-setter | 2 |

* **Methods per polluting file.** The bisection names one method per victim. In the dubbo files the
  pollution is in `@Before`, so every method of the class pollutes.
* **Read files first.** Comparing the sides, the file column is the reliable one: polluter files are
  3/23 vs 0/94 (Fisher's exact test, two-sided, p = 0.007).
* Details and root causes: `OD_RESULTS.md`.

## Table 3: what each of the 7 generated test methods does that causes it

| # | Side | Generated test method | Role | What in the test causes it | Effect on the developer tests |
|---|---|---|---|---|---|
| 1 | flaky | dubbo aa9f16e `ChangeTelnetHandler_telnet_0_1_Test#testTelnetWithEmptyMessage` | polluter | `@Before` puts a Mockito mock into the static singleton `DubboProtocol.INSTANCE` by reflection (`setAccessible(true)`, `set(null, mock)`), with no `@After` to restore it. The method body itself only calls `telnet(channel, "")`. | Developer tests call `DubboProtocol.getDubboProtocol().export(...)`, get the mock, and nothing is exported: `ChangeTelnetHandlerTest` (3) gets "No such service", `ListTelnetHandlerTest` (5) gets an empty list |
| 2 | flaky | same class `#testTelnetWithNullMessage` | polluter | same `@Before`; body calls `telnet(channel, null)` | `InvokerTelnetHandlerTest` (3): service not found, assertion fails |
| 3 | flaky | same class `#testTelnetWithCancelService` | polluter | same `@Before`; body tests `cd /` | `PortTelnetHandlerTest#testListClient`: no server opens on port 20887, "Connection refused" |
| 4 | flaky | dubbo 628ad77 `PortTelnetHandler_telnet_0_0_Test#testTelnetWithIllegalPort` | polluter | same pattern: `@Before` puts a mock into `DubboProtocol.INSTANCE` by reflection (here to fake `getServers()`), no `@After`. The body only calls `telnet(channel, "abc")`. | 12 developer tests in `ChangeTelnetHandlerTest`, `InvokerTelnetHandlerTest`, `ListTelnetHandlerTest`, `PortTelnetHandlerTest`, failing as in 1-3 |
| 5 | flaky | marineapi `AISMessageFactory_create_0_3_Test#testCreate_NoParserForMessageType` | polluter | `@Before` gets the singleton `AISMessageFactory.getInstance()` and replaces its private `parsers` map with an empty `HashMap` by reflection, to reach the "no parser" error path. Never restored. | Every later developer test that parses an AIS message gets "no parser for message type 1": `AISMessageFactoryTest` (2), `AbstractAISMessageListenerTest` (9) |
| 6 | flaky | ormlite `SchemaUtils_createSchema_3_0_Test#testCreateSchemaSQLException` | state-setter | Calls `SchemaUtils.createSchema(...)`. That loads the class `SchemaUtils`, whose `static Logger logger = LoggerFactory.getLogger(...)` initialises `LoggerFactory`, which reads `simplelogging.properties` and sets the global log level from `OFF` to "no filter". Nothing wrong in the test itself. | `LoggerTest#testNoCurliesButArgs` fails alone (log level `OFF`, so nothing is logged) and passes after it |
| 7 | non-flaky | ormlite `LoggerFactory_setLogBackendType_6_0_Test#testSetLogBackendTypeNotAvailable` | state-setter | Calls `LoggerFactory.setLogBackendType(...)`, which initialises `LoggerFactory` the same way as 6. (Its `@BeforeEach` also writes the static `logBackendFactory` by reflection, but that has no effect here.) | `LoggerTest#testNotEnoughArgs` and `#testShouldCallToString` fail alone and pass after it |

* **Polluters (1-5):** the cause is always in `@Before`, not in the test method body. It replaces a
  shared singleton (or its contents) by reflection, and never restores it.
* **State-setters (6-7):** they only load `LoggerFactory`. The defect is the developer test `LoggerTest`,
  which depends on the log level being set by some earlier test.
