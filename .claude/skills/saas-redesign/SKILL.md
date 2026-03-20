---
name: saas-redesign
description: Redesign a Vue 3 app's UI into a modern SaaS-style interface with a vertical navigation sidebar replacing a top nav bar
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Agent
argument-hint: [optional: path to App.vue]
---

# SaaS Redesign Skill

Transform a Vue 3 app's top-nav layout into a modern SaaS-style interface with a fixed left vertical sidebar.

## Step 1: Discover

Read the target App.vue file. Use `$ARGUMENTS` as the path if provided; otherwise default to `client/src/App.vue`.

Extract the following from App.vue:
- All `<router-link>` entries (path, label text)
- Logo/brand text (usually in a header element)
- All composables imported and used (e.g., `useFilterBar`, `useProfileMenu`, `useLanguageSwitcher`)
- Any modals, emits, or props passed between layout sections
- The existing `<style>` block — identify which classes are global utilities (`.badge`, `.card`, `.table`, `.status-*`, etc.) vs. layout/nav classes

## Step 2: Delegate to vue-expert (MANDATORY)

Per CLAUDE.md rules, ALL `.vue` file modifications MUST be delegated to the vue-expert agent. Do not edit App.vue yourself.

Pass the full context gathered in Step 1 to vue-expert with this instruction:

---

Rewrite `client/src/App.vue` to implement a SaaS-style layout with a fixed left vertical sidebar. All existing routing, composables, modals, and emits must remain fully functional — this is a layout-only transformation.

### Sidebar Layout Spec

**Sidebar** (`position: fixed; left: 0; top: 0; width: 240px; height: 100vh`):
- Background: `#0f172a` (dark slate)
- **Top section**: logo text + optional subtitle, padded, white text
- **Middle section** (flex-grow, scrollable if needed): vertical nav links
  - Each link: full-width, `padding: 10px 16px`, `color: #94a3b8`
  - Active state: `border-left: 3px solid #3b82f6`, `background: rgba(255,255,255,0.08)`, `color: #fff`
  - Hover state: `background: rgba(255,255,255,0.05)`, `color: #e2e8f0`
  - Leave a 20px left-padding area for a future icon (use a `<span class="nav-icon">` placeholder)
  - Use Vue Router's `router-link-active` or `:class` with `$route.path` for active detection
- **Bottom section**: stack LanguageSwitcher + ProfileMenu (or whatever composable components exist), padded, separated by a subtle border-top

**Main content area**:
- `margin-left: 240px`
- **Top bar** (thin, ~48px): shows current page title (derive from route name or a static map of path → label), right-aligned optional actions
- **FilterBar**: placed below the top bar, above page content (if FilterBar composable exists)
- Page `<router-view>` below FilterBar

### CSS Approach

- Use scoped styles in App.vue for all sidebar and layout classes
- Do NOT modify or remove any existing global utility classes (`.badge`, `.card`, `.table`, `.status-*`, `.btn`, etc.) — keep them in the unscoped `<style>` block exactly as they are
- Only the layout/nav classes change (remove old `.nav`, `.top-bar`, `.header` etc., add new `.sidebar`, `.main-content`, `.nav-link`, `.nav-icon` etc.)
- Maintain responsiveness awareness — sidebar can collapse to a narrow icon-only rail at `< 768px` if it doesn't break existing functionality, but a simple hidden sidebar is acceptable

### Routes to include in sidebar nav

[INSERT the router-link list discovered in Step 1]

### Composables / components to preserve

[INSERT the full list of composables, props, emits, and child components discovered in Step 1]

---

## Step 3: Verify

After vue-expert completes the rewrite:

1. Confirm the file was saved to `client/src/App.vue`
2. Check that all original `<router-link>` destinations are present in the new sidebar
3. Check that global utility CSS classes were not removed
4. Remind the user to run `cd client && npm run dev` and visually confirm the sidebar renders at `http://localhost:3000`
