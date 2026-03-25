# Channels API

Channels API provides member listing for public channels and live streams.

## Base URL

```
/api/v1/internal/channels
```

## Endpoints

### Get Channel Members

```http
GET /api/v1/internal/channels.members?roomId=ROOM_ID&status[]=online&filter=john&offset=0&count=50
```

Get paginated list of members in a channel with filtering.

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
GET /api/v1/internal/channels.members?roomId=ROOM_ID

# By room name
GET /api/v1/internal/channels.members?roomName=general

# Online users only
GET /api/v1/internal/channels.members?roomId=ROOM_ID&status[]=online

# Filter by name
GET /api/v1/internal/channels.members?roomId=ROOM_ID&filter=john

# Sorted by username
GET /api/v1/internal/channels.members?roomId=ROOM_ID&sort={"username":1}
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
    "total": 25
  }
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

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-room-param-not-provided` | Either roomId or roomName is required |
| `error-room-not-found` | Channel not found |
| `error-room-archived` | The channel is archived |

## Related Resources

- [Groups API](./groups.md) - Private group members
- [Rooms API](./rooms.md) - General room operations
- [Users API](./users.md) - User information
