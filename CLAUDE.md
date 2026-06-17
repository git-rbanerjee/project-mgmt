# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page resource & project planning app. The entire app — HTML, CSS, JS — lives in `index.html`. There is no build step, no package manager, no test suite.

- Open it: `open /Users/rabin_banerjee/Projects/project-mgmt/index.html` (or double-click).
- Reload the browser tab to see edits.

## Architecture

Everything is in one file. The mental model is straightforward; the non-obvious part is the availability model and the way the four data stores interact.

### State shape (persisted to `localStorage` under `rpPlannerData_v1`)

```
state = {
  resources:   [{id, name, color, active}],
  projects:    [{id, name, color, start, end}],
  allocations: { [resId]: { [date]: { [projectId]: fraction } } },  // see "Fractional allocations"
  availability:{ [resId]: { [date]: true } },   // see "Availability model"
  holidays:    [{date, name}],                  // global, applies to everyone
  view:        { startDate, dayCount, mode, showProjectDetails }
}
```

`save()` writes the whole `state` to localStorage on every mutation. `load()` runs once on startup and **migrates older payloads**: it drops `state.unavailability`, backfills any allocation as available, converts string-shaped allocations (`allocations[res][date] = "projId"`) to fractional maps (`{projId: 1}`), and defaults missing `resource.active` to `true`. When changing the schema, extend the migration block in `load()`.

### Active vs inactive resources

`getActiveResources()` returns `state.resources.filter(r => r.active !== false)`. The grid renderers, daily/weekly capacity, and "N of M resources available" tooltip all go through this helper — never iterate `state.resources` directly in those paths. The sidebar shows all resources but groups inactive ones under an "Inactive (N)" header, dimmed and strikethrough. Inactive resources keep their allocations (Project view still sees them).

### Fractional allocations

A cell's value is a `{projectId: fraction}` map. Fractions are unrestricted decimals (typically 0–1, but values > 1 are allowed and rendered with an `.overallocated` outline). Use the helpers (`getCellAllocations`, `getCellTotal`, `setCellAllocations`, `clearCellAllocations`, `dayUsedTotal`, `projectFractionOn`) instead of touching the map directly — they handle the empty-map cleanup and avoid littering the store with `{}` entries.

The single-cell and bulk modals both go through `openBulkModal`, which renders one input per project with quick-pick buttons (0 / ¼ / ½ / ¾ / 1). Bulk save **replaces** each selected cell's allocation map entirely with the modal values.

### Availability model — the non-obvious part

Every resource is "shared by default." A resource-day is **not yours** unless explicitly marked. The three cell states in Resource view are:

| State                    | How to set                | Counts toward capacity? |
|--------------------------|---------------------------|-------------------------|
| Not available (hatched)  | default; "Not available" in modal | no              |
| Available, unallocated   | "Available" in modal      | yes                     |
| Allocated to project     | pick a project in modal   | yes                     |

Allocating **requires** availability — `openBulkModal` refuses to assign a project to a resource-day that isn't marked available, and reports the count of skipped cells via `alert`. Clearing a project does not clear availability; that's a separate action. Preserve this invariant in any new mutation paths.

### Project date range — explicit vs derived

A project's effective date range is computed by `getProjectDateRange(projectId)`:

- If both `start` and `end` are set on the project, those are returned and treated as **hard constraints** — `isDateInProjectRange()` returns false outside them, and `openBulkModal` skips allocation attempts outside the window. The single-cell modal disables out-of-range project options inline.
- If either is missing, the range is **derived** from min/max allocation dates and surfaced with a "(derived)" suffix in the Project view label. No constraint is applied in this case (you can allocate anywhere).

The Project view's in-range shading uses the *effective* range (explicit or derived), so newly-allocated days will expand the visible timeline automatically.

### Render pipeline

`render()` is the single entry point that rebuilds the DOM after any state change. It calls `renderResources` / `renderProjects` / `renderHolidays` (sidebar) and `renderGrid` (which dispatches to `renderResourceGrid` or `renderProjectGrid` based on `state.view.mode`). Both grid renderers build an HTML string and assign to `innerHTML`, then re-attach handlers (`attachCellHandlers` for Resource view). Don't attempt partial DOM updates — the renderers assume a full rebuild.

### Cell interaction

- **Plain click** on a cell → `openAllocationModal(resId, date)` (which calls `openBulkModal` with single-element arrays).
- **Click & drag** along a single row → `openBulkModal([resId], dates)` for that date range. Dragging is constrained to the row where mousedown started; entering cells in other rows is ignored so the selection never goes multi-row. Determining click-vs-drag is done at mouseup: if `dragStart.date === dragCurrent.date`, treat as a plain click.

The drag state lives in the module-level `dragState` object so it survives re-renders. The `mouseup` listener is registered exactly once on module load — do not register it inside `attachCellHandlers` (a previous bug used `{ once: true }` there and broke the second drag).

### Utilization math

- **Daily**: `used / available` where `available = availableResourcesOn(date)`.
- **Weekly** (`weeklyUtilCell`): capacity = sum over visible days of resources where `isAvailable(r.id, d) && !weekend && !holiday`. Week grouping (`groupByWeek`) uses Monday as the week start.

Holidays are global and reduce everyone's capacity. Availability is per-resource. Both must be respected when changing the utilization formula.

### Color palettes

`RESOURCE_PALETTE` / `PROJECT_PALETTE` deliberately exclude red / yellow / green — the visual language is neutral slates and cool blues. The utilization color scale is also a slate gradient (`#cbd5e1` → `#1e293b`), where darker = busier. Don't reintroduce traffic-light colors without checking with the user.

### Layout

The page is a strict flex column: header (auto height) + `main` (flex: 1). `main` is `display: grid` with `grid-template-rows: minmax(0, 1fr)` — the `0` min-track is what keeps the grid container from expanding to fit its table; remove it and the page will scroll. The `.grid-container` scrolls internally for both axes.
