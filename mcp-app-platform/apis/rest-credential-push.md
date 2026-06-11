# REST API — Credential Push (Hub → App)

After `POST /api/v1/mcp-apps.connect` succeeds (direct apps only), the Hub performs a **best-effort outbound POST** to the MCP app server. This lets the app self-configure the credentials it needs to call back into PrivOS without manual copy/paste.

| | |
|---|---|
| **Direction** | PrivOS Hub → MCP App Server (outbound from Hub) |
| **Endpoint** | `POST {serverUrl}/.well-known/mcp/register` |
| **Implemented by** | The MCP app developer (optional but recommended) |
| **Timeout** | 10 seconds |
| **Source** | [`mcp-registration-pusher.ts`](../../../apps/meteor/server/services/mcp-registration-pusher.ts) |

## Request

Headers (sent by Hub):

```
Content-Type: application/json
```

Body:

```json
{
  "appId": "my-app",
  "clientId": "client_abc",
  "clientSecret": "secret_xyz",
  "timestamp": "2026-05-25T10:30:00.000Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `appId` | string | From `manifest.name` — stable business identifier |
| `clientId` | string | OAuth client ID, used for the OAuth2 client_credentials grant when the app calls back into PrivOS |
| `clientSecret` | string | OAuth client secret. Shown once — Hub will not re-deliver unless explicitly rotated |
| `timestamp` | string (ISO 8601) | When the Hub sent the push |

## Expected Response

| Your HTTP status | Hub interprets as | Hub behavior |
|---|---|---|
| 2xx | `success` | Credentials delivered, no further action |
| 404 | `skipped` | App intentionally does not support auto-config — no warning logged |
| any other 4xx / 5xx | `failed` | Warning logged with status and error; connect still succeeds |
| timeout / network error | `failed` | Warning logged with error message; connect still succeeds |

The connect flow **does not abort** on push failure — admin can still copy credentials manually from the `mcp-apps.connect` response.

## Implementing the endpoint (app server side)

### Express / Node.js

```typescript
import express from 'express';
import fs from 'node:fs/promises';

const app = express();
const EXPECTED_HUB_HOST = process.env.EXPECTED_HUB_HOST;

app.post('/.well-known/mcp/register', express.json(), async (req, res) => {
  const { appId, clientId, clientSecret, timestamp } = req.body || {};

  if (!appId || !clientId || !clientSecret) {
    return res.status(400).json({ error: 'Missing required fields' });
  }

  // Persist (idempotent — same appId always overwrites)
  await persistCredentials({ appId, clientId, clientSecret, timestamp });
  return res.status(200).json({ ok: true });
});
```

### Opting out

Return `404` from this path (or simply do not implement it). Hub treats 404 as a silent skip — no warning, no retry.

## Security model

This push intentionally has **no HMAC signature** because the app does not yet have a shared secret to verify against (chicken-and-egg). Security relies on:

1. **HTTPS transport** — Hub uses TLS, certificate chain validated normally
2. **Trust-by-URL** — the admin who registered `serverUrl` is the authority on what URL receives credentials
3. **Idempotency** — apps should use `appId` as a deterministic upsert key so retries are safe

### Recommended app-side defenses

- Pin the expected Hub origin/host in app config and reject unexpected sources
- Persist credentials only when all required fields are present and well-formed
- Treat re-registration with the same `appId` as a credential rotation (overwrite, do not duplicate)
- Log registration events for audit (without logging the secret itself)

For deeper rationale, see [security-and-data-model.md → Credential Push to App Server](../security-and-data-model.md#credential-push-to-app-server-direct-apps).

## Related

- [App Management — connect endpoint](./rest-app-management.md#connect-app) — the inbound endpoint that triggers this push
- [Relay Apps — pairing flow](./rest-relay-apps.md) — relay apps receive credentials via the pairing UI (not this push)
