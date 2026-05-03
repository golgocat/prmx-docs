# PRMX UI Design Principles

> Scope: Terminal workspace, landing pages, and all product surfaces.

## Design Intent

The PRMX UI has two distinct design surfaces:

1. **Terminal Workspace** — a dark, grid-backed console for climate risk operations. Professional tone, high density, inspectable data.
2. **Landing Pages** — cinematic marketing surfaces with gradient cards, parallax, and branded product lines.

Both share the same font stack and color token system but diverge in tone and layout.

## Non-Negotiables

1. **Terminal workspace** routes use the `.terminal-workspace` class (applied in the dashboard layout). Never apply landing-page gradients or marketing visuals inside it.
2. No logo art and no emoji icons in workspace UI. Lucide icons only.
3. Data-first composition (tables/rows before card mosaics).
4. Traceability on critical workflows:
   - Show endpoint/context JSON and replay `curl` for pricing and policy views.
5. Visual indicators that improve comprehension stay in place:
   - Coverage/threshold progress bars, status badges, timeline states.
6. Role-based navigation: DAO-only pages (Vault, DAO Underwrite, Oracle, Agents, Control Plane) are hidden from customer/LP roles.

## Visual System

### Color tokens

All colors are defined as CSS custom properties (RGB triplets for Tailwind `<alpha-value>` support):

| Token | Dark (default) | Terminal Workspace | Light |
|-------|---------------|-------------------|-------|
| `--bg-primary` | `10 14 23` | `2 6 12` | `248 250 255` |
| `--bg-secondary` | `17 24 39` | `6 10 18` | `241 248 255` |
| `--bg-tertiary` | `31 41 55` | `12 18 28` | `224 242 254` |
| `--bg-card` | `21 28 44` | `8 12 20` | `255 255 255` |
| `--text-primary` | `255 255 255` | `231 236 246` | `15 23 42` |
| `--text-secondary` | `156 163 175` | `143 155 178` | `71 85 105` |
| `--text-tertiary` | `107 114 128` | `98 111 138` | `148 163 184` |
| `--border-primary` | `55 65 81` | `34 45 64` | `186 230 253` |
| `--border-secondary` | `31 41 55` | `24 34 52` | `224 242 254` |

### Accent color remapping

The token `prmx-cyan` (`#00E5FF`) is the global accent, but remaps contextually:

- **Terminal workspace**: `prmx-cyan` remaps to **emerald-400** (`#34D399`) via `.terminal-workspace` CSS overrides. This includes all backgrounds, borders, text, ring, hover, and focus states.
- **Light mode**: `prmx-cyan` remaps to **sky-500** (`#0ea5e9`) via `html.light` CSS overrides.
- **Landing pages**: Use the raw `prmx-cyan` (#00E5FF) plus `brand-*` tokens (violet, teal, amber, magenta).

### Semantic status colors

| Semantic | Tailwind class | Value |
|----------|---------------|-------|
| Success | `success` | `#10B981` (emerald-500) |
| Warning | `warning` | `#F59E0B` (amber-500) |
| Error | `error` | `#EF4444` (red-500) |
| Info | `info` | `#3B82F6` (blue-500) |
| Purple | `prmx-purple` | `#9C27B0` |

Badge variants map to these: `badge-success`, `badge-warning`, `badge-error`, `badge-info`, `badge-cyan`, `badge-purple`.

### Typography

Three font families are loaded via `next/font/google` (no FOUT due to `display: swap`):

| Font | CSS Variable | Use |
|------|-------------|-----|
| Inter | `--font-inter` | Body text, UI labels |
| Plus Jakarta Sans | `--font-plus-jakarta` | Display headings on landing pages |
| JetBrains Mono | `--font-jetbrains-mono` | Code, monospace data, addresses, timestamps, block numbers, policy IDs |

Hierarchy within workspace:

- Page title: `text-2xl` to `text-3xl`, `font-semibold` or `font-bold`.
- Section headers: `text-base`, `font-semibold`.
- Workspace label (above title): `text-[10px] uppercase tracking-[0.2em] text-text-tertiary`.
- Metadata labels: `text-[10px]` to `text-xs`, `uppercase tracking-wider text-text-tertiary`.
- Stat values: `text-xl font-semibold text-text-primary`.
- Table headers: `text-[10px] uppercase tracking-wider text-text-tertiary font-semibold`.

### Layout density

- Favour single-page operational flow with section stacking.
- Prefer row/table structures for policy, request, and DAO records.
- Keep spacing tight but readable (`gap-2` to `gap-4`, panel padding `p-4` to `p-6`).
- Workspace max width: `max-w-6xl mx-auto` (applied in dashboard layout).
- Cards use `glass-card` base with `rounded-2xl` (terminal workspace overrides to `0.875rem`).

## Component Patterns

### Glass card (`glass-card`)

Base UI surface across all workspace pages.

- `backdrop-blur-xl` with semi-transparent `--bg-card` background.
- Subtle `--border-primary` border with `rounded-2xl`.
- Terminal workspace overrides: deeper dark gradients, stronger box-shadow, emerald hover glow.
- Light mode: `shadow-lg shadow-sky-100/50`.
- `glass-card-hover` variant adds emerald/cyan border glow on hover.

### Card composition

All cards use `<Card>`, `<CardHeader>`, `<CardContent>`, `<CardFooter>`:

```tsx
<Card>
  <CardHeader>
    <div className="flex items-center gap-2">
      <Icon className="w-5 h-5 text-prmx-cyan" />
      <h2 className="text-base font-semibold text-text-primary">Title</h2>
      <Badge variant="success" dot>Running</Badge>
    </div>
  </CardHeader>
  <CardContent>
    {/* DataRows or table */}
  </CardContent>
</Card>
```

### DataRow pattern

Used in operational pages (Vault, Agents, CCTP, Oracle):

```tsx
<div className="flex items-baseline justify-between py-2 border-b border-border-secondary">
  <span className="text-xs uppercase tracking-wider text-text-tertiary">Label</span>
  <span className="text-sm font-medium text-text-primary">Value</span>
</div>
```

### Grid stats

Summary metrics at the top of operational pages:

```tsx
<div className="grid grid-cols-2 sm:grid-cols-4 gap-6">
  <div>
    <p className="text-[10px] uppercase tracking-widest text-text-tertiary mb-1">Label</p>
    <p className="text-xl font-semibold text-text-primary">42</p>
  </div>
</div>
```

### Status indicators

- **Dot badge**: `<Badge variant="success" dot>Running</Badge>` — small coloured circle before text.
- **Inline dot**: `<span className="w-2 h-2 rounded-full bg-success" />` next to text labels.
- **Pulse dot**: `animate-pulse` on connection indicators (chain status in header).

### Loading state

All data-fetching pages use skeleton loading with `animate-pulse`:

```tsx
<div className="space-y-6 animate-pulse">
  <div className="glass-card p-6">
    <div className="h-4 w-40 bg-background-tertiary rounded mb-4" />
    <div className="h-3 w-full bg-background-tertiary rounded" />
  </div>
</div>
```

### Error state

Consistent error banner with icon:

```tsx
<div className="glass-card border-error/30 p-4 flex items-start gap-3">
  <AlertCircle className="w-5 h-5 text-error" />
  <div>
    <p className="text-sm font-medium text-error">Failed to load</p>
    <p className="text-xs text-text-tertiary mt-1">{message}</p>
  </div>
</div>
```

### Progress bars

- Coverage and threshold progress are linear and quantitative.
- Show percentage/value labels next to bars.
- Normal: emerald/cyan, approaching: amber, triggered: red.

### Raw inspection panels

Critical views include:

- `Raw JSON` panel with expand/copy/download.
- `curl` replay panel for endpoint reproducibility.

These are part of the UI contract for pricing and policy detail workflows.

### Tables

- Header: `text-[10px] uppercase tracking-wider text-text-tertiary` with `border-b`.
- Rows: `border-b border-border-secondary hover:bg-background-tertiary/40 transition-colors`.
- Monospace for addresses and IDs: `font-mono text-text-secondary`.
- Responsive: hide less important columns with `hidden md:table-cell` / `hidden lg:table-cell`.

### Buttons

Three tiers:

| Class | Use | Terminal Style |
|-------|-----|---------------|
| `btn-primary` | Main CTA | Emerald gradient with dark text |
| `btn-secondary` | Secondary actions | Dark bg, subtle border |
| `btn-ghost` | Tertiary/inline | Transparent, hover bg |

All include `active:scale-95` and `transition-all duration-300`.

### Form inputs

- `input-field` class: dark bg, border, cyan/emerald focus ring.
- `select-field` extends `input-field` with custom chevron.
- DateTime pickers respect `color-scheme: dark/light`.

## Navigation and IA

### Sidebar structure

```
PRMX TERMINAL
Operations Workspace
─────────────────────
Main Menu
  Dashboard
  Protection Terminal    ← unified purchase flow (all 14 products)
  Policies               ← role label: "My Policies" or "Ecosystem Policies"
  LP Trading             [dao, lp]
  Equity
  Vault                  [dao]
  DAO Underwrite         [dao]
  Oracle                 [dao]
  AI Agents              [dao]
  Control Plane          [dao]
─────────────────────
Support
  Help & Resources
  Settings
─────────────────────
[Climate Peril Map CTA]
```

Items tagged with `[role]` are hidden from other roles. Role is determined by `useUserRole()`.

### Header

Fixed header with:

- Mobile hamburger menu (md:hidden)
- UTC clock (full date+time on desktop, time-only on mobile)
- Chain status: green dot + block number link (opens Polkadot.js explorer)
- Oracle auth status: key icon + dot indicators with hover tooltip
- Account selector

### Breadcrumbs

Nested routes (e.g., `/policies/[id]`, `/markets/[id]`) show breadcrumbs below the header.

### Product line switching

Three product lines are exposed in the purchase terminal:

- `PRMX Rain Guard` — core rainfall
- `PRMX Weather Gate` — temperature, wind, pressure, precipitation intensity
- `PRMX Climate Parametrics` — specialty products (PM2.5, river discharge, wave height, etc.)

The unified `/climate-parametrics` page lets users browse all 14 products via dropdown groups.

## Landing Pages

Landing pages (`/`, `/products`, `/products/[line]`, `/provide-liquidity`, `/start`, `/manila`) use a separate design language:

- Background: `from-zinc-950 via-zinc-950 to-black`.
- Cards: `border-white/10 bg-white/[0.03]` with hover lift (`hover:-translate-y-0.5 hover:border-white/25`).
- Icons in bordered boxes: `rounded-xl border border-white/15 bg-white/5`.
- Badges: `border border-white/15 bg-white/5 text-white/70 uppercase tracking-[0.2em]`.
- Gradient text: `bg-prmx-gradient-horizontal bg-clip-text text-transparent`.
- Section heroes with animated visual backgrounds (`AnimatedHeroVisual`, `ParallaxHero`).

Landing page components live in `components/landing/` and `components/sections/`.

## Light Mode

Full light mode is supported via `html.light` class (toggled by theme store):

- Background shifts to icy blue/snow palette (`248 250 255` → `224 242 254`).
- Accent remaps: `prmx-cyan` → `sky-500` (#0ea5e9).
- Borders use light sky tones (`186 230 253`).
- Cards get `shadow-lg shadow-sky-100/50`.
- Theme resolved before first paint via inline `<script>` in root layout (avoids FOUWT).
- The theme store (`themeStore.ts`) exposes `preference` (light/dark/system) and resolved `theme`.
- localStorage key: `prmx-theme`.

**Light mode does NOT apply to the terminal workspace**. The `.terminal-workspace` class overrides all token values regardless of theme.

## Workspace-Specific Guidance

### Dashboard

- Hero panel with wallet info, balance, and quick actions.
- Overview stats grid (policies, active coverage, capital routes).
- Recent policies table.
- Capital control plane status cards (for DAO role).

### Protection Terminal (`/climate-parametrics`)

- Product group dropdown → city selector → threshold input → coverage dates.
- Real-time pricing API calls with `RawJsonPanel` and `CurlCommandPanel`.
- Purchase confirmation modal with summary.

### Policies

- Tabbed view by status (Active, Monitoring, Settled, All).
- Search, sort, and filter persisted in URL (`useSearchParams`).
- Policy detail pages: progress bars, timeline, raw inspection panels.

### Vault dashboard (`/vault`)

- System overview grid (Vault Reporter status, CCTP Relay status, counts).
- Discovered vaults table with explorer links.
- Auto-refresh every 30 seconds with polling indicator.

### AI Agents (`/agents`)

- Portfolio valuation metrics.
- Per-agent cards with balance, exposure, activity feed.
- Expandable detail sections.

### Equity (`/equity`)

- Tabs: Buy, Vest/Unlock, Claim Dividends.
- Transaction state machine (idle → signing → submitted → success/error).
- Token stats grid.

### Markets (`/markets`)

- Searchable city browser by region.
- Cards per city with product availability badges.

### DAO workspace (`/dao`)

- Service config and record streams in dense operational tables.
- Exception handling and status counts.

### LP workspace (`/lp`)

- Positions, exposure, and time-to-expiry readability.
- Order actions adjacent to numerical risk context.

## Accessibility and Interaction Quality

- All interactive cards use `role="button"`, `tabIndex={0}`, and `onKeyDown` (Enter/Space).
- Focus states use `focus:ring-2 focus:ring-prmx-cyan/20` (remapped per context).
- Colour is never the only signal (text labels + icon + bar values).
- Mobile retains same information model with stacked layout (not feature loss).
- Sidebar has a mobile drawer with backdrop blur overlay and close button.
- Tables hide less important columns responsively rather than truncating.

## Icon System

All icons come from **Lucide** (`lucide-react`). Common workspace icons:

| Context | Icons |
|---------|-------|
| Navigation | `LayoutDashboard`, `Shield`, `FileText`, `Wallet`, `Coins`, `Vault`, `Database`, `Activity`, `Bot`, `Monitor` |
| Status | `Wifi`/`WifiOff`, `AlertCircle`, `CheckCircle2`, `Loader2` |
| Actions | `RefreshCw`, `Copy`, `ExternalLink`, `Plus`, `Search`, `ChevronDown`/`Up` |
| Data | `Clock`, `Key`, `Globe2`, `MapPin`, `Link2` |

No custom SVG icons or emoji are used in the workspace.

## Definition of Done

A workspace page is complete when:

1. Uses `.terminal-workspace` inherited styles (no raw background overrides).
2. No emoji or logo art appears in operational workspace screens.
3. Loading skeleton and error states are handled.
4. Policy detail provides progress + timeline + raw data access.
5. Pricing workflows expose replayable endpoint context.
6. Role-based elements are gated via `useUserRole()`.
7. Light mode does not break layout (terminal workspace is dark-only, but shared components must handle both).
8. Lint/build pass with no regressions.
