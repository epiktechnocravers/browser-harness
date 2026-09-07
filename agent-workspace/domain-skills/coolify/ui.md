# Coolify (self-hosted PaaS) — web UI

Tested against **Coolify v4.3.17**. Laravel + Livewire + Alpine.js, TailwindCSS.

## URL patterns — navigate directly, don't click through

Application pages are plain server-rendered routes. The left settings nav is real
`<a href>` elements, so `goto_url()` beats hunting for a click target:

```
/project/{project_uuid}/environment/{environment_uuid}/application/{app_uuid}
  ├─ (no suffix)  General — name, description, domains, internal access
  ├─ /advanced    Build / Container / Deployment / Git / Proxy toggles
  ├─ /source      Git Source (repo, branch, source provider)
  ├─ /webhooks    webhook URLs + secrets
  ├─ /environment-variables
  ├─ /persistent-storage
  ├─ /healthcheck
  ├─ /logs        Runtime Logs
  ├─ /deployment  Deployment Logs (history table)
  └─ /terminal
```

The environment segment is a **uuid**, not the name — `/environment/njrchh6…/`, not
`/environment/production/`. Get it from the Coolify DB (`environments.uuid`) or by
opening the app once in the UI.

Databases and services follow the same shape with `/database/{uuid}` and `/service/{uuid}`.

## Settings nav re-renders as you navigate

The nav is **not** a fixed list. It shows different section groups depending on which page
is loaded (`Settings` / `Observe & troubleshoot` / `Deploy` / `Automation`), and items shift
position between pages. Coordinates captured on one page are wrong on the next — this is the
main way a click lands on the wrong item here.

Extract the hrefs instead:

```js
Array.from(document.querySelectorAll('a'))
  .filter(e => /advanced|git source|webhooks/i.test(e.textContent || ''))
  .map(e => e.getAttribute('href') + ' | ' + e.textContent.trim())
```

## Livewire swaps panel content asynchronously

Clicking a nav item updates the highlight immediately but the right-hand panel is replaced by
a Livewire round trip. `wait_for_load()` does **not** cover it — it's an XHR, not a navigation.
A screenshot taken right after the click shows the *previous* panel, which reads as "the click
didn't work" when it did.

Confirm by querying for content that only exists on the target page, then screenshot:

```js
Array.from(document.querySelectorAll('button'))
  .some(e => /Deploy on push/i.test(e.textContent || ''))   // true once /advanced rendered
```

## Dropdowns are custom, not `<select>`

Coolify's form dropdowns are Alpine components: a `<button>` showing the current value plus a
popup `<ul>`. `document.querySelectorAll('select')` returns nothing for them, and reading
`.value` / `.options` fails. Find them by the button's visible text, click to open, then click
the option row.

## Where the deploy-on-push switch lives

`/advanced` → **Deployment** section → **"Auto deploy"**. Two options:

- `Deploy on push (webhooks)` — build on every push to the configured branch
- `Manual deployments only` — pushes do nothing

Adjacent: **Preview deployments** (`Disabled` / enabled) controls whether PRs build.

## Reading deployment history

`/deployment` lists past deployments with a **Source** column that distinguishes how each was
triggered — `Webhook` (git push / auto-deploy) vs `API` (manual or API-triggered). That column
is the quickest way to verify an auto-deploy setting actually took effect.

## Status pill

The header breadcrumb ends with a status pill (`Running` / `Exited` / `Degraded`) that reflects
container state, not deployment state. A green pill in the page `<title>` (`🟢 name > …`) can
lag the pill in the body — trust the body.

## Auth

Session cookie based. If you land on `/login`, stop and ask the user rather than typing
credentials. An already-authenticated Chrome profile goes straight through.

## Related: the REST API is often faster

For read-only inventory work, `GET /api/v1/applications` etc. beats the UI entirely. Caveats:
the API sits behind an instance-wide enable flag **and** an IP allowlist (Settings → API), and
it exposes no endpoint for listing git sources — those must come from the `github_apps` table
in Coolify's own Postgres.
