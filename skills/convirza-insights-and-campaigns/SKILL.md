---
name: convirza-insights-and-campaigns
description: Configure what Convirza measures — campaigns for attribution, keyword and LLM-based insights, custom insight rules, and scorecards. Use when someone wants to create or review a campaign, set up keyword tracking or call analysis rules, audit what is being measured on calls, or manage scorecards and tags used for reporting.
---

# Convirza insights and campaigns

This skill sets up the measurement, not the report. Everything created here changes what future calls are classified as — it does not reclassify calls that already happened. Say so, because users routinely expect a new insight to backfill.

## Workflow — reviewing coverage

1. `list_insights` (keyword and LLM-based call analysis), `list_custom_insights` (rule-based conditions) and `list_scorecards` (quality evaluation forms). Show each as its own table.
2. Summarise what is tracked and — more usefully — what obvious categories are not. A coverage review whose only output is "here is what you have" answered nothing.
3. `get_insight` / `get_custom_insight` to explain any specific rule in plain words.
4. Propose additions as drafts for approval, never as writes.

## Workflow — campaigns

1. `list_campaigns` to check for an existing campaign before creating a near-duplicate; `list_ad_sources` and `list_industries` for the valid channel values.
2. Show the record, get approval, then `create_campaign` once (Admin role only). Only `campaign_name` and `org_unit_id` are needed — owner, start (now), timezone, spend and conversion value default.
3. `update_campaign` patches; `replace_campaign` overwrites every field. `delete_campaign` is a **soft delete** that preserves historical attribution — say that, because users expect deletion to break their old reports and it does not.
4. Explain the link to numbers: a campaign only collects data once tracking numbers point at it via `create_provisioned_route` — a route's `campaign_id` cannot be changed afterwards, so pick the campaign before creating the route.

## Workflow — insights

- `create_insight` sets up keyword tracking or LLM-based analysis on calls.
- `create_custom_insight` builds rule-based conditions; `update_custom_insight` patches and `replace_custom_insight` overwrites everything.
- Draft the rule in words first, get agreement on what should and should not match, then write it. A vague keyword rule produces confidently wrong analytics for months.
- `delete_insight` / `delete_custom_insight` remove the rule. Confirm before either.

## Workflow — scorecards and tags

`list_scorecards`, `create_scorecard`, `duplicate_scorecard` (copies all criteria) and `archive_scorecard` (retires it, keeps scored history). For reviewing results, use the `convirza-agent-quality` skill. Tags are freeform labels: `list_tags`, `create_tag` (accepts several at once and reports duplicates), `update_tag` (rename), `delete_tag` (soft delete). Cross-check against `list_provisioned_routes` before retiring anything a live configuration may reference.

## Tools

| Area | Read | Write |
| --- | --- | --- |
| Campaigns | `list_campaigns`, `get_campaign` | `create_campaign`, `update_campaign` (PATCH), `replace_campaign` (PUT), `delete_campaign` (soft), `move_campaign` (to another group) |
| Insights | `list_insights`, `get_insight` | `create_insight`, `delete_insight` |
| Custom insights | `list_custom_insights`, `get_custom_insight` | `create_custom_insight`, `update_custom_insight` (PATCH), `replace_custom_insight` (PUT), `delete_custom_insight` |
| Scorecards | `list_scorecards` | `create_scorecard`, `duplicate_scorecard`, `archive_scorecard` |
| Tags | `list_tags` | `create_tag`, `update_tag`, `delete_tag` (soft) |
| Reference data | `list_ad_sources`, `list_industries` | — |

Writes require Admin or Standard role.

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| 403 on a write | The connected user's role is Read Only. Writes need Admin or Standard; say which, don't try another tool |
| A new insight matches no past calls | Insights apply to future calls only; there is no backfill |
| A campaign collects no calls | No route points at it yet. Create the route with that `campaign_id`; it cannot be changed afterwards |

## When to decline or ask first

- **Say up front** that new insights and custom insights apply to future calls; there is no backfill tool. Never imply otherwise to make an answer tidier.
- **Decline** to build an insight or scorecard criterion that classifies callers by a protected characteristic — race, religion, national origin, disability, age, or a proxy for one such as accent or neighbourhood.
- **Decline** to create a keyword rule intended to flag callers for exclusion from service.
- **Ask first** before every write, and before any `replace_*` show which existing fields you are re-supplying.
- **Ask** which org unit when the account has several — an insight scoped to the wrong unit measures nothing and looks like a broken feature.
