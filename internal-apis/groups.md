# Groups API

Groups API provides member listing for private groups and broadcasts.

## Base URL

```
/api/v1/internal/groups
```

## Endpoints

### Get Group Members

```http
GET /api/v1/internal/groups.members?roomId=ROOM_ID&status[]=online&filter=john&offset=0&count=50
```

Get paginated list of members in a private group with filtering.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | No | Room ID (use roomId OR roomName) |
| `roomName` | string | No | Room name (use roomId OR roomName) |
| `status` | array | No | Filter by user status (`online`, `offline`, `away`, `busy`) |
| `filter` | string | No | Search filter for username/name |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50) |
| `sort` | object | No | Sort options (JSON stringified) |

**Request Examples:**

```bash
# By room ID
GET /api/v1/internal/groups.members?roomId=GROUP_ID

# By room name
GET /api/v1/internal/groups.members?roomName=private-team

# Online users only
GET /api/v1/internal/groups.members?roomId=GROUP_ID&status[]=online

# Filter by name
GET /api/v1/internal/groups.members?roomId=GROUP_ID&filter=john

# Sorted by username
GET /api/v1/internal/groups.members?roomId=GROUP_ID&sort={"username":1}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "members": [
      {
        "_id": "USER_1",
        "username": "john.doe",
        "name": "John Doe",
        "status": "online",
        "utcOffset": 7
      },
      {
        "_id": "USER_2",
        "username": "jane.smith",
        "name": "Jane Smith",
        "status": "away",
        "utcOffset": -5
      }
    ],
    "count": 2,
    "offset": 0,
    "total": 10
  }
}
```

---

## Broadcast Groups

For broadcast groups (announcement channels), additional permission checks apply:

- **Permission Required**: `view-broadcast-member-list`
- If user lacks permission, returns `403 Forbidden`

```json
{
  "success": false,
  "error": "error-forbidden"
}
```

---

## User Status Values

| Status | Description |
|--------|-------------|
| `online` | User is active |
| `away` | User is away |
| `busy` | User is busy |
| `offline` | User is offline |

---

## Sorting Options

Sort by any user field:

```json
{
  "username": 1,     // Ascending
  "username": -1,    // Descending
  "name": 1,
  "status": 1
}
```

## Difference from Channels

| Feature | Channels | Groups |
|---------|----------|--------|
| Room Type | Public (`t: 'c'`) | Private (`t: 'p'`) |
| Access Control | Open to join | Invitation only |
| Broadcast | No | Supported |
| Read-Only | Yes | Yes |

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-room-param-not-provided` | Either roomId or roomName is required |
| `error-room-not-found` | Group not found |
| `error-room-archived` | The group is archived |
| `error-forbidden` | Insufficient permissions (for broadcasts) |

## Related Resources

- [Channels API](./channels.md) - Public channel members
- [Rooms API](./rooms.md) - General room operations
- [Users API](./users.md) - User information
