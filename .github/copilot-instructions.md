# react-low-code — Project Context for AI Agents

## What this is
A POC low-code framework: JSON configs describe a tree of Chakra UI widgets. A React
rendering engine interprets that JSON into live UI, resolves data bindings against a
lightweight global store, and dispatches events through a named action-handler registry.
Bootstrapped with the TanStack Router CLI. This is a client-facing POC (not production),
built by an internal team, meant to be swapped later for the client's real design system.

See [PLAN.md](../PLAN.md) for the full phased implementation plan, file list, and
verification steps. This file captures the *why* behind the architecture so any agent
(or developer) picking up the repo has the full decision context, not just the steps.

## Core architecture decisions (and why)

1. **Config format: JSON + named-action registry, not a JS/JSX config file.**
   - JSON-only widget/prop schema keeps configs pure data — safe to store/transmit/persist,
     diffable, and buildable into a future visual editor UI.
   - A JS config file was considered (native functions, less DSL overhead) but rejected:
     executing arbitrary JS from a config is an injection risk (OWASP) if configs are ever
     loaded from an untrusted/remote source, and functions don't serialize into a DB for a
     future builder UI.
   - **Rule: never use `eval`/`new Function` to run logic from JSON.** All dynamic
     behavior goes through the action registry (real TS functions, referenced by name).

2. **Event handling: named-action + handler registry** (chosen over a declarative action
   DSL, and over inline JS strings evaluated at runtime).
   - JSON events reference an action by name, e.g. `"onClick": "submitForm"`.
   - `src/actions/registry.ts` maps action names to real functions:
     `(ctx: { state, setState, event, params }) => void`.
   - This gives unlimited logic complexity (real JS/TS) without the JSON layer ever
     capping what a handler can do, while keeping the schema itself safe/data-only.
   - A pure declarative DSL (`{"action":"setState", "path":"...", "value":"..."}`) was
     considered but rejected as insufficient for complex logic without endless DSL growth.

3. **JSON schema shape: component-tree model**, not a schema+uischema split (as in
   react-jsonschema-form/JSON Forms).
   - Each node: `{ id, type, props, bindings?, events?, children? }`.
   - Chosen because it maps directly to "widgets that render UI" and suits general UI
     (dashboards, carts), not just forms.

4. **Bindings: `{{state.path}}` dot-path strings only**, resolved by simple property
   lookup — no expression language/evaluator for this POC. Keep it that way unless a
   real need for computed expressions arises; if so, use a safe expression library
   (e.g. a restricted evaluator), never raw `eval`.

5. **State: single lightweight global store per page** (React context + state), seeded
   from each page config's initial `state`. Exposed to both bindings and action handlers.
   Decided over per-widget local state because event handlers/complex UI need to
   read/write shared state almost immediately.

6. **Design system: Chakra UI** for the POC (chosen over Mantine/shadcn/ui/Ant Design for
   simplest theming model and fastest ramp-up). The widget registry (`src/widgets/`) is
   the intentional swap point — replacing Chakra with the client's real design system
   later should only mean rewriting the registry's component mappings, not the engine.

7. **Data layer: static mock JSON only, synchronous.** No simulated network delay/loading
   states in this POC — deliberately simple. If async is added later, do it inside action
   handlers (e.g. a `fetchData` action), not inside the rendering engine.

8. **Libraries per view:**
   - Dashboard: Chakra `Select` for filter dropdowns (no extra dependency), Recharts for
     charts, **AG Grid Community** (not Enterprise — no license key available) for the grid.
   - Form: Chakra Input/Select/Button; validation logic lives inside the submit action
     handler, not the schema.
   - Shopping Cart: add/remove/qty + running total only — no checkout flow in scope.

9. **Scaffolding:** TanStack Router CLI (`npx @tanstack/cli create --router-only`) —
   file-based routing, TypeScript, no Tailwind (Chakra owns styling), npm.

## Security rules for anyone extending this
- Never introduce `eval(...)` or `new Function(...)` to interpret JSON-driven logic.
- All events/actions must resolve through the named-action registry, never inline code
  parsed from JSON.
- Treat JSON configs as potentially untrusted input once this moves beyond hardcoded
  local files (e.g. if configs come from a DB or API) — validate against the widget
  registry's known types before rendering (see PLAN.md verification step 4).

## Where to look
- `PLAN.md` — phased build steps, file list, verification checklist.
- `src/engine/` — Renderer, binding resolver, store (core, build first).
- `src/widgets/registry.tsx` — widget type → component map.
- `src/actions/registry.ts` — named action handlers.
- `src/configs/*.json` — per-page JSON configs (dashboard/form/cart).
