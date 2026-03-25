# App Platform — React SDK Reference

Package: `@privos/app-react`

Thin React wrapper around the MCP `@modelcontextprotocol/ext-apps` SDK with Privos-specific convenience hooks.

## Hooks

| Hook | Returns | Description |
|------|---------|-------------|
| `usePrivosApp()` | `McpApp` | MCP app instance for direct `callServerTool()` |
| `usePrivosContext()` | `PrivosContext` | userId, roomId, theme, userRoles |
| `usePrivosTool(name, args)` | `{ data, loading, error, refetch }` | Generic auto-fetching tool call |
| `useLists(roomId)` | `{ data, loading, error }` | Lists in room |
| `useFiles(roomId)` | `{ data, loading, error }` | Files in room |
| `useRoom(roomId?)` | `{ data, loading, error }` | Room metadata |

## Provider

Wrap your app with `PrivosAppProvider`:

```tsx
import { PrivosAppProvider } from '@privos/app-react';

export default function App() {
  return (
    <PrivosAppProvider>
      <MyComponent />
    </PrivosAppProvider>
  );
}
```

## usePrivosApp

Returns the MCP app instance. Use for mutations (writes) where you call tools directly:

```tsx
const app = usePrivosApp();

await app.callServerTool({
  name: 'privos.lists.createItem',
  arguments: { listId: 'abc', title: 'New item' }
});
```

## usePrivosContext

Subscribes to `HOST_CONTEXT_CHANGED` push + fetches Privos-specific context:

```tsx
const { roomId, userId, username, roomName, theme, surfaceColor, userRoles } = usePrivosContext();
```

- `theme` (`'light'` | `'dark'`) — updates in real-time when the user toggles theme
- `surfaceColor` (e.g. `'#080d0f'`) — exact room background color for pixel-perfect matching

Use with a `ThemeProvider` for Auto/Light/Dark mode support. See [Developer Guide — Theme Sync](./developer-guide.md#7-theme-sync-lightdark-mode).

## usePrivosTool

Generic hook — auto-fetches on mount and when args change. Best for reads:

```tsx
const { data, loading, error, refetch } = usePrivosTool('privos.lists.get', { listId });
```

**Note:** Skips fetch if any arg value is empty/null/undefined.

## useLists

Convenience wrapper for `privos.lists.getAll`:

```tsx
const { data: lists, loading, error } = useLists(roomId);
```

## Pattern: Reads vs Mutations

- **Reads** — use `usePrivosTool` or convenience hooks (`useLists`, `useFiles`, `useRoom`). Auto-fetches.
- **Mutations** — use `usePrivosApp()` to get the app instance, then call `app.callServerTool()` directly in event handlers.
