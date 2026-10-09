---
name: convirza-call-analytics
description: Report on Convirza call activity — volume, outcomes, missed-call recovery and marketing attribution. Use when someone asks about calls, call history, missed or abandoned calls, call recordings or transcripts, which ad source or campaign drove calls, or wants a call report for a date range. Do not use for agent scoring or call quality reviews.
---

# Convirza call analytics

Turn raw Convirza call records into a report someone can act on. Every number in the answer must come from a tool call — never estimate, extrapolate or carry a figure over from an earlier turn.

## Workflow

1. **Fix the window and the scope.** Convert whatever the user said ("this month", "last week") into explicit ISO 8601 UTC `start_date` / `end_date` values and state the window you used in your answer. If they named a team, location or division, call `get_group_tree` first and use the resulting `org_unit_id` — never filter by group name.
2. **Pick the right entry tool** for the question:
    - call volume, outcomes, a call log → `get_calls`
    - one specific call → `get_call`
    - missed, hung-up or unconverted calls → `get_opportunity_recovery`
    - which source or campaign drove calls → `get_attribution_engine`
    - text conversations rather than voice → `get_sms_conversations`
3. **Mind the range.** `get_calls` has no maximum range; a long range just means more pages, so use `total` for counts rather than paging through everything. Both date params default to the last 7 days when omitted.
4. **Pull detail only where it earns its place.** Add `include=recordings,transcription,tags` on `get_call` when the user asks what was said or wants to listen. Recording URLs expire after 7 days — say so when you hand one over.
5. **Report, don't dump.** Lead with a one-line headline (totals and the answered rate), reproduce the tool's markdown table in full, then add the breakdown that answers the actual question — by disposition, by campaign, or by repeat caller.
6. **Close with the next step.** Name the one or two follow-ups that matter, e.g. drilling into a specific call, or blocking a repeat nuisance caller.

## Tools

| Goal | Tool | Notes |
| --- | --- | --- |
| Call list / volume | `get_calls` | No max range; filter by `org_unit_id`, `campaign_id`, `provisioned_route_id`, `caller_id`, `phone_number`, `ring_to`, `disposition`, `external_id` |
| One call in full | `get_call` | `include` = tags, comments, custom_sources, call_flow, recordings, transcription |
| Missed opportunities | `get_opportunity_recovery` | Filter with `insight_ids`, `org_unit_id`; no date filter |
| Source / campaign performance | `get_attribution_engine` | Filter with `org_unit_id`, sort with `sort_by`; no date filter |
| SMS threads | `get_sms_conversations` | `type` = allOpen, assignedToYou, unassigned, close |
| Resolve names to IDs | `search`, `list_campaigns`, `get_group_tree`, `list_ad_sources` | Always before filtering |
| Score a recording | `upload_call` | mp3 or wav via `recording_url` or an attached `recording_file`; needs `call_date`, `org_unit_id`, `caller_id`, `ring_to`. No `call_id` comes back — poll `get_calls` with the returned `external_id`, then `get_call` with `include` = "scorecard,indicators,transcription". For a local file the tool cannot receive, omit both: the user uploads it through the returned "upload_url" (or the upload box, where the host shows one) and sees the score there. When call details are known but no URL or file was given, first ask the user: paste a recording URL, or upload the file through the upload link? |

Everything except `upload_call` is read-only. `upload_call` adds a call — confirm the group and phone numbers with the user first. When it returns `conversation_analytics_enabled: false`, tell the user the call will not be transcribed or scored; do not promise a score.

## Output requirements

- Table rows come from the tool response verbatim — never paraphrase or re-order silently.
- Dates as `July 24, 2026 at 2:32 PM`, never ISO strings.
- Durations as `3m 42s`, never raw seconds.
- Status icons: ✅ Answered · 🔕 No Answer · 📵 Busy · ❌ Failed · 📴 Hangup.
- When `total` exceeds what was returned, say so: `*Showing 1–25 of 47.*`

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| No calls come back | Dates default to the last 7 days. Widen the window, say so, and check the `org_unit_id` is the group the user meant |
| A recording link does not play | Recording URLs expire 7 days after the call; there is nothing to recover |
| Counts differ from the Convirza app | The app shows local time; this query used UTC. State the UTC window so a boundary-day difference is visible |
| Attribution or recovery ignores the dates asked | `get_attribution_engine` and `get_opportunity_recovery` have no date filter. Say so rather than implying a window |

## When to decline or ask first

- **Ask** which group or campaign when the account has several and the request is ambiguous — a wrong `org_unit_id` produces a confidently wrong report.
- **Decline** to guess a caller's identity, intent or creditworthiness from call metadata.
- **Decline** to produce a per-person productivity ranking from raw call counts; point at the `convirza-agent-quality` skill, which uses the scored data built for that purpose.
- **Stop and report** a 401 (session expired — the user must reconnect) or a 403 (the account lacks the required role; `get_agent_performance` needs Admin).

See `references/call-fields.md` for disposition values, `include` datasets and date rules.
