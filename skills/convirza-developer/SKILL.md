---
name: convirza-developer
description: Wire Convirza into the code in this workspace — install the website call tracking (DNI) snippet in the site's layout, build an endpoint that receives Convirza event subscriptions or call webhooks, or write a client that calls the Convirza API with OAuth credentials. Use when someone is coding against Convirza, asks to "add Convirza tracking to the site", "handle Convirza webhooks", "verify the webhook signature" or "call the Convirza API from our app".
---

# Convirza developer workflows

This skill writes code in the user's repository and configures their Convirza account. Code changes are shown as a diff before they are applied; account changes follow the confirm-first rules of the `convirza-integrations` and `convirza-number-provisioning` skills.

**Secrets never go in code or chat.** Signing secrets and API secrets reach the user only through one-time links. The code reads them from environment variables or the project's secret manager. Never write a secret value into a file, a test fixture or a commit, and never ask the user to paste one into the conversation.

## Workflow — website call tracking (DNI)

1. `list_dni_settings` to confirm DNI is set up for the org unit. If it is not, setting it up with `create_dni_setting` changes what visitors see on the live site — confirm that separately (see `convirza-number-provisioning`).
2. `get_dni_snippet` with the `org_unit_ids`. It returns the code to paste into the `<head>`, placement guidance and verification steps. An org unit without DNI, or outside the user's access, comes back under Errors.
3. Find the **one shared head** that every page renders: the root layout, `_document`, `index.html`, or the base template. Do not paste the snippet into individual pages.
4. Search the codebase for an existing Convirza snippet first. If one is there, compare it with the new one and replace it — never install two.
5. Insert the snippet unchanged, show the diff, and wait for approval. Then hand the user the verification steps `get_dni_snippet` returned; the swap can only be checked on a deployed page.

## Workflow — receiving events

Read the "Two ways to send data out" table in the `convirza-integrations` skill first. Event subscriptions are signed; call webhooks are not.

### Event subscription receiver (signed)

Build and deploy the endpoint **before** calling `create_subscription`. Creation sends a test request and fails unless the endpoint answers 2xx within 5 seconds.

The endpoint must:

1. **Read the raw body.** The signature covers the exact bytes Convirza sent. Verify before parsing; re-serialising parsed JSON changes the bytes and breaks the check.
2. **Answer the creation test.** It carries the header `X-Webhook-Test: true`, has a body of `{ "test": true, ... }` and is **not signed**. Return 200 and do nothing else.
3. **Check the signature.** `X-Webhook-Signature` is the hex HMAC-SHA256 of the raw body, keyed with the signing secret. Compare in constant time. Reject a mismatch with 401.
4. **Reject replays.** The body's `delivery_timestamp` is inside the signed payload. Reject deliveries older than a few minutes.
5. **Drop duplicates.** `X-Webhook-Idempotency-Key` (also `idempotency_key` in the body) stays the same across retries; `X-Webhook-ID` changes on every attempt. Store the idempotency key and ignore repeats.
6. **Answer fast.** Return 2xx within 10 seconds and do slow work afterwards. Three failed attempts disable the subscription.

Use `response_format` "JSON". Do not add a custom header whose name starts with `X-Webhook-`.

Node.js (Express) reference — adapt it to the project's framework and language:

```js
import crypto from "node:crypto";
import express from "express";

const app = express();

app.post("/convirza/events", express.raw({ type: "*/*" }), (req, res) => {
  if (req.get("X-Webhook-Test") === "true") return res.sendStatus(200);

  const expected = crypto
    .createHmac("sha256", process.env.CONVIRZA_WEBHOOK_SECRET)
    .update(req.body)
    .digest("hex");
  const given = req.get("X-Webhook-Signature") ?? "";
  if (
    given.length !== expected.length ||
    !crypto.timingSafeEqual(Buffer.from(given), Buffer.from(expected))
  ) {
    return res.sendStatus(401);
  }

  const event = JSON.parse(req.body.toString("utf8"));
  if (Date.now() - Date.parse(event.delivery_timestamp) > 5 * 60 * 1000)
    return res.sendStatus(400);

  res.sendStatus(200);
  // Then: skip if event.idempotency_key was already handled; act on event.event_type and event.entity_id.
});
```

Once deployed: create the subscription (Admin, confirm first). It returns `webhook_secret_link`. Tell the user to open it within 10 minutes and store the secret as `CONVIRZA_WEBHOOK_SECRET` in their environment or secret manager.

### Call webhook receiver (not signed)

Call webhooks carry no signature, so the endpoint authenticates the sender itself:

- HTTPS only, with an unguessable path.
- A shared token sent as a `static_parameters` entry (POST, so it travels in the body rather than the URL). Generate it with a CSPRNG, store it in the user's environment, and compare it in constant time.
- Say plainly in the code comments and to the user that these deliveries are unsigned.

Then create the webhook and the call action template that triggers it (see `convirza-integrations`).

## Workflow — calling the Convirza API

1. `create_oauth_credentials` (Admin, confirm first) returns a `client_id` and `client_secret_link`. The user opens the link within 10 minutes and stores both values as environment variables, e.g. `CONVIRZA_API_KEY` and `CONVIRZA_API_SECRET`.
2. Exchange them for a token: `POST https://platform-oauth.convirza.com/token/client-credentials` with the JSON body `{ "grant_type": "client_credentials", "api_key": <client_id>, "api_secret": <client_secret> }`. The response is `{ access_token, token_type: "Bearer", expires_in: 604800 }`.
3. Call the API with `Authorization: Bearer <access_token>`. Cache the token and fetch a new one before the 7 days run out, or when a request returns 401.
4. Production base URLs: OAuth `https://platform-oauth.convirza.com`, API `https://platform-api.convirza.com/v3`. Read them from configuration (e.g. `CONVIRZA_OAUTH_URL`, `CONVIRZA_API_URL`) with these as the defaults, rather than hard-coding them in each call.
5. The API's field names match the MCP tool parameters (snake_case). Lists are paged with `limit` and a 0-based `offset` or `page`, and return `total`.

## Common issues

| Issue                                  | What to do                                                                                                           |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Every signature check fails            | The body was parsed before verification, or the wrong secret is loaded. Verify against the raw bytes                 |
| `create_subscription` fails with 400   | The endpoint is not deployed, is not HTTPS, took over 5 seconds, or rejected the unsigned test request               |
| The same event is handled twice        | Retries reuse `X-Webhook-Idempotency-Key`. De-duplicate on it, not on `X-Webhook-ID`                                 |
| Token request returns 401              | Wrong key or secret, or the credential was rotated or deleted. Check with `list_oauth_credentials`                   |
| Phone numbers on the site never change | DNI is not enabled for that org unit, or the snippet is missing from the shared head. Recheck with `get_dni_snippet` |

## When to decline or ask first

- **Decline** to write a secret, token or API key into source code, a config file under version control, a test or a commit.
- **Decline** to skip signature verification on a subscription receiver to "get it working".
- **Ask first** before any account write (`create_subscription`, `create_webhook`, `create_oauth_credentials`, `create_dni_setting`), and show the code diff before editing files.
- **Stop** if the user wants call recordings or transcripts sent to a third-party system and has not confirmed they may share that content.
