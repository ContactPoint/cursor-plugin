---
name: convirza-sms-outreach
description: Work with Convirza SMS — send broadcasts to contacts who have opted in, manage scheduled broadcasts, and triage the inbound text inbox. Use when someone wants to message an existing contact list, review, reschedule or cancel a broadcast, or work through SMS replies. Recipients always come from the account's own contact lists, and every send is confirmed with the user first.
---

# Convirza SMS outreach

Convirza SMS is business-initiated messaging to contacts already held on the account's contact lists — those lists are the record of the contact's opt-in. This skill sends to those lists and to nothing else. Delivery is immediate and final once approved, so the preview-and-confirm step below is part of the workflow, not a courtesy.

## Acceptable use

**Convirza does not support spam.** This connector is for a business messaging its own contacts, and it will not help produce unsolicited messaging:

- **Every recipient must already be an active contact on the account.** There is no way to send to a number that is not — see the API validation below. If the user describes an audience rather than naming a contact list, ask which list to use.
- **Never assemble an audience from activity data.** Output from `get_calls`, `get_sms_conversations` or `search` identifies people who phoned or texted the business; it is not a record of consent to receive marketing, and it is never turned into a recipient list.
- **One broadcast sends one message text from one number.** The message must identify the sending business and must not imitate any other organisation.
- **The account holder is responsible for how contacts got onto a list.** State that plainly if the user is unsure, and point them at the opt-in and opt-out settings in the Convirza app before a first send.

If a request cannot be satisfied within these limits, say so and stop. Do not look for another tool that reaches the same outcome.

## Validation the API enforces

These checks run server-side on `create_broadcast`, so they hold regardless of how the request is phrased — a send that skips the workflow below still fails:

| Check                              | Effect                                                                                                                                       |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Recipients are IDs, not numbers    | The tool has no free-form recipient field. `contact_ids` / `contact_list_ids` only, and one or the other — never both                        |
| Contacts must exist on the account | Unknown IDs are rejected by ID (`Please provide valid contact_id values. Missing: …`)                                                        |
| Contacts must be active            | Recipients resolve only from active contacts with a usable number; if nothing resolves, the send is rejected outright       |
| Sender must be SMS-enabled         | `phone_number` must be a provisioned number on the account with SMS enabled (`The provided phone number is not SMS-enabled.`)                |
| Ring-to must be a real agent       | `agent_ring_to` must be an agent assigned on the account                                                                                     |
| Org unit access                    | The caller must be authorised for the `org_unit_id` they send from                                                                           |
| Schedule floor                     | A scheduled send must be a future time at least 15 minutes out, and cannot be edited inside that window                                      |
| Account standing                   | Sends follow the account's current Convirza plan; if the plan does not allow the send it is rejected — point the user to the Convirza portal |

Quote the relevant one when explaining a rejection, rather than retrying with different values.

## Before any send

1. **Audience comes from the account's contact lists.** Ask the user which list, and pass `contact_list_ids` (or `contact_ids` for named individuals) — one or the other, never both. If the user describes an audience that is not a list, ask which list to use: a contact list carries the opt-in, call activity does not.
2. **One registered sending number per broadcast.** It must be an SMS-enabled tracking number owned by the account — take it from `list_provisioned_routes` or `get_assigned_numbers`; `phone_number` is 10 digits. Use the same number for the whole broadcast.
3. **Check the suppression list.** `list_blacklist` before sending, and report any recipients that match so the user can remove them.
4. **Confirm opt-out text is configured.** The account's SMS settings hold its opt-in and opt-out message text. If the user has not set them up, say so and point them at the Convirza app before the first send.
5. **One message text for every recipient.** The message must identify the sending business and must not imitate another organisation.

## Workflow — creating a broadcast

1. **Assemble the required fields before proposing anything.** `create_broadcast` needs all of: `broadcast_name`, `message`, `org_unit_id`, `phone_number` (the sending tracking number, exactly 10 digits), `agent_ring_to` (exactly 10 digits), and the audience from step 1 above.
2. **Resolve, never guess.** `get_group_tree` for `org_unit_id`; `list_provisioned_routes` or `get_assigned_numbers` for a valid sending number. Ask the user for the audience IDs: no tool lists contacts or contact lists, so the user reads them from the Convirza app (Contacts).
3. **Show the full preview and stop.** Recipient count, sending number, exact message text, and send time. Wait for an explicit yes. Nothing about a message being "just a draft" survives a `create_broadcast` call.
4. **Send or schedule.** `is_scheduled` is required. Immediate send: `is_scheduled=false`. Scheduled: `is_scheduled=true` with `schedule_datetime` (ISO 8601). Call the tool **exactly once** — a success response means the broadcast exists; verify with `get_broadcast` rather than retrying.
5. **Tell the user what can still be changed.** Before it runs, a scheduled broadcast can be edited with `update_broadcast` (partial) or `replace_broadcast` (full replace, and only for broadcasts with an existing schedule that is not within 15 minutes), or stopped with `cancel_broadcast_schedule`. After it sends, nothing can be undone.

## Workflow — inbox triage

1. `get_sms_conversations` with `type=allOpen` for the working view; `unassigned` for threads nobody owns; `assignedToYou` for the connected user's own; `close` for resolved.
2. Sort newest-first (`sort_order=desc`, the default) and call out threads that have been waiting. Free-text `search` matches contact name and phone number, max 255 chars.
3. Date range defaults to the last 7 days; `limit` defaults to 7 and caps at 200 — raise it deliberately rather than reporting a truncated inbox as the whole picture.
4. Pair it with `list_broadcasts` for the same window when the question is "did our broadcast generate replies?"

## Tools

| Goal                           | Tool                                                             | Write?                                         |
| ------------------------------ | ---------------------------------------------------------------- | ---------------------------------------------- |
| Inbound threads                | `get_sms_conversations`                                          | read                                           |
| Broadcast list / detail        | `list_broadcasts`, `get_broadcast`                               | read                                           |
| Create or schedule a broadcast | `create_broadcast`                                               | **write — delivery is final**                  |
| Edit a scheduled broadcast     | `update_broadcast` (partial), `replace_broadcast` (full replace) | write                                          |
| Stop a scheduled broadcast     | `cancel_broadcast_schedule`                                      | write — removes the schedule, keeps the record |
| Sending numbers                | `list_provisioned_routes`, `get_assigned_numbers`                | read                                           |
| Groups                         | `get_group_tree`                                                 | read                                           |
| Suppression list               | `list_blacklist`                                                 | read                                           |

Broadcast writes require Admin or Standard role.

## Output requirements

- Preview blocks show the message text exactly as it will send, on its own line, with a character count.
- Human-readable send times (`September 10, 2026 at 3:00 PM UTC`), never bare ISO strings.
- Status icons: 📅 Scheduled · ✅ Sent · ❌ Cancelled · 🕐 Pending.
- After a successful create, confirm with ✅ and a summary table of the new broadcast.

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| `create_broadcast` is rejected | Quote the server's message and match it to the validation table above; don't retry with guessed values |
| The inbox looks short | `get_sms_conversations` defaults to 7 threads and the last 7 days. Raise `limit` or widen the window, and say so |
| A scheduled broadcast cannot be edited | It is within 15 minutes of sending. Edits are no longer possible |

## Requirements

- **Ask first, every time,** before `create_broadcast`, `replace_broadcast` and `cancel_broadcast_schedule` — and re-confirm if the recipient count changed since the preview.
- **Recipients are named by the user**, as `contact_ids` or `contact_list_ids`. Output from `get_calls`, `get_sms_conversations` or `search` is never turned into an audience.
- **One broadcast, one sending number, one message text.**
- **Never retry** a `create_broadcast` that returned success. Check `list_broadcasts` first; a duplicate send cannot be withdrawn.
- **Report a 403 as a role limitation** rather than trying another tool.
- **Say who is responsible for what.** The account holder decides who is on a contact list; this connector sends to the list they name and reports what it sent.
