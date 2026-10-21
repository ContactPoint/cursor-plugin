---
name: convirza-account-administration
description: Administer a Convirza account — users, groups and the org hierarchy, call tags, the caller blacklist, and OAuth API credentials. Use when someone wants to onboard or offboard a user, change a role or group, see the org structure, block or unblock a caller, tidy up tags, or review and rotate API credentials.
---

# Convirza account administration

Account changes affect who can get in and who can be reached. Look everything up, show the change, and wait for a yes.

## Workflow — the hierarchy comes first

Every resource in Convirza belongs to one organization unit (`org_unit_id`). Before any scoped read or any create, call `get_group_tree` and work from real IDs. `get_group_org_list` returns a subtree (parent plus two levels) when the full tree is unwieldy; `get_group` gives one unit in full.

## Workflow — onboarding a user

1. `search` on the name and the email address. If they already exist, stop and show the existing record — duplicate users are worse than no user.
2. `get_group_tree` for the target `org_unit_id`; `list_users` to see how comparable colleagues are configured, so the role you propose matches the account's conventions.
3. Show the exact record — name, email, role, group — and wait for approval.
4. `create_user` once with `first_name`, `last_name`, `username` (the email address), `role` (`Admin`, `Standard`, `Read-Only` or `Identified-only`) and `ct_user_org_unit_id` (**Admin role required**). The user can see that group and every group under it; to limit them to some sub-groups only (e.g. two locations), add `group_access_ids` with those group IDs from `get_group_tree` — they must sit under `ct_user_org_unit_id`. The new user is emailed a welcome link to set their own password (valid 72 hours). Confirm with `get_user` and a summary table, and tell the user the invite email is on its way; never invent a password or transmit one in chat. If the person did not get the email, the link expired or they forgot their password, call `send_password_reset_email` with their user ID.

## Workflow — offboarding a user

1. `search`, then `get_user` to confirm you have the right person. Two people with the same name is common; stop and ask rather than picking.
2. State plainly that `delete_user` is a **soft delete** — the record and history are preserved and access is revoked.
3. After approval, `delete_user` once (**Admin role required**).
4. Offer the cleanup that actually matters: `list_provisioned_routes` for routes ringing to their number, and `get_agent_performance` for their historic stats.

## Workflow — groups

`create_group` and `replace_group` / `update_group` require **Admin**. `delete_group` is a soft delete and is **not guarded**: the API deletes the group even when it still has child groups, users or tracking numbers. Before asking for approval, check `list_groups` and `list_users` with that `org_unit_id`, and tell the user exactly what is still inside. `update_group` patches; `replace_group` replaces everything.

## Workflow — blacklist

1. `list_blacklist` first, so nothing already blocked gets re-added.
2. Justify each addition from data — `get_calls` grouped by `caller_id`, where repeated very-short or `HANGUP` calls are the usual nuisance signature.
3. `add_to_blacklist` per confirmed number. Blacklisted numbers stop generating new call records, which also means they disappear from future reports.
4. `delete_from_blacklist` unblocks one number: `org_unit_id` plus its `caller_id`. `bulk_delete_blacklist` removes the `caller_ids` you list for an org unit — there is no "remove all", so take the numbers from `list_blacklist`, show them, and confirm first.

## Workflow — API credentials

1. `list_oauth_credentials` / `get_oauth_credential` to review. **Never print a secret value** that comes back in a response, and never repeat one the user pastes.
2. `create_oauth_credentials` (**Admin**) returns the `client_id` and `client_secret_link`, a one-time link. Give the user the link: opened in a browser within 10 minutes, it reveals the secret once. The secret never appears in chat.
3. `rotate_oauth_secret` (**Admin**) invalidates the old secret **immediately**; every integration using it breaks until updated, and the new secret comes back as `client_secret_link`, a one-time link (10 minutes, one use). Spell that out and get explicit approval before calling it.
4. `delete_oauth_credential` (**Admin**) permanently kills all API access using it.

## Tools

| Area                | Read                                                               | Write                                                                                          |
| ------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Users               | `get_my_profile`, `list_users`, `get_user`                         | `create_user`, `update_user` (PATCH), `replace_user` (PUT), `delete_user` (soft), `send_password_reset_email` — Admin |
| Groups              | `get_group_tree`, `list_groups`, `get_group_org_list`, `get_group` | `create_group`, `update_group` (PATCH), `replace_group` (PUT), `delete_group` (soft) — Admin   |
| Tags                | `list_tags`                                                        | `create_tag` (accepts several; reports duplicates), `update_tag` (rename), `delete_tag` (soft) |
| Blacklist           | `list_blacklist`                                                   | `add_to_blacklist`, `delete_from_blacklist`, `bulk_delete_blacklist`                           |
| API credentials     | `list_oauth_credentials`, `get_oauth_credential`                   | `create_oauth_credentials`, `rotate_oauth_secret`, `delete_oauth_credential` — Admin           |
| Reference data      | `list_industries`, `list_ad_sources`                               | —                                                                                              |
| Cross-entity lookup | `search`, `fetch`                                                  | —                                                                                              |

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| A known user is missing from `list_users` | It returns active users only by default. Pass `user_status` "inactive" or "deleted" for offboarded people |
| A full name matches nothing | `name` matches first OR last name on its own — search "John", not "John Smith", or use `email` or `search` |
| A user never got the welcome email | Call `send_password_reset_email` with their user ID; the link is valid 72 hours |

## When to decline or ask first

- **Decline** to create a user, change a role or grant Admin without the account owner asking for that specific person and that specific role. Privilege escalation dressed up as a convenience is still privilege escalation.
- **Decline** to print, repeat, log or store a `client_secret`, password or token. Tools never return one; new secrets come back only as a one-time `*_secret_link`.
- **Decline** to blacklist a number on a hunch, or to bulk-remove blacklist entries without the user seeing the list first.
- **Ask first** before every write here, and quote the resolved ID you are about to act on.
- **Stop and report** a 403 as a role limitation — do not attempt a different tool to work around a permission the account does not have.
