# Rooms API

Rooms are conversation channels in the system. This API provides room queries and member information.

## Base URL

```
/api/v1/internal/rooms
```

## Endpoints

### Get Rooms

```http
GET /api/v1/internal/rooms.get?filter=search&types[]=c&count=50&offset=0
```

Retrieve rooms with filtering and pagination.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filter` | string | No | Search filter for room name |
| `types` | array | No | Room types to filter (c=channel, p=group, d=dm) |
| `count` | number | No | Items per page (default: 50) |
| `offset` | number | No | Pagination offset (default: 0) |
| `sort` | string | No | Sort field (e.g., `{"name": 1}`) |

**Request Examples:**

```bash
# Get channels only
GET /api/v1/internal/rooms.get?types[]=c

# Search rooms
GET /api/v1/internal/rooms.get?filter=project

# Paginated results
GET /api/v1/internal/rooms.get?offset=50&count=25
```

**Response:**

```json
{
  "success": true,
  "data": {
    "rooms": [
      {
        "_id": "ROOM_ID",
        "name": "project-alpha",
        "t": "c",
        "fname": "Project Alpha",
        "ro": false,
        "archived": false,
        "topic": "Project discussion"
      }
    ],
    "count": 10,
    "offset": 0,
    "total": 45
  }
}
```

---

### Get Room Members

```http
GET /api/v1/internal/rooms.members?roomId=ROOM_ID
```

Get all members of a room with their positions and skills.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "users": [
      {
        "_id": "USER_1",
        "username": "john.doe",
        "name": "John Doe",
        "position": {
          "_id": "pos_1",
          "name": "Developer"
        },
        "skills": [
          {
            "_id": "skill_1",
            "name": "JavaScript"
          },
          {
            "_id": "skill_2",
            "name": "React"
          }
        ]
      },
      {
        "_id": "USER_2",
        "username": "jane.smith",
        "name": "Jane Smith",
        "position": {
          "_id": "pos_2",
          "name": "Designer"
        },
        "skills": [
          {
            "_id": "skill_3",
            "name": "Figma"
          }
        ]
      }
    ],
    "total": 2
  }
}
```

---

### Get User Info in Room

```http
GET /api/v1/internal/rooms.user.info?roomId=ROOM_ID&userId=USER_ID
```

Get detailed information about a specific user within a room.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `userId` | string | Yes | User ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "user": {
      "_id": "USER_1",
      "username": "john.doe",
      "name": "John Doe",
      "position": {
        "_id": "pos_1",
        "name": "Developer"
      },
      "skills": [
        {
          "_id": "skill_1",
          "name": "JavaScript"
        },
        {
          "_id": "skill_2",
          "name": "React"
        }
      ]
    }
  }
}
```

---

## Room Object

```typescript
interface IRoom {
  _id: string;
  name: string;
  fname: string;          // Full name (display name)
  t: string;              // Type: 'c'=channel, 'p'=group, 'd'=dm
  ro: boolean;            // Read-only
  archived: boolean;
  topic?: string;         // Room topic/description
  prid?: string;          // Parent room ID (for discussions)
  broadcast?: boolean;    // Is broadcast room
}
```

## Room Types

| Type | Code | Description |
|------|------|-------------|
| Channel | `c` | Public channel |
| Private Group | `p` | Private group |
| Direct Message | `d` | Direct message |

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Room ID is required |
| `error-room-not-found` | Room not found |
| `error-user-not-in-room` | User is not a member of the room |

## Related Resources

- [Channels API](./channels.md) - Channel-specific operations
- [Groups API](./groups.md) - Group-specific operations
- [Users API](./users.md) - User management
