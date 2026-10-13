# Convirza plugin for Cursor

Connects Cursor to a Convirza call tracking account through the hosted Convirza MCP server (`https://connect.convirza.com/mcp`). Ask about calls, recordings and transcripts, ad source attribution, missed-call recovery, agent scorecards, tracking numbers and routing, SMS broadcasts, AI voice agents and webhooks.

The plugin contains no server code. It points Cursor at the hosted server and bundles nine skills that guide the agent through common workflows.

## Install

Install **Convirza** from the Cursor Marketplace. Or add the server by hand to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "convirza": { "url": "https://connect.convirza.com/mcp" }
  }
}
```

## Sign in

The first time a Convirza tool runs, Cursor opens the Convirza sign-in page in your browser. Sign in with your Convirza account and approve access. No API key or token goes in the config.

Only users with the **Admin**, **Standard** or **Read Only** role can connect. Read Only users can use the read tools only.

## What's included

| Skill                             | Use it for                                                                |
| --------------------------------- | ------------------------------------------------------------------------- |
| `convirza-call-analytics`         | Call volume, outcomes, recordings, transcripts, attribution, missed calls |
| `convirza-agent-quality`          | Agent performance and scorecards                                          |
| `convirza-insights-and-campaigns` | Campaigns, keyword and custom insights, scorecard setup                   |
| `convirza-number-provisioning`    | Finding, adding and routing tracking numbers, geo routing, DNI            |
| `convirza-sms-outreach`           | SMS conversations and broadcasts                                          |
| `convirza-ai-voice-agents`        | Reviewing and retiring AI voice agents                                    |
| `convirza-integrations`           | Webhooks, event subscriptions, call action templates                      |
| `convirza-account-administration` | Users, groups, tags, blacklist, OAuth API credentials                     |
| `convirza-developer`              | DNI snippet in your site, webhook receivers with signature checks, API client |

## Example prompts

**Calls and attribution**

- "How many calls did we get last week, and how many were missed?"
- "Which ad source drove the most calls this month?"
- "Show me the transcript of the last call from 801-555-0142"

**Agent quality**

- "Rank my agents by average quality score for September"
- "Which calls pulled down Dana's scorecard results?"

**Numbers and routing**

- "Find an available tracking number in area code 801"
- "Which tracking numbers have no forwarding destination?"
- "Get the DNI snippet for our website"

**SMS**

- "Show unassigned SMS threads from the last 3 days"
- "Schedule a broadcast to the Spring Promo contact list for Friday at 10 AM"

**Account and integrations**

- "List the webhooks on this account and where they send data"
- "Who has Admin access?"

**In your codebase**

- "Add the Convirza DNI snippet to our site layout"
- "Build an endpoint that receives Convirza event subscriptions and verifies the signature"
- "Write a client that pulls yesterday's calls from the Convirza API"

## Troubleshooting

| Problem                                    | Fix                                                                            |
| ------------------------------------------ | ------------------------------------------------------------------------------ |
| Tools fail with "Not authenticated" or 401 | The session expired. Open **Cursor Settings → MCP**, then reconnect Convirza   |
| Sign-in says your role is not allowed      | Only Admin, Standard and Read Only users can connect. Ask your account admin   |
| A write fails with 403                     | Your role doesn't allow it. Read Only users can't write; some tools need Admin |
| Too many tools are active                  | Turn off the Convirza tools you don't need in **Cursor Settings → MCP**        |

## Write actions

Write tools (create, update, delete, move, add a number, send a broadcast) run as soon as they are called; there is no preview step. Every tool is labelled read-only, destructive or open-world, so Cursor can ask you before running one. Read what the agent proposes before you approve it.

Billing and plan changes aren't supported. For those, contact Convirza support.
