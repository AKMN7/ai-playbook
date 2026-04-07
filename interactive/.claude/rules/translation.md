# Translation process rules

These rules define how translations **must** be managed in this project.

## 1 — Supported languages

- **English** and **Arabic** at minimum.
- Additional locales (e.g. `fr.json`) may be added when an event is international — the same rules below apply to every locale file.

## 2 — File locations

- Locale files live in `src/locales/`.
- One flat JSON file per language (e.g. `en.json`, `ar.json`).

## 3 — Structure: flat object only

- All translation keys live in a **single flat object**.
- **No nesting** — no `{ "auth": { "signIn": "..." } }`.
- Use a single level of key-value pairs.

```json
// ✅ GOOD
{
  "sign-in": "Sign In",
  "forgot-password": "Forgot Password",
  "email-is-required": "Email is required."
}

// ❌ BAD
{
  "auth": {
    "signIn": "Sign In",
    "forgotPassword": "Forgot Password"
  }
}
```

## 4 — Key naming: kebab-case

- All keys **must** use `kebab-case`.
- Words are lowercase and separated by hyphens.

```json
// ✅ GOOD
"confirm-password"
"email-is-required"
"select-date-range"

// ❌ BAD
"confirmPassword"
"email_is_required"
"SelectDateRange"
```

## 5 — Naming: clear and context-relevant

- Keys should be **descriptive** and reflect where they are used.
- Prefer specificity over brevity when it clarifies context.
- Use suffixes for variants when needed (e.g. `-q` for question, `-dots` for placeholder).
- Exhibit text should be **concise** — visitors scan, not read.

| Context         | Example Key         | Example Value        |
| --------------- | ------------------- | -------------------- |
| Button label    | `sign-in`           | "Sign In"            |
| Form validation | `email-is-required` | "Email is required." |
| Placeholder     | `search-event-dots` | "Search event..."    |
| Question        | `forgot-password-q` | "Forgot Password?"   |

### 5.1 — Breadcrumb & route segment keys

- **Use segment names directly as keys** — No `breadcrumb-` or other prefix.
- Use `t(el.name)` and `t(active.name)` directly in the component.

```tsx
// ✅ GOOD
<Link to={el.path}>{t(el.name)}</Link>
<BreadcrumbPage>{t(active.name)}</BreadcrumbPage>

// ❌ BAD
<Link to={el.path}>{t(`breadcrumb-${el.name}`)}</Link>
```

### 5.2 — Singular vs plural

- Use **singular** when the context is about one item.
- Use **plural** when the context is about many items.

| Context                        | Key                 | Example Value     |
| ------------------------------ | ------------------- | ----------------- |
| Placeholder: select one event  | `select-event-dots` | "Select event..." |
| Empty state: no events in list | `no-events-found`   | "No events found" |
| List/group heading             | `events`            | "Events"          |

### 5.3 — Reuse keys across contexts

- When the same word appears in multiple places with the **same value**, use a **single key**.

```json
// ✅ GOOD — one key for "Dashboard" everywhere
"dashboard": "Dashboard"

// ❌ BAD — redundant keys for same value
"dashboard": "Dashboard",
"breadcrumb-dashboard": "Dashboard"
```

### 5.4 — Do not internationalise `alt` text

- **Do not** use `t()` or locale keys for HTML **`alt`** attributes on images.
- Keep `alt` as a **static string** or use **`alt=""`** for decorative images.

```tsx
// ✅ GOOD
<img src={url} alt="Avatar" />

// ❌ BAD
<img src={url} alt={t("avatar-alt")} />
```

## 6 — Adding or modifying keys

- Add or modify keys in **all** locale files in the same change.
- Keep keys **alphabetically sorted** for easier maintenance.
- Never leave a key in one file without its counterpart in the others.
- No duplicate keys allowed. Always make sure of that.

## 7 — Do not modify

- **Never modify** `src/components/ui/` — shadcn/ui components are managed by the CLI; do not edit them manually.
