# Next tasks: Claude experiments

Both tasks use Claude Opus 5.5, so all Claude results in the paper come from one model. Both students need their own `ANTHROPIC_API_KEY`, exported in the shell where they run the scripts. Each package's README has the details.

## Student 1: ChatUniTest with Claude Opus

**Package:** `chatunitest_claude_package.zip`

**What:** ChatUniTest generates tests for the focal methods from your focal step, the same way as the main study, but with Claude Opus instead of gpt-4o. Then the usual flakiness checks run on the tests that pass.

**Input:** your focal results folder (`focal_package/out`, with `focal_manifest_v2.csv` and `expansion_manifest.csv`). That gives 182 rows:
- rows 1-53: the 53 main-study tests that ChatUniTest with gpt-4o and the Opus agent already ran (same tests, same focal methods);
- rows 54-67: the 14 other main-set tests your focal step found;
- rows 68-182: the expansion. Tests that share a focal method get the same prompt, so each focal method is generated once (115 of 155).

**Steps** (in tmux, with JDK 8, 11 and 17, Maven and about 100 GB of disk):
1. `bash run.sh setup /path/to/focal_package/out`. It should print 182 rows.
2. Pilot: `bash run.sh generate --limit 1`. Send `out/generation_results.csv` and `out/proxy_usage.csv` before going on.
3. The 53: `bash run.sh generate --limit 53` (about $15-25).
4. Checks: `bash run.sh checks` (several days, no API cost). Send the results.
5. Later, once we have looked at the 53: `MAX_USD=40 bash run.sh generate --limit 67` for the 14 other main-set tests (about $5), then `MAX_USD=100 bash run.sh generate` for the expansion (about $30-50), and `bash run.sh checks` again.

The proxy stops sending requests once the total reaches $30. Raise it with `MAX_USD=...` as above.

**Send back:** `out/generation_results.csv`, `out/proxy_usage.csv`, `out/runs/`, `genreplay_out/`, `genreplay_fix_out/`, `out_checks/`, `task_csvs/` and the three log files.

## Student 2: agent conditions with Claude Opus

**First:** upload `out_checks_A/` from your Opus run on the 53 (agent_checks_package). We need it to confirm your polluter and victims and to find the row of the wildfly victim.

**Package:** `agent_variants_package.zip`

**What:** mini-swe-agent with Claude Opus in the four conditions (A, F, FC, FCV), 3 runs per focal method and condition, as in the gpt-4o experiment. Then the same flakiness checks as before.

**Which focal methods:** the ones where your Opus test on the 53 was order-dependent or writes static or singleton state, the same rule we used for gpt-4o. There are 9 so far, listed with reasons in `inputs/focal_rows.txt`: blade StaticOptions, two elastic-job JobRegistry ones, elastic-job GsonFactory, wikidata DirectoryManagerFactory, both dubbo telnet handlers, and the two marine-api victims. The wildfly victim is added at the end once we know its row.

**Steps** (in tmux, with JDK 8 and 11, Maven and about 10 GB of disk):
1. `bash run.sh setup`. It should print 9 focal methods x 3 runs = 27 rows.
2. A first test: `CONDITIONS=A bash run.sh agent 1`. Check that row 1 is `generated` with a cost above 0.
3. All conditions: `bash run.sh agent` (108 runs, about $15-25). It stops before a new condition once $40 is spent.
4. Checks: `bash run.sh checks` (several days, no API cost).

**Send back:** `agent_experiment/out_A/`, `out_F/`, `out_FC/`, `out_FCV/` (without `extracted`), `agent_experiment/agent_*.log`, `checks/out_checks_A/` to `checks/out_checks_FCV/`, `agent_experiment/manifest.csv` and `inputs/focal_rows.txt`.

## Cost

About $30-50 for the first round (student 1's 53 and all of student 2's runs). Student 1's expansion is another $35-55, only once we decide to run it.
