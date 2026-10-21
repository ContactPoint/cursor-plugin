---
name: convirza-number-provisioning
description: Provision and route Convirza tracking phone numbers. Use when someone wants to find, add, release or audit a phone number, set up or change call forwarding and call flows, configure geographic routing, or set up Dynamic Number Insertion for website call tracking. Involves adding real numbers from a carrier and changing where live calls ring.
---

# Convirza number provisioning and routing

Adding a number takes a real number from the carrier into the account and changes what happens to real inbound calls. Every write in this skill is confirmed with the user before it is issued, and every ID is looked up rather than guessed.

## Workflow — launching a new tracking number

1. **Resolve the destination org unit.** `get_group_tree` → the `org_unit_id` the number will belong to. Confirm it with the user by name, not just ID.
2. **Check the campaign exists.** `list_campaigns`. If the campaign the user named is missing, offer `create_campaign` (see the `convirza-insights-and-campaigns` skill) before provisioning, so the first call is attributed correctly.
3. **Search inventory.** `search_phone_numbers` with `npa` for an area code ("801") or a toll-free prefix ("888"), or `state` + `rc` (rate center, e.g. "UT" + "SALT LAKE"). It has no city filter: search the city's area code and say which code you searched; if nothing comes back, try the city's other area code, then offer toll-free — never switch codes silently. Use `match` only to find a specific digit pattern: for carrier numbers it can match the digits anywhere, even with a trailing `*`. `match` cannot be combined with the others. Present the candidates as a table and let the user pick.
4. **Confirm, then add.** Restate the exact number, the org unit and the intended destination, and wait for an explicit yes. Then call `add_tracking_number` **once** with `phone_numbers: [{ number, source }]`, copying both values from the search result. This adds a real number from the carrier; usage follows the account's current Convirza plan, managed in the Convirza portal — send plan or usage questions there. Never retry on an apparent timeout without first checking `get_assigned_numbers` (added numbers land there as reserved). Each added row returns `number_id`.
5. **Attach routing.** `create_provisioned_route` with `provisioned_route_name`, `org_unit_id`, `ad_source_id` (from `list_ad_sources`), `routing_configuration` (`{ routing_type: "simple", config: { ringto: "<10 digits>" } }` for plain forwarding) and `phone_numbers: [{ phone_number, phone_number_id }]` for the number just added (`phone_number_id` = the `number_id` returned by `add_tracking_number`; the route cannot be created without a number). Add `campaign_id` when the user named a campaign. Without a route a provisioned number has no behaviour.
6. **Verify and explain.** `get_provisioned_route`, then describe in plain words what a caller dialling that number will now experience.

## Workflow — auditing what exists

1. `list_provisioned_routes` — every routing configuration and its status.
2. `get_provisioned_numbers` and `get_assigned_numbers` — the number inventory.
3. Reconcile the three and flag: routes with no forwarding destination (`routing_configuration.config.destination` in read responses), inactive routes still holding a number, and provisioned numbers attached to no route.
4. `get_provisioned_route` for detail on anything suspicious. Recommend fixes; change nothing without approval.

## Workflow — geographic routing

A **geo location** holds the geographic rule; **location routes** inside it point at destinations. Read the existing setup with `list_geo_locations` and `list_location_routes` before adding to it, then `create_geo_location` followed by `create_location_route` — each confirmed separately.

## Workflow — website call tracking (DNI)

`list_dni_settings` first, always. `update_dni_setting` is a **full replacement (PUT)** — read the current values and merge them yourself, or you will silently blank fields. DNI changes what real visitors see on a live website, so confirm before writing.

## Tools

| Goal | Tool | Write? |
| --- | --- | --- |
| Find available numbers | `search_phone_numbers` | read (carrier inventory) |
| Number inventory | `get_provisioned_numbers`, `get_assigned_numbers` | read |
| Add a tracking number | `add_tracking_number` | **write — real carrier number** |
| Release a number | `delete_phone_number` | **destructive** |
| List / read routing | `list_provisioned_routes`, `get_provisioned_route` | read |
| Create routing | `create_provisioned_route` | write |
| Change routing | `patch_provisioned_route` (partial), `replace_provisioned_route` (full overwrite) | write |
| Delete routing | `delete_provisioned_route` | **destructive** — any routing type; the number stays in the account as reserved |
| Move a number to another campaign | `move_provisioned_route` | **destructive** — moves the number to another campaign (and group, if different); pool and DNI numbers refused |
| Geo rules | `list_geo_locations`, `get_geo_location`, `create_geo_location`, `update_geo_location` (PATCH), `replace_geo_location` (PUT), `delete_geo_location` | mixed |
| Geo destinations | `list_location_routes`, `get_location_route`, `create_location_route`, `update_location_route` (PATCH), `replace_location_route` (PUT), `delete_location_route` | mixed |
| Website tracking | `list_dni_settings`, `get_dni_snippet` (read-only: snippet to paste into the site), `create_dni_setting`, `update_dni_setting` (**PUT — full replace**), `delete_dni_setting` | mixed |
| Call flow audio | `list_media_prompts`, `get_media_prompt`, `delete_media_prompt` | mixed |

Writes require Admin or Standard role.

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| 403 on a write | The connected user's role is Read Only. Writes need Admin or Standard; say which, don't try another tool |
| `add_tracking_number` seemed to time out | Check `get_assigned_numbers` (added numbers land there as reserved) before any retry |
| `move_provisioned_route` is refused | Pool and DNI numbers cannot be moved |
| DNI fields went blank after an update | `update_dni_setting` is a full PUT. Re-read the setting and re-supply every field |

## When to decline or ask first

- **Always confirm before adding a number.** State the number, the org unit and that this takes a real number from the carrier into the account.
- **Always confirm before deleting a route or releasing a number.** Live calls stop arriving; a released number generally cannot be recovered.
- **Read before any PUT.** `replace_*` and `update_dni_setting` / `update_call_action_template` overwrite every field. Fetch current values first and show the user the diff.
- **Decline** to point a tracking number at a destination the user has not explicitly given you — never infer a forwarding number from a call record.
- **Decline** to add numbers in bulk, or in volume across many area codes, without a stated business purpose and an explicit per-batch confirmation.
- **Call each write exactly once.** A success response means it happened; verify with the matching `get_*` tool instead of retrying.

See `references/routing-model.md` for how numbers, routes, campaigns and org units fit together.
