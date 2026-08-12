# Brand Hub — Product Specification (v1)

A visual asset hub for the motion design studio. Team members find a branded element,
confirm it is the approved current version, and copy its server path straight into
After Effects, Premiere, or code.

Status: specification agreed, pending brand palette input. See [Open items](#12-open-items).

---

## 1. Locked decisions

| Area | Decision | Why |
|---|---|---|
| Stack | Vite + React + TypeScript + Tailwind, static build | React ergonomics without a Node server to maintain |
| Content source | Sanity CMS (new project `brand-hub`, dataset `production`) | Producers add assets without a developer or a deploy |
| Previews | Web-hosted media from Sanity CDN; server path is text-only metadata | Browsers cannot load `\\server\share` or `file://` from an HTTP page |
| Primary path format | Windows UNC, with Mac and POSIX/URL variants in a dropdown | PC-based suites are the default consumer |
| Broadcast simulation | Cut from v1 | Depends on a proven asset pipeline; revisit once real renders exist |
| Scale target | 50–200 assets, fetched once, filtered client-side | No pagination or virtualization complexity at this size |
| Hosting | Vercel with deployment protection enabled | App exposes internal server topology and must not be publicly indexable |
| Access control | Deferred, but the app ships behind Vercel protection from day one | RBAC is a later phase; obscurity is not access control |

---

## 2. Scope

### In scope for v1
- Responsive asset grid with high-impact visual cards
- Hover-to-play looping video previews for motion assets, still fallback
- Per-brand dynamic theming with accessible token contract
- Search across name, description, tags, and filename
- Filter by brand, category, type, and status
- Detail drawer with large preview, full metadata, and all path variants
- Copy-path split button on every card and in the drawer
- Asset lifecycle: `approved` / `wip` / `deprecated`, with supersede pointers
- Deep-linkable asset URLs for sharing in Slack
- Sanity Studio mounted at `/studio` for content editing

### Explicitly out of v1
- Live preview / broadcast simulation
- Role-based access control and per-brand permissions
- File uploads of the source assets themselves (the hub links, it does not host)
- Usage analytics
- Bulk import from the existing share

---

## 3. The path problem, and how it is solved

This is the constraint the original spec did not account for, and it shapes the data model.

A browser cannot fetch `\\studio-nas\Brand\Elections\l3_v2.mogrt`. Chrome blocks
`file://` requests originating from an HTTP page outright. Therefore:

- **Preview media** is uploaded to Sanity and served over HTTPS from its CDN.
- **The server path** is a string that is only ever rendered and copied. It is never
  used as an `src`, never fetched, never validated by the browser.

These are two separate fields describing the same asset and must never be conflated.

### Path variants are derived, not stored

Storing three hand-typed paths per asset guarantees they drift. Instead each asset stores
one canonical relative path and references a `share` document holding the mount roots:

```
share: {
  title: "Brand Library",
  unc:   "\\\\studio-nas\\Brand",
  mac:   "/Volumes/Brand",
  http:  "https://assets.internal.example/brand"   // optional
}

asset.relPath: "Elections/Lower3rd/l3_election_v2.mogrt"
```

Variants are computed at render time:

| Variant | Result |
|---|---|
| Windows (default) | `\\studio-nas\Brand\Elections\Lower3rd\l3_election_v2.mogrt` |
| macOS | `/Volumes/Brand/Elections/Lower3rd/l3_election_v2.mogrt` |
| POSIX / URL | `https://assets.internal.example/brand/Elections/Lower3rd/l3_election_v2.mogrt` |

When a volume is remounted or the NAS is renamed, one `share` document changes and every
asset follows. Separator conversion and URL-encoding of the HTTP variant happen in a single
`derivePaths()` utility with unit tests.

### Clipboard behaviour

`navigator.clipboard.writeText` requires a secure context. On Vercel over HTTPS this is
satisfied, but a future intranet deployment on plain `http://` would fail silently. The copy
utility therefore tries the async Clipboard API, falls back to a hidden textarea with
`document.execCommand('copy')`, and surfaces a visible failure toast if both fail — never a
button that appears to work and does nothing.

---

## 4. Content model (Sanity)

### `brand`
| Field | Type | Notes |
|---|---|---|
| `name` | string | Display name |
| `slug` | slug | Used in URLs and theme lookup |
| `theme` | object | Token set, see §5 |
| `logo` | image | Shown in the brand switcher and section header |
| `order` | number | Controls nav ordering |

### `share`
| Field | Type | Notes |
|---|---|---|
| `title` | string | Human label, e.g. "Brand Library" |
| `unc` | string | Windows root, no trailing separator |
| `mac` | string | macOS mount root |
| `http` | url | Optional HTTP root for the URL variant |

### `asset`
| Field | Type | Notes |
|---|---|---|
| `name` | string | Standardized nomenclature, the team-wide identifier |
| `slug` | slug | Deep-link key |
| `brand` | reference → `brand` | Drives the active theme |
| `category` | string (list) | Logos, Lower Thirds, Stings, Backplates, Fonts, LUTs, Templates |
| `fileType` | string (list) | `mogrt`, `aep`, `mov`, `png`, `svg`, `c4d`, `lut`, `otf` |
| `description` | text | What it is |
| `usageContext` | text | When to use it, and when not to |
| `tags` | array of string | Free-form search terms |
| `previewStill` | image | Required. Card and drawer poster |
| `previewLoop` | file (mp4/webm) | Optional. Plays on hover for motion assets |
| `aspectRatio` | string (list) | `16:9`, `9:16`, `1:1`, `4:5` — drives card sizing |
| `share` | reference → `share` | Mount roots for path derivation |
| `relPath` | string | POSIX-style path relative to the share root |
| `status` | string (list) | `approved`, `wip`, `deprecated` |
| `version` | string | e.g. `v2.1` |
| `supersededBy` | reference → `asset` | Set when `status` is `deprecated` |
| `owner` | string | Who to ask about this asset |
| `updatedAt` | datetime | Surfaced on the card and in "Recently updated" |

Lifecycle fields are the reason a brand hub exists rather than a shared folder: a deprecated
asset stays visible but is visually marked and links directly to its replacement, so nobody
ships last season's logo.

---

## 5. Theme engine

### Token contract

Every brand defines the same locked set of CSS custom properties. Nothing in the UI may
reference a brand color outside this set:

```
--bg            page background
--surface       card and drawer background
--surface-2     hover and elevated state
--border        hairlines
--text          primary foreground, must hit 4.5:1 on --bg and --surface
--text-muted    secondary foreground, must hit 4.5:1 on --surface
--accent        brand color for chips, focus rings, active states
--accent-fg     foreground guaranteed legible ON --accent
```

Pairing `--accent` with `--accent-fg` is what prevents the accessibility trap in dynamic
theming: a brand with a bright yellow accent supplies dark `--accent-fg`, and accent is never
used as text directly on `--bg`. A dev-mode contrast assertion logs any brand whose tokens
fail 4.5:1 so palette problems surface at authoring time, not in production.

### Mechanics
- Tokens are applied to a wrapper element as inline custom properties by a `ThemeProvider`
  reading the active brand from React context. No Zustand needed at this scale.
- Switching brands crossfades tokens over ~200ms via `transition` on color properties.
  Instant palette flips read as a rendering bug.
- The theme also respects `prefers-reduced-motion`, disabling the crossfade and hover loops.

---

## 6. Architecture

```
src/
  main.tsx
  App.tsx
  routes/
    HubRoute.tsx          grid + filters + drawer
    StudioRoute.tsx       embedded Sanity Studio at /studio
  theme/
    ThemeProvider.tsx     brand context, token injection
    contrast.ts           dev-only WCAG assertions
  data/
    sanity.ts             client config
    queries.ts            GROQ
    useAssets.ts          fetch, cache, error and loading states
    types.ts              generated/hand-kept TS types
  lib/
    derivePaths.ts        canonical path → win/mac/url variants
    clipboard.ts          async API + execCommand fallback
    search.ts             client-side match and rank
  components/
    AssetGrid.tsx
    AssetCard.tsx         preview, name, status badge, copy split-button
    CopyPathButton.tsx    primary copy + variant dropdown
    AssetDrawer.tsx       large preview, metadata, all variants
    BrandSwitcher.tsx
    FilterBar.tsx         search input, category/type/status chips
    StatusBadge.tsx
    EmptyState.tsx / ErrorState.tsx / GridSkeleton.tsx
sanity/
  schemaTypes/            brand.ts, share.ts, asset.ts
  structure.ts            Studio desk organized by brand
```

---

## 7. Interaction and states

- **Grid** — masonry-ish responsive columns honoring each asset's `aspectRatio`; cards are
  image-dominant with metadata as a gradient overlay that solidifies on hover.
- **Hover** — `previewLoop` plays muted and looping; leaves back to `previewStill`. Suppressed
  under `prefers-reduced-motion` and on touch devices.
- **Copy** — split button. Primary click copies the Windows path and shows an inline
  "Copied" confirmation on the button itself, not a corner toast. Dropdown exposes Mac,
  POSIX/URL, and filename-only.
- **Detail** — drawer, not a modal, so the grid stays visible. Opening pushes
  `?asset=<slug>` so the URL is shareable; closing pops it. A direct load of that URL opens
  the drawer on the right asset.
- **Deprecated assets** — desaturated preview, amber `Deprecated` badge, and a
  "Use <name> instead →" link that navigates to the superseding asset.
- **Loading** — skeleton cards matching the real grid geometry, no layout shift on arrival.
- **Empty** — distinguishes "no assets in this brand yet" from "no results for this search",
  with a clear-filters action for the latter.
- **Error** — explicit failed-to-load state with a retry, never an indefinite spinner.
- **Keyboard** — `/` focuses search, arrow keys move through the grid, `Enter` opens the
  drawer, `Esc` closes, `Cmd/Ctrl+C` on a focused card copies its path. Focus rings use
  `--accent` and are never suppressed.

---

## 8. Search and filtering

At 50–200 assets everything is fetched once and filtered in memory — instant, no request
per keystroke. Matching runs across `name`, `description`, `usageContext`, `tags`, and the
filename portion of `relPath`, with results ranked name-match first. Filters for brand,
category, file type, and status compose with the query and are reflected in the URL so a
filtered view is shareable.

---

## 9. Delivery phases

1. **Scaffold** — Vite + React + TS + Tailwind, token plumbing, static build verified.
2. **Sanity** — create project and dataset, deploy the three schema types, mount `/studio`,
   seed 12–15 representative assets across brands and categories.
3. **Core hub** — grid, cards, `derivePaths`, copy button with fallback, search and filters.
4. **Theming** — brand switcher, crossfade, contrast assertions across all supplied palettes.
5. **Detail and polish** — drawer, deep links, lifecycle badges, empty/loading/error states,
   keyboard nav.
6. **Deploy** — Vercel with deployment protection, add the resulting domain to Sanity CORS.

Phases 1 and 3 are the critical path; theming can proceed in parallel once palettes arrive.

---

## 10. Testing

- Unit tests on `derivePaths` covering separator conversion, trailing-separator tolerance,
  nested directories, spaces and non-ASCII characters in filenames, and URL-encoding of the
  HTTP variant.
- Unit tests on the search ranking function.
- A contrast test asserting every brand's token set meets 4.5:1 for `--text` on `--bg` and
  `--surface`, and `--accent-fg` on `--accent`.

---

## 11. Security notes

The hub publishes internal server topology — share names, directory structure, hostnames.
That is inherent to its purpose, but it means the deployment must not be publicly reachable.
Vercel deployment protection is therefore a v1 requirement, not a nicety. The Sanity dataset
holds no secrets, and the read token is public-scoped; write access stays with authenticated
Studio users.

---

## 12. Open items

1. **Brand names and hex codes.** Blocking for phase 4. For each sub-brand: display name and
   the palette values behind the eight tokens in §5, or the brand guideline document.
2. **Real share roots.** The UNC and macOS mount roots for the asset library. Placeholder
   values are fine to build against and change in one document later.
3. **Category list confirmation.** The seven categories in §4 are a proposal based on a motion
   design workflow; confirm or replace.
4. **Studio editor accounts.** Which teammates need Sanity Studio logins to add assets.
