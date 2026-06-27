# Quest HQ — Customizable Command Center Dashboard (Developer Handoff)

A working, self-contained prototype of the customizable dashboard: role views, rep + date filters, an add-widget tray, drag-to-rearrange, and ~30 widgets across Sales, Operations, Value drivers, Growth, Capacity, People, Cash flow, Strategy, EOS, Finance, and Reputation.

**This prototype runs on sample data.** Your job is to (1) port the widget framework into the app and (2) wire each widget's `render()` to real data. See `WIDGET-DATA-SPEC.md` for the exact source of every widget.

## Files
- `Quest-HQ-Custom-Dashboard.html` — the full interactive prototype (open in any browser).
- `WIDGET-DATA-SPEC.md` — per-widget data mapping (table / fields / query + build status).
- `dashboard-exec.png`, `dashboard-scale.png`, `dashboard-eos.png` — reference screenshots.

## Architecture (matches our vanilla-JS / Vite style — no framework, no chart lib)
The whole thing is one registry + a render loop. Three concepts:

1. **`WIDGETS`** — an object map of `id → { title, group, span, sub, render() }`. `render()` returns an HTML string. All charts are pure CSS/SVG (donut = `conic-gradient`, bars = `<i>` with `width:%`), so there's nothing to install.
2. **`GROUPS`** — ordered category names for the Add-widget tray.
3. **`DEFAULTS` / `layout`** — which widget ids show (in order) per role view: `exec`, `sales`, `ops`, `scale`, `eos`.

State is just `ROLE`, `RANGE`, `REP`, `EDIT`, and the `layout` object. The render loop writes the active widgets for the current role into `#grid`. Filters call helpers (`pool()`, `scaled()`) so numbers recompute live.

## Porting into the app
1. **Add a Dashboard module/route** (or replace the Home page) that renders the shell: role tabs, filter bar, `#tray`, `#grid`.
2. **Move the `WIDGETS` registry into the app.** Keep the same shape. Replace each widget's sample-data references with calls to our real data helpers (`companyDeals`, `financeSummary`, `companyJobs`, `companyTasks`, etc. — see spec).
3. **Wire the filters** (`REP`, `RANGE`) to actually filter the queries instead of scaling sample numbers.
4. **Persist layouts.** Save `layout` per user (and/or per role) so customizations stick. Suggested Supabase table below.
5. **Permissions.** Gate widgets by role/permission the same way the rest of the app does (`can(...)`), so a rep doesn't see EBITDA, etc.

## Suggested persistence (Supabase)
```sql
create table dashboard_layouts (
  id uuid primary key default gen_random_uuid(),
  company_id text not null,
  user_id uuid not null references auth.users(id),
  role_view text not null,              -- 'exec' | 'sales' | 'ops' | 'scale' | 'eos' | custom
  widget_ids jsonb not null default '[]',
  updated_at timestamptz default now(),
  unique (company_id, user_id, role_view)
);
-- RLS: user can read/write only their own rows within their company (mirror existing policies)
```
On load: fetch the row for `(company, user, role)`; fall back to `DEFAULTS[role]` if none. On reorder/add/remove: upsert `widget_ids`.

## Customization behaviors already implemented (replicate these)
- **Role tabs** switch layouts.
- **Rep filter + date range** recompute widget numbers.
- **＋ Add widget** opens a tray grouped by `GROUPS`; clicking adds the id to the current layout.
- **✎ Customize** toggles edit mode → shows a drag handle + ✕ remove on each card; HTML5 drag reorders the layout array.

## Notes
- The prototype uses in-memory state (no localStorage) on purpose — swap in the Supabase persistence above.
- Widgets marked **span2** take the full row; keep that flag.
- Keep the group taxonomy — it's how owners find widgets in the tray.
- Charts are intentionally dependency-free. If you'd rather use a chart library later, the data shapes are already separated from the markup, so it's a clean swap.
