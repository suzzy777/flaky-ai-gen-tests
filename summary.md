# Results per RQ, in short (2026-11-01)

Numbers as they stand in the paper now (paper/sections/results.tex).

## RQ1: How often are generated tests flaky?
- **Flaky alone:** none. No generated test method is flaky when run alone (100 reruns, NonDex, NIO), in any experiment (main study: 0 of 223 methods).
- **Polluters:** generated tests often break developer tests.
  - Main study (ChatUniTest; 117 files pass alone, 96 of them checkable by iDFlakies): **3 of 96 files are polluters**, and **10 of the 223 test methods in the 117 files pollute** (each method run alone before a victim, twice). Together they break 35 developer tests.
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
  - Line coverage 86% vs 94% (p = 0.003); mutation 74% vs 82% (p = 0.049, borderline).
  - Most pairs are identical.
- **Agents that read the tests write the strongest tests:** mutation F 90%, FCV 88%, vs ChatUniTest 59–68% (rows 1–15).
- **Cost per run:**
  - ChatUniTest ≤ $0.10; agent $0.26–0.47.
  - The cleanup sentence almost doubles the agent's cost ($0.26 → $0.46).

## One-line takeaway
Generated tests aren't flaky by themselves, but they break developer tests by changing shared state and not restoring it. One cleanup sentence mostly fixes that, but it makes generation harder and more expensive.
