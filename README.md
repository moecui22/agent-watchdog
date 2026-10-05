# agent-watchdog

**Did each scheduled AI agent actually do its work?** Judged by the file it left behind, not by its exit code.

Unattended agents fail quietly. In my own setup, one agent reported `succeeded` five days running in 3–12 seconds while producing nothing. Another wrote a tidy 703-byte file whose whole content was *"blocked: access can't be approved"*. That file exists, so a check for "did it leave an output?" calls it a success.

`agent-watchdog` checks in two stages, cheap first:

| Stage | Question | How |
|---|---|---|
| 1. Existence | Did the agent leave its output for that day? | Plain code, no model. Always runs. |
| 2. LLM-as-judge | Does that output hold the work, or only a report that the run failed? | One yes/no judgment per file from a typed-output model ([TypeSafe](https://typesafe.ai) System One, a "Noul"). |

On known files, the judge scored a failure report **0.04** and real work **0.77–0.94**. Anything between 0.35 and 0.65 is reported as *unclear* and goes to a human rather than being forced either way. If the judge is unavailable, the check falls back to stage 1 and **says so**: a weaker check never passes silently.

It reports **state, not runs**. A notification goes out only when the set of problems changes. It names the agents rather than giving a count, says how to fix each one, and goes quiet until something changes. One green message when everything recovers.

```
$ agent-watchdog --days 1
Agent watchdog - did each agent do its work? (last 1 days)
  ok Research briefing                Sun 04 Oct  508 bytes  (0.91)
  x Video digest                     Sun 04 Oct  183 bytes - failure report only (0.05)

1 problem(s)
```
(Real output on the two files in `examples/artifacts/`.)

## Use

```bash
mkdir -p ~/.config/agent-watchdog
cp examples/agents.json ~/.config/agent-watchdog/agents.json   # then edit
export TYPESAFE_API_KEY=...        # optional; without it you get stage 1 only
./agent-watchdog                   # one line per agent per day
./agent-watchdog --quiet           # silent unless something is wrong (for cron / launchd)
./agent-watchdog --selftest        # checks the "when do we interrupt a human" logic
```

Each agent in `agents.json` has a `name`, the `due_hour` by which its output should exist, and an `output` path template (`{Y} {m} {d}`, globs allowed). Optional fields:
- `fix`: the action a human takes when this agent fails.
- `by_mtime`: for files that are overwritten each run, so only today can be judged.
- `judge: false`: for non-prose outputs.
- `paused`: an intentional pause, reported as paused rather than failed.

To get notifications, add a top-level `"notify": ["your-command"]`. It is called with `light, title, body`.

## Design notes

- **Judge the artifact, not the process.** Exit status and duration both proved unreliable on real runs. A file either holds the day's work or it doesn't.
- **The detector must not depend on what it watches.** Run it from cron or launchd, not from the agent scheduler it is checking. An earlier version that ran as a scheduled task inside that scheduler never ran at all.
- **Same fault twice is one fault.** Problems are normalised without their dates, so a broken agent produces one alert, not one per day.

Python 3.8+, standard library only. One file.

MIT licence.
