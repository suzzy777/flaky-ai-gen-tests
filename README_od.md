# Detecting OD Flakiness in ChatUniTest-Generated Tests Using ReproFlake

## Goal

Use the existing **ReproFlake OD Docker environment** to:

1. select an OD subject from the CSV files;
2. find the corresponding ReproFlake ZIP;
3. generate a new test with ChatUniTest;
4. run that generated test together with the other tests in the module under many random test orders;
5. detect whether the **generated test** becomes order-dependent.

The main pipeline is:

```text
CSV -> ReproFlake ZIP -> OD Docker
    -> ChatUniTest generates test
    -> generated test + existing tests
    -> ~100 random orders
    -> custom Surefire
    -> generated test passes in some orders and fails in others?
```

## Repositories

ReproFlake:

https://anonymous.4open.science/r/ReproFlake-C9E6/

Custom Surefire:

https://github.com/TestingResearchIllinois/maven-surefire

ChatUniTest setup:

https://github.com/devanshik19/srse-flakiness-research/blob/main/chatunitest_setup.md

---

## 1. Input CSVs

Use:

```text
master-sheet.csv
test_config.csv
```

From `master-sheet.csv`, only use:

```text
test_type = od
```

Match the subject using:

```text
master-sheet.container
        =
test_config.result_container
```

From the matching `test_config.csv` row, get:

```text
zip
module
url
```

The ReproFlake ZIP is expected at:

```text
data/<zip>.zip
```

If it is missing, download it from the `url` column.

If a `container` has no matching row in `test_config.csv`, report it and skip it.

---

## 2. ReproFlake OD files

These are the important existing OD files.

### `flaky_analysis_tool_od.sh`

This is the **outer/host script**.

It currently:

```text
unzip subject
    ->
prepare Flaky/ and Fixed/
    ->
build Dockerfile.od
    ->
mount source at /app/source
    ->
mount Flakym2/.m2 at /root/.m2
    ->
start container
    ->
call od_statistics_generator.sh
```

Use this script as the base for the new workflow.

For this experiment, it should eventually:

```text
read CSV subject
    ->
find ZIP + module
    ->
start the same OD Docker container
    ->
run ChatUniTest
    ->
run the new random-order detector
```

For OD detection of generated tests, we mainly need the **Flaky** source version. The current `Fixed`, `Fixed.patch`, `FlakyPssingOrder`, and `FixedPssingOrder` logic is not needed for the first version of this experiment.

---

### `Dockerfile.od`

This creates the OD Docker image used by ReproFlake.

Keep this environment.

It contains the custom Maven/Surefire support, including the Maven extension copied to:

```text
/usr/share/maven/lib/ext/
```

This is what allows commands such as:

```bash
-Dsurefire.runOrder=testorder
```

and test-order files.

ChatUniTest should also run **inside this same container**.

Pass the API key into the container, for example:

```bash
-e OPENAI_API_KEY="$OPENAI_API_KEY"
```

---

### `od_statistics_generator.sh`

This is the **current inner OD runner**.

Right now it compiles the module and repeatedly runs a known pair:

```bash
-Dtest="$precedingtest,$flakytest"
-Dsurefire.runOrder=testorder
```

This assumes the polluter/state-setter and victim are already known.

For our experiment, either modify this script or create a new script such as:

```text
od_detection_generator.sh
```

The new version should instead:

```text
compile module once
    ->
generate/read random order files
    ->
run each order
    ->
check the ChatUniTest-generated test result
    ->
save passing/failing orders
```

The Maven command becomes:

```bash
mvn test \
  -Dtest=/app/source/.od-orders/order-001.txt \
  -Dsurefire.runOrder=testorder \
  -pl "$MODULE" \
  $MVNOPTIONS
```

---

### `python-scripts/parse_surefire_report.py`

The current OD runner uses this to read Surefire XML results.

Reuse it if possible to determine whether the **generated test**:

```text
passed
failed
errored
```

for each order.

The full Surefire reports should still be saved.

---

### `Flaky/`

This is the project source mounted into:

```text
/app/source
```

ChatUniTest should generate the new test for this source tree.

The generated test should eventually be copied into the appropriate:

```text
<MODULE>/src/test/java/
```

directory.

---

### `Flakym2/.m2`

This contains the subject-specific cached Maven repository.

It is mounted as:

```text
/root/.m2
```

Keep using it so the project builds in the same ReproFlake environment.

---

### Coverage files

Files such as:

```text
coverage_generator.sh
jacocoagent.jar
jacococli.jar
modify_pom_for_coverage.sh
```

are not necessary for the initial OD-detection experiment.

They can remain in the infrastructure, but coverage collection does not need to be part of the first detector.

---

## 3. Run ChatUniTest inside the container

After the subject is mounted and built, generate a new test with ChatUniTest.

Follow:

https://github.com/devanshik19/srse-flakiness-research/blob/main/chatunitest_setup.md

Example:

```bash
mvn io.github.zju-aces-ise:chatunitest-maven-plugin:2.1.1:method \
  -DselectMethod="<CLASS>#<METHOD>" \
  -DapiKeys="$OPENAI_API_KEY" \
  -Durl=https://api.openai.com/v1/chat/completions \
  -Dmodel=gpt-4o-mini
```

ChatUniTest generates tests under approximately:

```text
<MODULE>/chatunitest-tests/
```

Copy the selected generated test into:

```text
<MODULE>/src/test/java/
```

First run the generated test **alone**.

If it does not compile or does not pass alone, record that and do not run the OD experiment for that generated test.

---

## 4. Create random orders

Get the existing tests from the same module and include the new generated test.

Example test set:

```text
package.TestA#test1
package.TestB#test2
package.GeneratedTest#generatedTest
package.TestC#test3
```

Generate about 100 shuffled order files:

```text
/app/source/.od-orders/order-001.txt
/app/source/.od-orders/order-002.txt
...
/app/source/.od-orders/order-100.txt
```

Each file contains one test per line.

Keep every order file so a failure can be reproduced later.

---

## 5. Run the orders

For every order:

```bash
mvn test \
  -Dtest=/app/source/.od-orders/order-001.txt \
  -Dsurefire.runOrder=testorder \
  -pl "$MODULE" \
  $MVNOPTIONS
```

Use the same Docker container and Maven environment for all orders.

Compile once before running the orders rather than recompiling before every order.

---

## 6. Detect OD behavior

For each order, record only the generated test's outcome:

```text
order-001 -> pass
order-002 -> pass
order-003 -> failure
order-004 -> pass
```

If the generated test:

```text
passes in at least one order
```

and:

```text
fails/errors in at least one other order
```

then mark it as a **candidate OD generated test**.

Save:

```text
container
generated test
passing order
failing order
Maven logs
Surefire reports
```

Then replay the saved passing and failing orders several times to confirm the order-dependent behavior.

---

## Final file flow

```text
master-sheet.csv
       |
       | container
       v
test_config.csv
       |
       | zip + module + url
       v
data/<zip>.zip
       |
       v
flaky_analysis_tool_od.sh
       |
       +--> Flaky/
       +--> Flakym2/.m2
       +--> Dockerfile.od
                 |
                 v
          ReproFlake OD container
                 |
                 +--> build project
                 |
                 +--> ChatUniTest generates test
                 |
                 +--> generated test -> src/test/java
                 |
                 +--> create ~100 order files
                 |
                 +--> od_detection_generator.sh
                           |
                           v
                 custom Surefire testorder
                           |
                           v
                 generated test results
                           |
                           v
                    candidate OD test
```

## Optional: iDFlakies

iDFlakies can also be tried inside the **same ReproFlake Docker container** after ChatUniTest generates and adds the new test.

For minimal changes, do not create another script. In `flaky_analysis_tool_od.sh`, after the ChatUniTest step, either:

```text
call modified od_statistics_generator.sh
    -> our ~100 custom-Surefire orders
```

or:

```text
run iDFlakies
    -> iDFlakies performs the OD detection
```

This lets both approaches use the same ReproFlake project, Docker environment, and ChatUniTest-generated test.

https://github.com/UT-SE-Research/iDFlakies
