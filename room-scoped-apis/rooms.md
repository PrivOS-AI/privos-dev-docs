# Room-Scoped Rooms API

## Overview

Access room information and member details within a specific room context.

**Base Path:** `/api/v1/internal/rooms/:roomId`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get Room Info

```http
GET /api/v1/internal/rooms/:roomId/info
```

**Description:** Get detailed room information with admin fields.

**Response:**

```json
{
  "success": true,
  "data": {
    "room": {
      "_id": "ROOM_ID",
      "name": "Project Alpha",
      "t": "c",
      "ro": false,
      "default": false,
      "fname": "Project Alpha",
      "topic": "Project discussion channel",
      "description": "Main channel for Project Alpha",
      "announcement": "Sprint starts tomorrow!",
      "customFields": {
        "projectCode": "PROJ-001",
        "teamSize": 5
      },
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-02T00:00:00Z",
      "lastMessage": {
        "_id": "MSG_ID",
        "msg": "Latest message content",
        "u": {
          "_id": "USER_ID",
          "username": "john.doe"
        },
        "ts": "2024-01-02T12:00:00Z"
      },
      "membersCount": 8,
      "msgs": 150
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/info" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get Room Members

```http
GET /api/v1/internal/rooms/:roomId/members
```

**Description:** Get all members in a room with their positions and skills.

**Response:**

```json
{
  "success": true,
  "data": {
    "members": [
      {
        "_id": "USER_ID",
        "username": "john.doe",
        "name": "John Doe",
        "position": {
          "_id": "POS_ID",
          "name": "Software Engineer"
        },
        "skills": [
          {
            "_id": "SKILL_ID",
            "name": "JavaScript"
          },
          {
            "_id": "SKILL_ID_2",
            "name": "React"
          }
        ]
      }
    ],
    "total": 1
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/members" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get Specific Member

```http
GET /api/v1/internal/rooms/:roomId/members/:userId
```

**Description:** Get information about a specific member in the room.

**Response:**

```json
{
  "success": true,
  "data": {
    "member": {
      "_id": "USER_ID",
      "username": "john.doe",
      "name": "John Doe",
      "position": {
        "_id": "POS_ID",
        "name": "Software Engineer"
      },
      "skills": [
        {
          "_id": "SKILL_ID",
          "name": "JavaScript"
        }
      ]
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/members/USER_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get Room Agent Bots

```http
GET /api/v1/internal/rooms/:roomId/agentBots
```

**Description:** Get all agent bots configured for a room. Returns bots that have agent flow configuration (`agent.flowId` set).

**Response:**

```json
{
  "success": true,
  "data": {
    "agents": [
      {
        "id": "flow-id-123",
        "name": "Customer Support Bot",
        "description": "Handles customer inquiries",
        "botUserId": "bot-user-id",
        "botUsername": "support-bot",
        "commands": [
          {
            "name": "help",
            "description": "Show help information",
            "isBuiltIn": true
          }
        ]
      }
    ]
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Agent flow ID (same as flow ID in Privos Studio) |
| `name` | string | Agent display name |
| `description` | string | Agent description |
| `botUserId` | string | PrivOS Chat bot user ID |
| `botUsername` | string | Bot username |
| `commands` | array | Array of commands the agent can execute |

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/agentBots" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Notes:**
- Returns empty array if no agent bots are configured for the room
- Used by AI Chat to populate agent dropdown
- Agents must have `type: 'bot'` and `agent.flowId` set to appear in results
- Falls back to "Default Agent" when no agents are found

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-room-not-found` | Room not found |
| `error-user-not-found` | User not found |
| `error-user-not-in-room` | User is not a member of this room |

---

## Room Type Reference

| Type (`t`) | Description |
|------------|-------------|
| `c` | Channel |
| `p` | Private Group |
| `d` | Direct Message |
| `l` | Live Chat |

## Room Properties

| Property | Type | Description |
|----------|------|-------------|
| `_id` | string | Room ID |
| `name` | string | Room name (for channels) |
| `fname` | string | Full/friendly name |
| `t` | string | Room type |
| `ro` | boolean | Read-only |
| `topic` | string | Room topic |
| `description` | string | Room description |
| `announcement` | string | Room announcement |
| `customFields` | object | Custom room fields |
| `membersCount` | number | Number of members |
| `msgs` | number | Message count |
| `createdAt` | Date | Creation timestamp |
| `updatedAt` | Date | Last update timestamp |
| `lastMessage` | object | Last message details |
