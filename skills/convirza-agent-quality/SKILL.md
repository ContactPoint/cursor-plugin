---
name: convirza-agent-quality
description: Review Convirza agent performance and call quality using scorecards and scored calls. Use when someone asks how their reps or agents are performing, wants a coaching list, asks about quality scores, conversion rates, scorecards or evaluation forms, or wants to know which calls were reviewed. Do not use for raw call volume reporting.
---

# Convirza agent quality and coaching

Answer performance questions from Convirza's scored data, not from raw call counts. A high call count is not performance; a quality score against a scorecard is.

## Workflow

1. **Establish the window and the population.** Convert the period to ISO 8601 UTC `start_date` / `end_date` and say which window you used. If a team or individual is named, resolve them first — `search` or `list_users` for a person (gives `user_id`), and pass `team` as the team-name filter. Never filter on a name you have not resolved.
2. **Call `get_agent_performance`** for the comparative view. It returns per-agent call handling and scoring stats. **Admin role required** — a 403 here means the connected user cannot see this data; say so plainly rather than substituting `get_calls` counts.
3. **Sort deliberately.** Default `sort_by` is `conversion_rate`; `agent_name`, `total_calls` and `avg_quality_score` are also available. Say which measure you ranked on, because the ranking changes completely between them.
4. **Ground the coaching in specific calls.** Use `get_scorecard_summary` for the period (`show_by` groups it by "Group Name" or "User Name") to see average scores and outcomes per group or agent, and per scorecard criterion. It is a summary with no call IDs and no scorecard filter — use `list_scorecards` to name the scorecards involved, `get_calls` to find the agent's calls, then `get_call` with `include=transcription,recordings` on the ones worth discussing.
5. **Say what the scores have in common.** The useful output is not "Dana scored 62" but "the four lowest-scoring calls all skipped the needs-assessment question". Read the transcripts before making that claim.
6. **Label a call only on request.** `add_call_title` overwrites any existing title on a scored call (the old title is not kept). Confirm the exact text and the call ID with the user first.

## Tools

| Goal                    | Tool                                      | Notes                                                                       |
| ----------------------- | ----------------------------------------- | --------------------------------------------------------------------------- |
| Compare agents          | `get_agent_performance`                   | **Admin only.** Filters: `team`, `user_id`, `org_unit_id`; `limit` max 1000 |
| Scorecard results       | `get_scorecard_summary`                   | Filter by date range; `show_by` = Group Name or User Name                   |
| Available scorecards    | `list_scorecards`                         | Resolve a scorecard name to an ID here                                      |
| The conversation itself | `get_call`                                | `include=transcription,recordings`; recording URLs expire after 7 days      |
| Resolve a person        | `search`, `list_users`, `get_user`        | Returns the `user_id` the filters need                                      |
| Label a scored call     | `add_call_title`                          | Write — confirm the ID and text first                                       |
| New / copied scorecard  | `create_scorecard`, `duplicate_scorecard` | Write — Admin or Standard role                                              |
| Retire a scorecard      | `archive_scorecard`                       | Destructive: removes it from active use, keeps scored history               |

## Output requirements

- Reproduce the tool's markdown table in full, then add the read-out beneath it.
- Always name the ranking measure and the date window in the first line.
- Report scores as they came back — never rescale, round to a grade, or invent a benchmark.
- Human-readable dates (`July 24, 2026 at 2:32 PM`) and durations (`3m 42s`).

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| An agent is missing from the results | They had no scored calls in the window. Dates default to the last 7 days; widen the window and say so |
| Scores differ from the Convirza app | The window (UTC here) or the scorecard differs. State both in the answer |

## When to decline or ask first

- **Decline** to recommend firing, disciplining or formally rating a named employee. Report the measured numbers and the specific behaviours in the transcripts, and leave the employment decision to the user.
- **Decline** to infer a protected characteristic, emotional state or intent about an agent or a caller from call data.
- **Ask first** before archiving a scorecard — scored history survives, but the scorecard leaves active use for everyone.
- **Ask which scorecard** when several exist; scores are not comparable across scorecards.
- **Stop and report** a 403 on `get_agent_performance` rather than assembling a substitute ranking from `get_calls` — that would answer a quality question with a volume metric.
