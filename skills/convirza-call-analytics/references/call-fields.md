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

"Missed calls" in ordinary speech means `NO ANSWER,BUSY,HANGUP`. Confirm that reading with the user before reporting on it, and prefer `get_opportunity_recovery` when they want a follow-up list rather than a count.

## `include` datasets

`get_calls` and `get_call` accept a comma-separated `include`:

- `tags` — labels attached to the call
- `comments` — agent notes
- `custom_sources` — custom attribution values
- `call_flow` — the routing configuration that handled the call
- `recordings` — audio URLs; **these expire 7 days after the call**
- `transcription` — speech-to-text of the conversation

Only request what the answer needs. `include=recordings,transcription` on a large list returns a great deal of data for no benefit.

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
