# Users API

Users API provides access to user information including positions and skills.

## Base URL

```
/api/v1/internal/users
```

## Endpoints

### List Users

```http
GET /api/v1/internal/users.list?query={}&fields={}&offset=0&count=50
```

Retrieve a paginated list of users with filtering and field selection.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | object | No | MongoDB query object (JSON stringified) |
| `fields` | object | No | Field projection (JSON stringified) |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50) |

**Request Examples:**

```bash
# Get all active users
GET /api/v1/internal/users.list?query={"active":true}&count=100

# Get specific fields only
GET /api/v1/internal/users.list?fields={"username":1,"name":1,"email":1}

# Search by name
GET /api/v1/internal/users.list?query={"name":{"$regex":"john","$options":"i"}}
```

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
        "active": true,
        "language": "en",
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
    ],
    "count": 1,
    "offset": 0,
    "total": 150
  }
}
```

**Query Operators Supported:**

- `$and`, `$or` - Logical operators
- `$regex` - Regular expression search
- `$in`, `$nin` - In / Not in array
- `$eq`, `$ne` - Equal / Not equal
- `$gt`, `$gte`, `$lt`, `$lte` - Comparison

---

### Get User Info

```http
GET /api/v1/internal/users.info?userId=USER_ID&fields={}
```

Get detailed information about a specific user.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | string | Yes | User ID |
| `fields` | object | No | Field projection (JSON stringified) |

**Request Example:**

```bash
GET /api/v1/internal/users.info?userId=USER_1&fields={"username":1,"name":1,"position":1,"skills":1}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "user": {
      "_id": "USER_1",
      "username": "john.doe",
      "name": "John Doe",
      "active": true,
      "language": "en",
      "nameInsensitive": "john doe",
      "position": {
        "_id": "pos_1",
        "name": "Senior Developer"
      },
      "skills": [
        {
          "_id": "skill_1",
          "name": "JavaScript"
        },
        {
          "_id": "skill_2",
          "name": "TypeScript"
        },
        {
          "_id": "skill_3",
          "name": "Node.js"
        }
      ]
    }
  }
}
```

---

## User Object

```typescript
interface IUser {
  _id: string;
  username: string;
  name: string;
  active: boolean;
  language?: string;
  position?: IPosition['_id'];
  skills?: ISkillReference[];
  nameInsensitive?: string;  // Lowercase name for searching
}

interface IPosition {
  _id: string;
  name: string;
}

interface ISkillReference {
  _id: string;
  name: string;
}

interface ISkill {
  _id: string;
  name: string;
}
```

## Field Projection

Use the `fields` parameter to limit returned data:

```json
{
  "username": 1,
  "name": 1,
  "active": 1,
  "position": 1,
  "skills": 1
}
```

Returns only specified fields. Position and skills are automatically included and enriched.

## Common Queries

### Get Active Users Only

```http
GET /api/v1/internal/users.list?query={"active":true}
```

### Get Users by Position

```http
GET /api/v1/internal/users.list?query={"position":"pos_1"}
```

### Get Users by Skill

```http
GET /api/v1/internal/users.list?query={"skills.skillId":"skill_1"}
```

### Search Users by Name

```http
GET /api/v1/internal/users.list?query={"name":{"$regex":"john","$options":"i"}}
```

### Get Admin Users

```http
GET /api/v1/internal/users.list?query={"roles":"admin"}
```

---

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | User ID is required |
| `error-invalid-query` | Query structure is invalid |
| `error-user-not-found` | User not found |

## Response Pagination

Large result sets are automatically paginated. Use `offset` and `count` for navigation:

```bash
# First page
GET /api/v1/internal/users.list?count=50&offset=0

# Second page
GET /api/v1/internal/users.list?count=50&offset=50
```

## Related Resources

- [Rooms API](./rooms.md) - Get users within specific rooms
- [Channels API](./channels.md) - Get channel members
- [Groups API](./groups.md) - Get group members
