# Brain Proxy API

Lets trusted external services (e.g. **privos-crm**) invoke PrivOS Sandbox through privos-hub instead of holding their own Brain credentials. Brain config is read from the room (`customFields.privosBrain.{url,apiKey,defaultProvider,defaultModel}`) with fallback to global admin settings (`PrivOSBrain_URL` / `PrivOSBrain_API_Key`).

**Auth:** standard Rocket.Chat user auth (`X-Auth-Token` + `X-User-Id`, or `Authorization: Bearer <botToken>`). The caller must be a member of the target `roomId` (verified via Subscriptions).

**SSRF guard:** Brain URL is validated by `validateBrainUrl()` — `http(s)` only, no localhost/private IPs.

---

## POST `/api/v1/agents.brain.upload`

Upload a base64-encoded file to Brain's `/api/uploads`, return a `tempId` that can be reused as `fileIds[]` in subsequent generation calls (or in `agents.reply`).

### Body

| Field | Type | Required | Notes |
|---|---|---|---|
| `roomId` | string | yes | Room used for access check + Brain-config lookup |
| `base64` | string | yes | File bytes, base64-encoded |
| `mimeType` | string | yes | e.g. `image/png` |
| `fileName` | string | yes | Original file name |

### Response

```json
{ "success": true, "tempId": "abc123", "source": "room" | "global" }
```

### Errors

- `No access to target room` — caller is not a member of `roomId`
- `Brain not configured` — neither room nor global Brain settings present
- `Invalid base64 payload`, `PrivOS Sandbox upload <status>: <body>` — propagated from Brain

---

## POST `/api/v1/agents.brain.generate`

One-shot synchronous Brain generation (`POST /api/attempts` with `request_method: 'sync'`). Handles Brain's 408 → background-poll recovery transparently (up to 15 min total). Use for short tasks like title/summary generation.

### Body

| Field | Type | Required | Notes |
|---|---|---|---|
| `roomId` | string | yes | Access check + Brain-config lookup |
| `prompt` | string | yes | Up to 50,000 chars |
| `systemContext` | string | no | Prepended to prompt server-side; up to 200,000 chars |
| `provider` | string | no | Override room/global default |
| `model` | string | no | Override room/global default |
| `projectId` | string | no | Brain projectId (default: `roomId`) |
| `projectName` | string | no | Default: `External Brain Proxy` |
| `taskId` | string | no | Default: `roomId` |
| `taskTitle` | string | no | Default: equals `projectName` |
| `fileIds` | string[] | no | Brain `tempId`s from prior `agents.brain.upload` calls |

### Response

```json
{ "success": true, "text": "<assistant text>", "source": "room" | "global" }
```

### Errors

- `No access to target room`
- `Brain not configured`
- `PrivOS Sandbox <status>: <body>` — Brain non-2xx
- `PrivOS Sandbox attempt <id> did not complete within 900000ms after sync 408` — long-running attempt didn't finish within poll cap

---

## Notes

- Both endpoints are **synchronous**: the HTTP response holds until Brain replies (or 408 polling completes). Use them for tasks that need a single answer.
- For streaming bot replies in a room (where chunks must arrive over DDP), keep using `POST /api/v1/agents.reply`.
- Implementation: `apps/meteor/app/api/server/v1/agent-brain-proxy-endpoints.ts`
