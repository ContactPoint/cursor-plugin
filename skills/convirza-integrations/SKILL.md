---
name: convirza-integrations
description: Wire Convirza into other systems — webhooks, event subscriptions and automated call action templates. Use when someone wants Convirza to notify an external endpoint about calls or messages, review or change existing webhook and subscription configuration, or set up automated triggers and actions on calls.
---

# Convirza integrations

These tools send Convirza data to systems outside the account. Getting an endpoint or an event filter wrong leaks data to the wrong place, so read the current state first and confirm every write.

## Two ways to send data out — pick the right one

| | Call webhook | Event subscription |
| --- | --- | --- |
| Set up with | `create_webhook`, then `create_call_action_template` with `triggerName` "webhook" and `triggerValues` ["<webhook_id>"] | `create_subscription` with one `entity_type` and `event_types` (add, update, delete) |
| Fires when | A call matches the template's rules | A record of that entity type (Calls, Users, Campaigns, Numbers…) is added, updated or deleted |
| Payload | Call data, plus optional DNI logs, indicator scores, IVR keypresses, data append and `static_parameters` | The event: `event_type`, `entity_id`, the entity data, `delivery_timestamp`, `idempotency_key` |
| Signed | **No.** Authenticate with HTTPS plus a shared token in `static_parameters` | **Yes** — `X-Webhook-Signature` is a hex HMAC-SHA256 of the raw body, keyed with the subscription's signing secret |
| On create | Nothing is sent | Convirza sends a test request first; creation fails unless the endpoint answers 2xx within 5 seconds |
| Failures | Retried, then an error email | 3 attempts in total; after the third failure the subscription is **disabled** and the billing Admin is emailed |

A webhook on its own does nothing: it only fires when a call action template points at it.

## Workflow

1. **Inventory before adding.** `list_webhooks` and `list_subscriptions` together. Duplicate delivery to the same endpoint is a common and confusing failure — show the user what is already configured before proposing anything new.
2. **Get the endpoint from the user, verbatim.** Never infer, complete or correct a URL. Require HTTPS and say why if the user offers HTTP. Confirm the host is one they own.
3. **Show the payload and stop.** Endpoint, events, name. State plainly that this will send Convirza call and message data to a third-party system. Wait for an explicit yes.
4. **Create once.** For call data: `create_webhook` (Admin), then a call action template that triggers it, then verify with `get_webhook`. For entity events: make sure the endpoint is live first, then `create_subscription` (Admin); it returns `webhook_secret_link`, a one-time link (10 minutes) where the user reveals the signing secret. Verify with `list_subscriptions`.
5. **Edit with the right verb.** `update_webhook` patches; `replace_webhook` overwrites every field. Read the record first before any replace.
6. **Call action templates** attach triggers and actions to call handling. `create_call_action_template` creates; **`update_call_action_template` is a full PUT** — its name says update and it replaces triggers and actions wholesale. Fetch and re-supply everything, or automation silently disappears.

## Tools

| Goal                            | Tool                                                                   | Write?                                            |
| ------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------- |
| Webhook inventory               | `list_webhooks`, `get_webhook`                                         | read                                              |
| Create a webhook                | `create_webhook`                                                       | write — third-party delivery                      |
| Change a webhook                | `update_webhook` (PATCH), `replace_webhook` (PUT — full overwrite)     | write                                             |
| Stop a webhook                  | `delete_webhook`                                                       | **destructive** (soft delete; future events stop) |
| Event subscriptions             | `list_subscriptions`                                                   | read                                              |
| Subscribe / unsubscribe         | `create_subscription`, `delete_subscription`                           | write — Admin                                     |
| Call automation                 | `create_call_action_template`, `update_call_action_template` (**PUT**) | write                                             |
| API credentials for the far end | see the `convirza-account-administration` skill                        | —                                                 |

## Verifying an integration

There is no "send test event" tool. A subscription is tested once, at creation. To confirm a call webhook works, read it back with `get_webhook`, then have the user watch their own endpoint logs while a real call or message arrives — `get_calls` or `get_sms_conversations` for the same window shows what Convirza recorded, which is what the endpoint should have received. Say this plainly rather than implying a test fired.

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| A call webhook receives nothing | No call action template triggers it, or no call has matched the rules yet. Check the template, then compare with `get_calls` for the same window |
| `create_subscription` fails with 400 | The endpoint did not answer 2xx within 5 seconds. Deploy it first, then retry once |
| A subscription stopped delivering | It was disabled after 3 failed attempts. There is no re-enable tool: fix the endpoint, then `delete_subscription` and create it again (this issues a new signing secret) |
| Triggers or actions vanished from a call action template | `update_call_action_template` is a full PUT. Fetch the template and re-supply every field |

## When to decline or ask first

- **Decline** to point a webhook at an endpoint the user did not give you explicitly, at a domain they do not control, or at a plain-HTTP URL without saying why that is unsafe.
- **Decline** to put a credential, API key or bearer token into a webhook URL as a query parameter — it will end up in logs at both ends.
- **Decline** to configure delivery of call recordings or transcripts to a third party without the user confirming they have the right to share that content; recordings can contain personal data spoken by callers who never consented to onward transfer.
- **Ask first** before every write, and before any `replace_*` or `update_call_action_template` show the user which existing fields you are re-supplying.
- **Stop and report** a 403 as a role limitation (subscriptions need Admin).
- **Decline billing requests.** Event subscriptions here are webhook-style notifications, not the customer's Convirza plan. Cancelling a plan, invoices and refunds are not handled by this connector — say so and point the user to Convirza support without calling any subscription tool.
