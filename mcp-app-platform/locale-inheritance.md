# App Platform — Locale Inheritance

How an MCP app learns which language the user reads the Hub in, so it can show its own text in
that language and format dates, numbers and currencies the way the user expects.

## Why this exists

An MCP app renders in a sandboxed, cross-origin iframe (see
[Security & Data Model](./security-and-data-model.md)). It cannot read the Hub's `<html lang>` or
the user's profile, and `navigator.language` only reports the browser's language, which is often
not the language the user chose in the Hub. So the Hub **pushes** its UI language to the app over
the same postMessage bridge that carries the theme (see [Theme Inheritance](./theme-inheritance.md)).

## The contract

### `hostCapabilities.locale`

A Hub that sends the locale sets `locale: true` in the `hostCapabilities` of its first message,
`ui/initialize`:

```json
{ "hostCapabilities": { "tools": true, "context": true, "theme": true, "locale": true } }
```

Feature-detect on this flag. Then an app knows the Hub will send a locale before the first context
push arrives.

### `locale` on `HOST_CONTEXT_CHANGED`

| Field | Type | Meaning |
|-------|------|---------|
| `locale` | `string` \| absent | The language the Hub UI is rendering in, as a canonical [BCP 47](https://www.rfc-editor.org/info/bcp47) tag |

- The value is the user's own language (Account → Preferences → Language). When the user never
  picked one, it is the workspace default language (Administration → Settings → General → Language),
  and `en` when that is empty too.
- Examples: `en`, `vi-VN`, `de`, `pt-BR`, `zh-TW`. Some tags carry only a language and no region
  (`en`, `de`), so do not assume a region is present.
- The tag is canonical, so `Intl` accepts it as is. The Hub omits the field rather than send a
  malformed code.
- The field is optional and backward-compatible. An app that ignores it works exactly as before.
  It is absent on a Hub that predates it and when the app runs standalone outside a workspace.

### Timing — when a push (re-)arrives

`HOST_CONTEXT_CHANGED` carries `locale` every time it is sent:

- once, right after `ui/initialize`, on iframe load;
- again when the language the Hub shows changes. That happens when the user picks another
  language, or when an admin changes the workspace default for a user who never picked one. An open
  app receives the new value without a reload;
- on every other context push (theme flip, live theme save, deep-link route), unchanged.

Compare against the previous value if switching language is expensive in your app (for example,
loading a translation bundle). Do not reload on every push.

## Reading it

### With the `@privos_ai/app-react` SDK (`>= 0.9.0`)

`usePrivosContext()` returns `locale` and re-renders when it changes:

```tsx
import { usePrivosContext } from '@privos_ai/app-react';

function OrderTotal({ amount }: { amount: number }) {
  const { locale } = usePrivosContext();
  const lang = locale ?? navigator.language;

  return <span>{new Intl.NumberFormat(lang, { style: 'currency', currency: 'EUR' }).format(amount)}</span>;
}
```

Older SDK versions drop the field: `usePrivosContext()` keeps only the fields it knows.

### Without the SDK

Listen for the push from the parent frame and ignore every other source:

```js
let locale;
window.addEventListener('message', (event) => {
  if (event.source !== window.parent) return;
  const { jsonrpc, method, params } = event.data ?? {};
  if (jsonrpc === '2.0' && method === 'HOST_CONTEXT_CHANGED' && typeof params?.locale === 'string') {
    locale = params.locale;
    render();
  }
});
```

## Using it in your app

### Pick your translation

Match on the language subtag and fall back to a language you ship:

```ts
const SUPPORTED = ['en', 'vi'] as const;
type AppLanguage = (typeof SUPPORTED)[number];

export function pickLanguage(locale: string | undefined): AppLanguage {
  const base = (locale ?? navigator.language ?? 'en').split('-')[0].toLowerCase();
  return (SUPPORTED as readonly string[]).includes(base) ? (base as AppLanguage) : 'en';
}
```

Use the full tag (not just the subtag) for `Intl`, so regional formats stay right: `pt-BR` and
`pt-PT` write numbers differently.

### Format with `Intl`

```ts
const lang = locale ?? navigator.language;
new Intl.DateTimeFormat(lang, { dateStyle: 'long' }).format(new Date());
new Intl.NumberFormat(lang).format(1234567.891);
new Intl.RelativeTimeFormat(lang, { numeric: 'auto' }).format(-1, 'day');
```

### Set `<html lang>` to the language you render

The SDK does not set `<html lang>` for you, unlike `data-theme`. Only the app knows which
language its text is in. An English-only app tagged `vi-VN` makes screen readers read English
text with a Vietnamese voice. Set it to the language you actually picked:

```ts
document.documentElement.lang = pickLanguage(locale);
```

### Your backend does not receive it

`locale` is UI context. Tool calls do not carry it. If a backend tool needs the user's language
(for example, to write a PDF or an email in it), send it in the tool arguments:

```ts
await app.callServerTool({ name: 'export_report', arguments: { reportId, locale: lang } });
```

## Graceful degradation

- **No `locale`** (older Hub, or the app running standalone): use `navigator.language`, then your
  default language. The `locale ?? navigator.language` pattern above covers both.
- **A language you do not ship**: show your default language, but still format dates and numbers
  with the full `locale` tag. `Intl` supports far more locales than any app translates.

## See it live

The `privos-mcp-app-demo` reference app's **Language & locale** tab (`src/ui/locale-panel.tsx`)
shows the pushed `locale`, the language the app picked from it, and a date, a number, a currency
amount and a relative time formatted with it. Switch your Hub language while the tab is open to
watch it update with no reload. See the demo's
[README — Language and locale](https://github.com/PrivOS-AI/privos-mcp-app-demo#language-and-locale)
section.
