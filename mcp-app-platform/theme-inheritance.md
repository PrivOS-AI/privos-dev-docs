# App Platform — Theme Inheritance

How an MCP app visually inherits the workspace's configured theme (colours, radius, font,
light/dark mode) without ever reading the Hub's own stylesheet.

## Why this exists

An MCP app renders in a sandboxed, cross-origin iframe (see
[Security & Data Model](./security-and-data-model.md)). It cannot read the parent document's DOM
or its CSS custom properties — a hardcoded app palette is the only fallback available on its own.
To let an app match a workspace's configured colours without breaking that isolation, the Hub
**pushes** a small, curated set of resolved design tokens to the app over the same postMessage
bridge used for all other non-secret host context (`userId`, `roomId`, `theme`, …).

## The contract

### `hostCapabilities.theme`

The first message a Hub sends an app iframe, `ui/initialize`, carries `hostCapabilities`. A Hub
that supports token inheritance sets:

```json
{ "hostCapabilities": { "tools": true, "context": true, "theme": true } }
```

Feature-detect on this rather than probing for the field's presence on `HOST_CONTEXT_CHANGED` —
it lets an app know a Hub is theme-token-aware before the first context push even arrives.

### `theme` and `themeTokens` on `HOST_CONTEXT_CHANGED`

`HOST_CONTEXT_CHANGED` params include:

| Field | Type | Meaning |
|-------|------|---------|
| `theme` | `'light' \| 'dark'` | Resolved light/dark mode (high-contrast resolves to `dark`) |
| `themeTokens` | `Record<string, string>` \| absent | Resolved values for the 12 tokens below, for the **current** mode |

`themeTokens` is **additive and backward-compatible**: an app that ignores it keeps its own
styling exactly as before. It is absent (not an empty object) when the Hub has nothing to offer —
theme styling disabled workspace-wide, or an older Hub that predates the field. A workspace that
never overrode a given seed omits just that one key rather than sending an empty string, so an app
can tell "not provided, use my own default" apart from "explicitly blank".

### The 12 tokens

A small, **stable** subset of the ~180 tokens the Hub's own admin-styling sheet generates — picked
so an app can apply them verbatim as CSS custom properties on its own `:root` and get a
recognisable, on-brand look without importing the Hub's full design system:

| CSS variable | Meaning |
|---|---|
| `--base-primary` | Primary action colour |
| `--base-primary-hover` | Primary action colour, hover state |
| `--base-bg-main` | Main background |
| `--base-bg-menu` | Menu / sidebar background |
| `--base-bg-surface` | Card / surface background |
| `--base-bg-header` | Header / table-head background |
| `--base-border` | Border colour |
| `--base-text-primary` | Primary text colour |
| `--base-text-secondary` | Secondary text colour |
| `--base-info` | Link / info colour |
| `--base-radius-md` | Corner radius |
| `--base-font-family` | Font family |

### Timing — when a push (re-)arrives

`HOST_CONTEXT_CHANGED` (with `themeTokens` for the current mode) is sent:

- once, right after `ui/initialize`, on iframe load;
- again on every light/dark(/high-contrast) mode flip;
- again on a **live admin theme save** — an admin editing and saving the workspace theme while
  this app is open re-pushes fresh `themeTokens` for the *same* mode, so an already-open app picks
  up a colour/radius/font edit without a reload.

An app must not assume `themeTokens` only changes alongside `theme` — a same-mode save is a real,
distinct push.

## How the `@privos_ai/app-react` SDK applies this (`>= 0.6.0`)

Wrapping your app in `PrivosAppProvider` is enough to get inheritance for free — no extra code:

```tsx
import { PrivosAppProvider } from '@privos_ai/app-react';

export default function App() {
  return (
    <PrivosAppProvider>
      <MyComponent />
    </PrivosAppProvider>
  );
}
```

On every `HOST_CONTEXT_CHANGED` the provider:

1. sets `data-theme` on `<html>` to the pushed `theme` (`'light'` | `'dark'`);
2. writes every `themeTokens` entry onto `<html>` via `style.setProperty(name, value)`, so any CSS
   in the app referencing `var(--base-primary)` etc. updates immediately — no React re-render
   required, because it is a real CSS custom property, not app state.

This runs from a module-scope listener registered at import time, so it also catches the very
first push that can otherwise race React's own mount (see the SDK's `PrivosAppProvider.tsx`
comment on `bufferedHostContext`). No-op outside a DOM (SSR, or a test without `jsdom`) and
tolerant of a context carrying neither field, so an app that never adopts theming is unaffected.

## Consuming it in your app

### Option A — zero-config: `var(--base-*)` in CSS

Do nothing beyond wrapping in `PrivosAppProvider`. Reference the tokens directly in your
stylesheet, with a literal fallback so the app still looks correct standalone (outside a Privos
workspace) or before the first push arrives:

```css
.my-button {
  background: var(--base-primary, #156ff5);
  color: #fff;
  border-radius: var(--base-radius-md, 6px);
  font-family: var(--base-font-family, system-ui, sans-serif);
}

.my-button:hover {
  background: var(--base-primary-hover, #095ad2);
}

a {
  color: var(--base-info, var(--base-primary, #156ff5));
}
```

### Option B — read the raw values via `usePrivosContext()`

Use this when you need the actual resolved value rather than a live CSS variable — for example, to
theme a `<canvas>`, or configure a third-party widget that does not respond to CSS custom
properties:

```tsx
import { usePrivosContext } from '@privos_ai/app-react';

function MyWidget() {
  const { theme, themeTokens } = usePrivosContext();

  const primary = themeTokens?.['--base-primary'] ?? '#156ff5';
  const isDark = theme === 'dark';

  return <ThirdPartyChart accentColor={primary} darkMode={isDark} />;
}
```

`themeTokens` is `Record<string, string> | undefined` on the `PrivosContext` type — always guard
with `?.` and a fallback, exactly as with any other optional context field.

## Graceful degradation

Both consumption paths degrade the same way, by design — an app that adopts theming must still
render correctly when nothing arrives:

- **No `themeTokens`** (older Hub, theme styling disabled workspace-wide, or the app running
  standalone outside a Privos workspace entirely): every `var(--base-*, <fallback>)` reference
  resolves to its literal fallback, and `usePrivosContext().themeTokens` is `undefined`. The app
  keeps its own hardcoded palette — nothing breaks.
- **Partial `themeTokens`** (a workspace never overrode one particular seed): the missing key is
  simply absent from the object rather than present with an empty string, so the same
  `var(--base-*, <fallback>)` / `themeTokens?.[key] ?? fallback` pattern handles it identically to
  the "absent entirely" case.

## See it live

The `privos-mcp-app-demo` reference app's **Theme inheritance** tab
(`src/ui/theme-inheritance-panel.tsx`) renders a legend of all 12 tokens with their live resolved
values and colour swatches, plus a primary button / bordered card / link styled purely through
`var(--base-*)` — open it alongside a workspace theme edit to watch it restyle with no reload. See
the demo's [README — Theme inheritance](https://github.com/PrivOS-AI/privos-mcp-app-demo#theme-inheritance)
section.
