# Call record reference

## Dispositions

The `disposition` filter on `get_calls` takes one or more of these, comma-separated:

| Value       | Meaning                                   |
| ----------- | ----------------------------------------- |
| `ANSWERED`  | A person picked up and the call connected |
| `NO ANSWER` | Rang out with no answer                   |
| `BUSY`      | Destination was busy                      |
| `FAILED`    | Call could not be completed               |
| `HANGUP`    | Caller hung up before the call connected  |
| `NONE`      | No outcome was recorded                   |

No other values are accepted — the API rejects anything else with a 400.

"Missed calls" in ordinary speech means `NO ANSWER`; add `BUSY,FAILED` to widen. Say which values you used. Results may also show `CALLER HANGUP`, which cannot be filtered on. `get_opportunity_recovery` is AI scoring of lost opportunities, not call outcome, and has no date filter — use it only when the user asks for that.

## `include` datasets

`get_calls` and `get_call` accept a comma-separated `include`:

- `custom_sources` — custom attribution values
- `tags` — labels attached to the call
- `comments` — agent notes
- `indicators` — scored insight indicators
- `summary` — AI summary of the conversation
- `transcription` — speech-to-text of the conversation
- `transcription_details` — word-level transcript; large, request it for one call, not a list
- `extended_caller` — caller contact details (name, address)
- `dni_data` — web-visit data from dynamic number insertion
- `scorecard` — scorecard results

Any other value is rejected with a 400. Recording URLs are always included and **expire 7 days after the call**.

Only request what the answer needs — transcripts and caller details are personal data, and `include=transcription` on a large list returns a great deal of data for no benefit.

## Date handling

- Both `start_date` and `end_date` are ISO 8601 UTC, e.g. `2026-09-01T00:00:00Z`.
- Omitting both defaults to the last 7 days through now.
- `get_calls` has no maximum range; long ranges just mean more pages — use `total` for counts.
- `get_sms_conversations` uses the same params but maps them to `start` / `end` internally; pass `start_date` / `end_date` as documented.
- Users think in their local timezone. State the UTC window you queried so a boundary-day discrepancy is visible rather than mysterious.

## Attribution vocabulary

- **Campaign** — a logical source group used for reporting ("Google Ads Q1"). Numbers are assigned to campaigns; calls report the campaign that drove them.
- **Ad source** — the channel a number tracks (Google CPC, Organic Search, Direct Mail). Enumerate valid values with `list_ad_sources`.
- **Custom source** — an account-defined attribution value, surfaced via `include=custom_sources`.
- **Organization unit (`org_unit_id`)** — a node in the account hierarchy. Every call belongs to one. Discover IDs with `get_group_tree`.
