## Plan: JSON-Driven Low-Code React Rendering Engine (POC)

TL;DR: Build a POC low-code framework where JSON configs describe a component tree of
Chakra UI widgets; a React rendering engine interprets the JSON into live UI, resolves
`{{state.path}}` bindings against a lightweight global store, and dispatches events through
a named action-handler registry (real JS functions, so complexity is never capped by the
JSON layer). Scaffold via TanStack Router CLI (file-based routing, TypeScript, npm). Three
demo views prove the engine: a filterable Dashboard (dropdown filters + Recharts charts +
AG Grid Community grid), an interactive Form (validation + submit + results panel), and a
Shopping Cart (product list, add/remove/qty, running total) — all data static mock JSON,
no async simulation.

**Steps**

Phase 0: Scaffold project
1. Run `npx @tanstack/cli create --router-only` — file-based routing, TypeScript, no
   Tailwind, npm.
2. Install deps: @chakra-ui/react (+ @emotion/react, @emotion/styled, framer-motion),
   ag-grid-community + ag-grid-react, recharts.
3. Chakra provider at root with a theme file (basic color/spacing/typography tokens)
   wrapping the TanStack router root route.

Phase 1: Core rendering engine (foundational — blocks Phase 2)
4. Define JSON schema/TS types: `WidgetNode { id, type, props, bindings?, events?, children? }`,
   page config `{ id, state, tree: WidgetNode[] }`.
5. Widget registry: `type -> React component` map (Box, Stack, Text, Heading, Button,
   Input, Select, plus composite widgets: AgGridWidget, RechartsWidget, ProductCard,
   CartItem, etc.).
6. Binding resolver: resolves `{{state.path}}` strings via simple dot-path get (no eval).
7. Action registry + dispatcher: `src/actions/registry.ts` maps named actions to
   `(ctx: {state, setState, event, params}) => void`; JSON events reference actions by name.
8. Global store: React context + useState/useReducer per page, seeded from page config's
   initial state, exposed to bindings and action handlers.
9. `Renderer` component: recursively walks JSON tree, resolves bindings, looks up widget
   registry, wires event props to action dispatcher, renders children.

Phase 2: Demo views (each *depends on Phase 1*; views can be built in parallel with each other)
10. **Dashboard** — 2-3 Chakra `Select` filter widgets bound to state; Recharts widget
    whose data is derived by client-side filtering static mock dataset by filter state;
    AG Grid Community widget below with same filtered dataset. Filter change -> action ->
    state update -> bindings recompute -> chart/grid re-render (synchronous).
11. **Form** — Chakra Input/Select/Button; validation logic lives in the submit action
    handler (checks required fields, sets `errors` state slice); results panel bound to
    `state.result`, hidden until successful submit. This is the flagged core focus area
    (event/action system).
12. **Shopping Cart** — static mock product list (`ProductCard` composite widget) with
    "Add to Cart" action appending to `state.cart`; cart panel listing `state.cart` with
    qty +/- and remove actions; running total via binding/derived state. No checkout step.

Phase 3: Wiring routes
13. One TanStack file-based route per view (`/dashboard`, `/form`, `/cart`), each loading
    its JSON config through the shared `Renderer`.
14. Simple nav/layout shell (Chakra Tabs or sidebar) to switch between views for the demo.

**Relevant files** (to be created)
- `src/configs/dashboard.json`, `form.json`, `cart.json` — page JSON configs
- `src/widgets/registry.tsx`, `src/widgets/*` — Chakra + AG Grid + Recharts widget adapters
- `src/actions/registry.ts` — named action handlers
- `src/engine/Renderer.tsx`, `src/engine/bindings.ts`, `src/engine/store.tsx` — core engine
- `src/routes/dashboard.tsx`, `form.tsx`, `cart.tsx` — TanStack file-based routes
- `src/theme.ts` — Chakra theme tokens
- `src/mock-data/*.json` — static datasets for dashboard/cart

**Verification**
1. `npm run dev` — manually exercise all 3 views: dashboard filters update chart+grid;
   form validation errors + successful submit shows results panel; cart add/remove/qty
   updates running total.
2. `npm run build` succeeds with no TypeScript errors.
3. Grep codebase for `eval(` / `new Function(` — must return nothing (confirms bindings/
   actions stay injection-safe, no raw JS execution from JSON).
4. Dev-time console warning for any JSON widget `type` not present in the registry (avoids
   silent render failures).

**Decisions**
- Config format: JSON (schema/props) + named-action registry for events. No eval, no raw
  JS in JSON. Chosen over a JS config file for safety/serializability (future visual
  builder friendliness), while action handlers remain full JS (no complexity ceiling).
- Design system: Chakra UI for POC (swap point isolated to widget registry layer for
  later replacement with client's real design system).
- JSON schema shape: component-tree model (`{type, props, events, children}`), not a
  schema+uischema split — simpler, matches general UI (not just forms).
- Bindings: `{{state.path}}` dot-path only, no expression language, for this POC.
- State: single lightweight global store per page, seeded from page config.
- Data: all static mock JSON, synchronous — no simulated network delay/loading states.
- Dropdowns: Chakra's own `Select` (no extra dependency) for consistency.
- AG Grid Community edition (free, no license key needed); Recharts for charts.
- Shopping cart: add/remove/qty + running total only, no checkout flow.
- Scaffolding: TanStack Router CLI, file-based routing, TypeScript, no Tailwind, npm.

**Status**: Plan approved-pending — awaiting explicit user go-ahead before implementation.
