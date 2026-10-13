---
name: convirza-ai-voice-agents
description: Review and retire Convirza AI voice agents — the voice bots that answer or place calls. Use when someone asks what voice agents exist, wants to understand how one is configured, wants to know which tracking numbers route to an agent, or wants to remove an agent that is no longer used. Creating or editing an agent is not available through this connector.
---

# Convirza AI voice agents

An AI voice agent is an autonomous bot that speaks to real callers on the phone. Through this connector you can **read** agent configuration and **delete** an agent — nothing else.

## What this connector cannot do

The create, patch and update agent tools are **not available** through this connector. Agent authoring is done in the Convirza app.

When a user asks to create an agent, change a greeting, edit a system prompt, adjust a voice, or upload a knowledge-base file, say plainly that agent authoring happens in the Convirza app and this connector is read-and-retire only. **Do not** try to reach the same outcome another way — do not attempt it through `create_provisioned_route`, a webhook, or a call action template. Offer to show the current configuration instead so the user knows what to change once they are in the app.

## Workflow — reviewing what exists

1. `list_ai_agents` first, with optional status filter, text search and sorting. Show the full markdown table.
2. `get_ai_agent` for the one in question — voice, prompts, greetings, notifications and files. Explain in plain words what a caller will actually hear and what the agent is instructed to do, rather than reprinting raw fields.
3. Cross-check with `list_provisioned_routes` to see which tracking numbers route to an agent. An agent nothing points at is a strong retirement candidate; an agent several numbers point at must not be deleted casually.

## Workflow — retiring an agent

1. Confirm the exact agent with `get_ai_agent` — never work from a name alone when several are similar.
2. Check `list_provisioned_routes` for numbers routing to it. If any do, say so and stop: deleting it changes what live callers experience immediately.
3. State that `delete_ai_agent` is **permanent** — the agent and its whole configuration go, with no restore path, and it cannot be recreated through this connector.
4. Wait for explicit approval, then call `delete_ai_agent` exactly once. Confirm with `list_ai_agents`.

## Tools

| Goal                        | Tool                                               | Write?                                              |
| --------------------------- | -------------------------------------------------- | --------------------------------------------------- |
| List agents                 | `list_ai_agents`                                   | read — filters: status, search, sort                |
| Full configuration          | `get_ai_agent`                                     | read                                                |
| Retire an agent             | `delete_ai_agent`                                  | **destructive, permanent** — Admin or Standard role |
| See what routes to an agent | `list_provisioned_routes`, `get_provisioned_route` | read                                                |

## Common issues

| Issue | What to do |
| --- | --- |
| 401 Unauthorized | The session expired. Ask the user to reconnect the Convirza connector, then retry once |
| 403 on `delete_ai_agent` | The connected user's role is Read Only. Deleting needs Admin or Standard |
| The user wants to create or edit an agent | Not available here. Point them at the Convirza app and offer to show the current configuration |

## When to decline or ask first

- **Decline** to create or edit an agent, and do not offer a workaround — the capability is not exposed here. Point the user at the Convirza app.
- **Ask first** before `delete_ai_agent`, every time, and say that it is permanent and unrecoverable through this connector.
- **Stop** if tracking numbers still route to the agent, and report which ones before the user decides.
- **Decline** to speculate about what an agent will say beyond what `get_ai_agent` returns; read the configured greeting and prompt rather than inventing behaviour.
