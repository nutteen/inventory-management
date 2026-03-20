---
name: vue-analyzer
description: Analyzes Vue 3 component structure and suggests targeted optimizations for performance and code reuse. Use this skill when asked to review, audit, or optimize Vue components.
---

# Vue Component Analyzer

This skill provides a systematic framework for analyzing Vue 3 components and producing actionable, prioritized optimization suggestions. Follow the analysis steps in order, then produce a structured report.

## How to Invoke This Skill

When the user runs `/vue-analyzer` (optionally with a path or component name), perform the full analysis below on the specified target. If no target is given, analyze all `.vue` files under `client/src/`.

```
/vue-analyzer                        # Analyze all components
/vue-analyzer Dashboard.vue          # Analyze one file
/vue-analyzer client/src/views/      # Analyze a directory
```

---

## Step 1 — Inventory the Files

Start by mapping the codebase:

```bash
# Line counts for all components
wc -l client/src/**/*.vue client/src/components/**/*.vue

# List composables
ls client/src/composables/
```

Flag any file **over 300 lines** as a candidate for decomposition. Files over 600 lines are high-priority.

---

## Step 2 — Structural Analysis Checklist

For each `.vue` file, check every item below. Record findings by file and category.

### 2a. Template Complexity

| Signal | What to look for |
|---|---|
| Deep nesting | `<div>` trees more than 4–5 levels deep |
| Repeated markup | Identical block structures (e.g., multiple KPI cards, table rows with same layout) |
| Inline expressions | Logic in `{{ }}` that could move to computed (e.g., `{{ (a / b * 100).toFixed(1) }}`) |
| Missing keys | `v-for` with `:key="index"` instead of a stable unique ID |
| Wrong toggle | `v-if` used for content toggled frequently (use `v-show` instead) |
| Wrong toggle | `v-show` used for content that rarely appears (use `v-if` instead) |

**Example — Repeated KPI card markup in Dashboard.vue:**
```html
<!-- Repeated 5× — extract to <KpiCard> component -->
<div class="kpi-card">
  <div class="kpi-header"><span class="kpi-label">...</span></div>
  <div class="kpi-value">...</div>
  <div class="kpi-goal">...</div>
  <div class="kpi-progress-bar"><div class="kpi-progress"></div></div>
</div>
```

### 2b. Script / Setup Complexity

| Signal | What to look for |
|---|---|
| Computed vs method misuse | Values derived from reactive state that are defined as methods (called every render instead of cached) |
| Missing computed | Inline template expressions doing filtering, formatting, or math |
| Duplicated data-loading | Multiple views with the same `loading / error / try-catch / finally` pattern |
| Watch overuse | `watch` used to derive values that could be `computed` |
| Direct prop mutation | `props.x = y` instead of `emit('update:x', y)` |
| Missing date guard | `new Date(str).getMonth()` without `isNaN` check |
| Unextracted composable logic | API calls, filter logic, or auth logic repeated across views |

**Example — inline template math that belongs in computed:**
```js
// In template: {{ ((summary.value / revenueGoal - 1) * 100).toFixed(1) }}%
// Should be:
const revenueVariancePct = computed(() =>
  ((summary.value.total_orders_value / revenueGoal.value - 1) * 100).toFixed(1)
)
```

### 2c. Component Extraction Candidates

Extract a block into a new component when:
- The block is **used 2+ times** in one template
- The block is **used in 2+ different views**
- The block has its own clear responsibility and is **>30 lines** of template
- The block manages its own local state

**Extraction checklist:**
1. Identify props the block reads from the parent scope
2. Identify events the block needs to emit upward
3. Create `client/src/components/<Name>.vue`
4. Pass data via props; communicate changes via emits
5. Delegate the new file to `vue-expert` agent

### 2d. Composable Extraction Candidates

Extract logic into a composable (`client/src/composables/use<Name>.js`) when:
- The same `ref + computed + watch + API call` pattern appears in 2+ views
- State needs to be **shared as a singleton** across views (like `useFilters`)
- Logic is testable in isolation

**Common patterns found in this project:**
```js
// Every view repeats this — candidate for useDataLoader(fetchFn)
const loading = ref(true)
const error = ref(null)
const data = ref([])
const loadData = async () => {
  try {
    loading.value = true
    error.value = null
    data.value = await api.getSomething(filters)
  } catch (e) {
    error.value = 'Failed to load data'
  } finally {
    loading.value = false
  }
}
```

### 2e. Performance Signals

| Signal | Fix |
|---|---|
| `methods` computing derived values | Move to `computed` |
| Heavy filter/sort inside `v-for` expression | Move to `computed` |
| Component not lazy-loaded but only shown conditionally | Use `defineAsyncComponent` |
| Large list without virtualization | Suggest windowing if >200 rows |
| Event listeners not cleaned up | Add `onUnmounted` cleanup |
| Watchers triggering API calls without debounce | Use `watchDebounced` |

### 2f. Code Style & Maintainability

| Signal | What to look for |
|---|---|
| Magic numbers | Hardcoded values like `0.9333`, `800000` with no named constant |
| Hardcoded strings in template | User-visible text not wrapped in `t()` i18n calls |
| Style duplication | Scoped styles repeated across multiple components |
| Missing prop validation | Props without `type`, `required`, or `default` |
| Missing error boundary | No `v-else-if="error"` state for async views |

---

## Step 3 — Produce the Analysis Report

After completing the checklist, output a structured report in this format:

```
## Vue Component Analysis Report

### Summary
- Files analyzed: N
- Total lines: N
- High-priority issues: N  (components > 600 lines or critical bugs)
- Medium-priority issues: N
- Low-priority issues: N

---

### [COMPONENT NAME] — path/to/Component.vue (N lines)

**Decomposition** [HIGH/MED/LOW]
- <specific block> is repeated N times → extract as <SuggestedName>.vue
  Props: label, value, goal, progress
  Emits: (none)

**Performance** [HIGH/MED/LOW]
- `calculateX()` method called in template → move to computed `xValue`
- `v-if` on `.modal` toggled on click → change to `v-show`

**Code Reuse** [HIGH/MED/LOW]
- Loading/error pattern duplicated with Inventory.vue and Orders.vue
  → extract `useDataLoader(fetchFn, filters)` composable

**Maintainability** [LOW]
- Revenue goal hardcoded as `800000` → move to named constant `MONTHLY_REVENUE_GOAL`
- 3 user-visible strings not wrapped in `t()` calls

---
```

Prioritize by impact:
- **HIGH**: Causes bugs, performance degradation, or the component is unmaintainable (>600 lines)
- **MED**: Duplicated logic across 2+ files, or clear perf win available
- **LOW**: Style, naming, minor refactor opportunity

---

## Step 4 — Offer to Implement

After the report, ask:

> Would you like me to implement any of these suggestions? I can:
> - Extract component(s) via the `vue-expert` agent
> - Create a composable in `client/src/composables/`
> - Convert methods to computed properties
> - All of the above for a specific component

Wait for user confirmation before making any changes.

---

## Project-Specific Patterns to Always Check

These are patterns specific to this inventory management codebase:

### KPI Card Repetition (Dashboard.vue)
The dashboard has 5+ repeated KPI card blocks. Recommend extracting a `KpiCard.vue` component:
```
Props: label (String), value (String|Number), goal (String), progress (Number 0-100), variant ('default'|'success')
```

### useDataLoader Opportunity
`loading / error / data / loadData` is repeated in every view. A `useDataLoader(fetchFn)` composable returning `{ loading, error, data, reload }` would eliminate ~15 lines per view.

### Filter Watch Pattern
All views watch `filterParams` and call `loadData()`. This watch logic is identical and could live in a shared composable.

### Inline Currency Formatting
Multiple views call `.toFixed(2)`, `.toLocaleString()`, and `currencySymbol` inline. A `formatCurrency(value, currency)` utility (already in some views) should be used consistently everywhere.

### i18n Coverage
Check all user-visible strings for `t()` wrapping. New views sometimes hard-code English strings.

---

## Delegation Rules

When implementing suggestions:
- **Always delegate `.vue` file creation/modification to `vue-expert` agent** (per CLAUDE.md)
- Composables (`.js` files) can be written directly
- Use `code-reviewer` agent after implementing significant changes
