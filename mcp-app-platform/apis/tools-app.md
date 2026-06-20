# PrivOS MCP Tools — App

## `mcpapp.app.getLocalData`

Get the app's localData storage. This is per-app storage unique to each MCP app instance.

| | |
|---|---|
| **Scope** | None (app-scoped, no user data access) |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `key` | string | No | Specific key to retrieve (returns all if omitted) |

### Response

```json
{
  "data": {
    "apiKey": "xxx",
    "preferences": { "theme": "dark" },
    "cache": { "users": [...] }
  }
}
```

Or when `key` is specified:

```json
{
  "data": {
    "apiKey": "xxx"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `data` | object | All app data (when no key specified) or specific key-value pair (when key specified) |

### Example (SDK)

```typescript
// Get all localData
const allData = await app.callServerTool({
  name: 'mcpapp.app.getLocalData',
  arguments: {}
});
// { data: { apiKey: "xxx", preferences: { theme: "dark" } } }

// Get specific key
const apiKey = await app.callServerTool({
  name: 'mcpapp.app.getLocalData',
  arguments: { key: 'apiKey' }
});
// { data: { apiKey: "xxx" } }
```

### Example (React Hook)

```tsx
import { useServerTool } from '@anthropic/mcp-react-sdk';

function MyComponent() {
  const { data } = useServerTool('mcpapp.app.getLocalData');
  return <div>API Key: {data?.apiKey}</div>;
}
```

---

## `mcpapp.app.setLocalData`

Set/update the app's localData. Merges with existing data by default.

| | |
|---|---|
| **Scope** | None (app-scoped) |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `data` | object | Yes | Data to set (key-value pairs) |
| `merge` | boolean | No | Merge with existing (true) or replace (false). Default: true |

### Response

```json
{
  "success": true,
  "data": {
    "apiKey": "new-key",
    "preferences": { "theme": "dark" }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | true if operation succeeded |
| `data` | object | The updated localData after the operation |

### Example (SDK)

```typescript
// Merge with existing data (default)
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: {
      apiKey: 'new-secret',
      preferences: { theme: 'dark' }
    },
    merge: true
  }
});
// Result: { success: true, data: { apiKey: "new-secret", preferences: { theme: "dark" }, existingField: "unchanged" } }

// Replace entire localData
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: { freshStart: true },
    merge: false
  }
});
// Result: { success: true, data: { freshStart: true } }
```

### Example (React Hook)

```tsx
import { useServerToolCallback } from '@anthropic/mcp-react-sdk';

function Settings() {
  const updateTheme = useServerToolCallback('mcpapp.app.setLocalData');

  const handleThemeChange = (theme: string) => {
    updateTheme({
      data: { preferences: { theme } },
      merge: true
    });
  };

  return <button onClick={() => handleThemeChange('dark')}>Set Dark Theme</button>;
}
```

---

## `mcpapp.app.deleteLocalData`

Delete specific key(s) from localData.

| | |
|---|---|
| **Scope** | None (app-scoped) |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `key` | string | Yes | Key to delete (use dot notation for nested) |

### Response

```json
{
  "success": true,
  "data": {
    "deletedKey": "apiKey"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | true if operation succeeded |
| `data.deletedKey` | string | The key that was deleted |
| `data.message` | string | Optional message (e.g., if key didn't exist) |

### Example (SDK)

```typescript
// Delete top-level key
await app.callServerTool({
  name: 'mcpapp.app.deleteLocalData',
  arguments: { key: 'apiKey' }
});
// Result: { success: true, data: { deletedKey: "apiKey" } }

// Delete nested key using dot notation
await app.callServerTool({
  name: 'mcpapp.app.deleteLocalData',
  arguments: { key: 'preferences.theme' }
});
// Result: { success: true, data: { deletedKey: "preferences.theme" } }
```

### Example (React Hook)

```tsx
import { useServerToolCallback } from '@anthropic/mcp-react-sdk';

function ClearCache() {
  const clearCache = useServerToolCallback('mcpapp.app.deleteLocalData');

  return (
    <button onClick={() => clearCache({ key: 'cache' })}>
      Clear Cache
    </button>
  );
}
```

---

## `mcpapp.app.clearLocalData`

Clear all app localData. Use with caution — this cannot be undone.

| | |
|---|---|
| **Scope** | None (app-scoped) |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `confirm` | boolean | Yes | Must be `true` to confirm clearing all data |

### Response

```json
{
  "success": true,
  "data": {
    "message": "All localData cleared successfully"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | true if operation succeeded |
| `data.message` | string | Confirmation message |

### Example (SDK)

```typescript
// Clear all data (requires explicit confirmation)
await app.callServerTool({
  name: 'mcpapp.app.clearLocalData',
  arguments: { confirm: true }
});
// Result: { success: true, data: { message: "All localData cleared successfully" } }

// Attempt without confirmation - will fail
await app.callServerTool({
  name: 'mcpapp.app.clearLocalData',
  arguments: { confirm: false }
});
// Error: Must confirm by setting confirm=true
```

### Example (React Hook)

```tsx
import { useServerToolCallback } from '@anthropic/mcp-react-sdk';

function ResetApp() {
  const clearData = useServerToolCallback('mcpapp.app.clearLocalData');

  const handleReset = () => {
    if (confirm('Are you sure you want to clear all app data?')) {
      clearData({ confirm: true });
    }
  };

  return <button onClick={handleReset}>Reset App Data</button>;
}
```

---

## Use Cases

### 1. Store App Configuration

```typescript
// Save app settings on first run
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: {
      config: {
        defaultCity: 'Hanoi',
        units: 'celsius',
        language: 'en'
      }
    }
  }
});
```

### 2. Cache Expensive Operations

```typescript
// Cache API response
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: {
      cache: {
        users: fetchedUsers,
        lastUpdate: new Date().toISOString()
      }
    }
  }
});

// Retrieve cached data
const { data } = await app.callServerTool({
  name: 'mcpapp.app.getLocalData',
  arguments: { key: 'cache' }
});
```

### 3. Maintain App State

```typescript
// Track user progress
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: {
      state: {
        onboardingComplete: true,
        lastAction: 'imported_data',
        currentStep: 3
      }
    }
  }
});
```

### 4. Store API Keys/Tokens

```typescript
// Securely store app-specific credentials
await app.callServerTool({
  name: 'mcpapp.app.setLocalData',
  arguments: {
    data: {
      credentials: {
        apiKey: process.env.API_KEY,
        webhookUrl: 'https://api.example.com/webhook'
      }
    }
  }
});
```

---

## Security & Limitations

- **App-scoped only**: Each app can only access its own `localData`
- **No user data access**: These tools don't provide access to user or room data
- **MongoDB storage**: Subject to MongoDB document size limit (16MB)
- **Not shared across instances**: Each `appId` has its own isolated `localData`
- **No OAuth scope required**: Apps manage their own data without special permissions

---

## Best Practices

1. **Use descriptive keys**: Organize data with clear, hierarchical keys
   ```typescript
   {
     "cache": { ... },
     "preferences": { ... },
     "state": { ... }
   }
   ```

2. **Handle missing data gracefully**: Always check for undefined/null
   ```typescript
   const { data } = await app.callServerTool({
     name: 'mcpapp.app.getLocalData',
     arguments: {}
   });
   const apiKey = data?.apiKey || 'default-key';
   ```

3. **Use merge mode**: Default to `merge: true` to preserve existing data
   ```typescript
   await app.callServerTool({
     name: 'mcpapp.app.setLocalData',
     arguments: { data: newSettings, merge: true }
   });
   ```

4. **Clean up unused data**: Delete old keys to avoid bloat
   ```typescript
   await app.callServerTool({
     name: 'mcpapp.app.deleteLocalData',
     arguments: { key: 'old.cache' }
   });
   ```

5. **Document your schema**: Since `localData` is flexible, document what keys your app uses
   ```typescript
   /**
    * App LocalData Schema:
    * - config: { defaultCity, units, language }
    * - cache: { users[], lastUpdate }
    * - state: { onboardingComplete, currentStep }
    */
   ```
