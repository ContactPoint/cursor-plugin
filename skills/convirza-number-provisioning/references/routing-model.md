# Convirza routing model

## The objects, and how they nest

```
Organization unit (org_unit_id)      the account hierarchy node everything belongs to
└── Provisioned route                a tracking line: one routing configuration
    ├── phone number(s)              the dialable number(s) attached to it
    ├── routing_configuration        routing_type + config; config.ringto is where a call
    │                                forwards (read responses show it as config.destination)
    ├── ad_source_id                 the channel this line represents (list_ad_sources)
    └── media prompts                greeting / whisper / hold audio it plays
Campaign (campaign_id)               reporting group a route reports against
Geo location                         a geographic rule
└── Location route                   a destination within that rule
DNI setting                          swaps a tracking number into a live web page
```

Provisioning is two steps, and both are required: `add_tracking_number` acquires the number, `create_provisioned_route` gives it behaviour. A number with no route does nothing useful; a route with no number cannot be dialled.

## PATCH vs PUT

Getting this wrong silently erases configuration. The partial-update tools are safe; the full-replacement tools are not.

| Partial (PATCH — safe)    | Full replace (PUT — overwrites every field)                    |
| ------------------------- | -------------------------------------------------------------- |
| `patch_provisioned_route` | `replace_provisioned_route`                                    |
| `update_geo_location`     | `replace_geo_location`                                         |
| `update_location_route`   | `replace_location_route`                                       |
| `update_campaign`         | `replace_campaign`                                             |
| `update_webhook`          | `replace_webhook`                                              |
| `update_user`             | `replace_user`                                                 |
| `update_group`            | `replace_group`                                                |
| —                         | `update_dni_setting` (named "update", behaves as PUT)          |
| —                         | `update_call_action_template` (named "update", behaves as PUT) |

The last two rows are the trap: their names say update and their behaviour is replace. Before calling either, read the current record and supply every field.

## Route status and deletion

- A route is `active` or `inactive`. An inactive route still holds its number.
- `delete_provisioned_route` deletes a route of any routing type and cleans up its resources; the number stays in the account as `reserved`.
- `delete_phone_number` removes the number from inventory. Treat it as unrecoverable.

## Search constraints

`search_phone_numbers` accepts `match` (a digit pattern; `*` or `%` is a wildcard, so `801*` means area code 801 — a bare `801` matches those digits anywhere in the number) **or** the `state` + `rc` pair, never both. `state` and `rc` must be supplied together.
