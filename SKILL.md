---
name: makeswift-components
description: >-
  Create or update Makeswift page-builder components in a BigCommerce Catalyst
  storefront (Next.js). Use this whenever the user wants to add a new
  drag-and-drop component to the Makeswift visual builder, add/change a prop
  or control on an existing Makeswift component, wire builder-editable content
  (fonts, images, links, lists of items) into a component, or mentions
  `registerComponent`, `lib/makeswift/components/`, the Makeswift builder
  panel, or a component "not showing up" / "prop not editable" in Makeswift.
---

# Makeswift Components (Catalyst / Next.js)

Makeswift components live in two layers in a Catalyst repo:
- `vibes/soul/...` — the actual presentational React component (or a section under `app/`), with **no** Makeswift import. This is what gets reused across the whole app, not just the builder.
- `lib/makeswift/components/<name>/` — a thin adapter that registers a builder-facing version of that component: `register.ts` (the `runtime.registerComponent(...)` call + control definitions) and `client.tsx` (a `'use client'` wrapper that maps Makeswift's prop shape onto the underlying component's real props, doing any data-fetching the visual/static component doesn't do itself).

Keep this separation. Never import `@makeswift/runtime/controls` from a `vibes/soul/` file.

## Workflow

1. **Find precedent first.** Look at an existing sibling under `lib/makeswift/components/` that's structurally similar (a carousel, a list, a single-item card) and copy its shape rather than inventing a new pattern. Check `references/controls.md` in this skill for the exact control API if you're unsure of an option name.
2. **Decide: new component or new prop on an existing one?**
   - New component → create `lib/makeswift/components/<name>/register.ts` + `client.tsx`, then register the import somewhere the runtime bootstrap already imports all components from (grep for how a sibling folder gets pulled in — usually a central `components.ts`/`register.ts` barrel).
   - New prop → add the control in `register.ts`'s `props: {}`, then thread it through `client.tsx` into whatever it needs to reach. If the client wrapper does `<UnderlyingComponent {...props} />` (spreading the rest), and the new prop is meant for `UnderlyingComponent`, check whether `MSComponentProps` is typed via `ComponentPropsWithoutRef<typeof UnderlyingComponent>` — if so, adding the prop to the underlying component's own interface is enough for it to flow through automatically; you only need to touch `client.tsx` if the wrapper explicitly destructures every prop (some do, to pass to loading/empty/skeleton states too — check for those branches, they're easy to miss).
3. **Wire the control to a real prop type.** Every control resolves to a specific TS shape (see `references/controls.md`). Match the underlying component's prop type to that shape exactly — don't invent a different shape and hope it coerces.
4. **If the component needs BigCommerce data** (products, categories, prices): Makeswift components render **client-side** in the visual builder canvas, so they generally can't do server-side GraphQL fetches directly. The established pattern in this codebase is: a Next.js Route Handler under `app/api/...` does the GraphQL fetch server-side, a `use...` hook (SWR-based) calls that route from the client wrapper, and a Zod schema re-validates the JSON that crossed the network boundary. Follow that pattern rather than trying to `await` a GraphQL client call inside a `'use client'` component.
5. **Verify**, in this order:
   - `pnpm run typecheck`
   - `pnpm exec eslint <changed files>` (full `pnpm run lint` also runs `next typegen` as a prelint step, which fails locally with "Client configuration must include a channelId" if `.env.local` isn't configured — that failure is expected/unrelated in an unconfigured sandbox; the eslint step itself still runs and matters)
   - If you touched a GraphQL fragment/query, run `pnpm run generate` first (needs `.env.local`) so `gql.tada` types pick up the new fields — if that's not available, at least sanity-check the field exists in the committed `bigcommerce.graphql` schema file.

## Gotchas specific to this stack

- **Tailwind class safety.** Any Tailwind utility class must appear as a *literal string* somewhere Tailwind's `content` scanner reads — string concatenation/interpolation (`` `@2xl:basis-1/${n}` ``) will not generate CSS. For a value driven by a number (e.g. "visible items 1–6"), build a literal lookup object (`{1: 'basis-full', 2: 'basis-full @md:basis-1/2 ...', ...}`) with every full class string spelled out, and index into it — don't compose the string at runtime.
  - Check `tailwind.config.js`'s `content` array actually includes the directory you're editing. We found and fixed a real bug this session where `./lib/**/*.{ts,tsx}` was missing entirely — classes used only inside `lib/makeswift/components/**` were silently dropped from the compiled CSS (while the same fractions used elsewhere in `vibes/**`, which *was* scanned, kept working, masking the gap until an unused-elsewhere fraction like `basis-1/5` exposed it).
- **`hidden: true` components.** Site-wide slots (header, site theme provider) are registered with `hidden: true` so they don't appear in the component picker — they're rendered by the app shell itself, not drag-and-dropped. Don't add `hidden: true` to a normal reusable component; that's the one thing that makes it invisible in the builder even though it's fully registered.
- **`type` is permanent.** Once a component type string has been used in a live Makeswift page, don't change it — existing instances break ("Component not found"). Add new props instead of renaming/retyping.
- **Font tokens (Site Theme).** This project exposes global heading/body/accent fonts as `Font` controls with `variant: false` in `lib/makeswift/controls/font-tokens.ts`, each `defaultValue: { fontFamily: 'var(--font-family-<name>)' }`. Those CSS variables are set by `next/font` in `app/fonts.ts` and applied via `<html className={clsx(fonts.map(f => f.variable))}>`. To make a new Google Font available: add it in `app/fonts.ts` (register with `next/font/google`, give it a `--font-family-*` variable, add it to the exported `fonts` array), *then* it can be referenced as a font-token default or picked live in the builder's Fonts panel. Editing a token's code `defaultValue` only changes the *fallback* used before anyone customizes it in the builder — if the project's Makeswift Site Theme already has a saved override, the code default won't take effect until that's changed in the builder UI too.
- **Image dimensions without a client-side load.** Prefer `Image({ format: Image.Format.WithDimensions })` over the default URL-only format whenever the consuming component needs `width`/`height` up front (e.g. to size a container via `aspect-ratio` before the browser has fetched the image) — this avoids a load-then-resize layout jump. The resolved prop becomes `{ url, dimensions: { width, height } } | undefined` instead of a bare `string`.
- **`ComponentPropsWithoutRef<typeof X>` reuse.** Several Makeswift wrappers type their props as `Omit<ComponentPropsWithoutRef<typeof UnderlyingComponent>, 'someOverriddenProp'> & { makeswiftOnlyProps }`. This means extending the underlying `vibes/soul` component's own prop interface is often *sufficient* to make a new capability available to Makeswift — no `register.ts`/`client.tsx` change needed beyond adding the control and (if the wrapper destructures explicitly rather than spreading) threading it through.
- **Complexity limits.** ESLint enforces a max cyclomatic complexity (20) per function. Piling several features onto one component (hover state + swatches + preload + skeleton branches, etc.) can trip this — split into a custom hook (`use-thing.ts`) and/or child components rather than fighting the linter with more branches in one function.
- **`hidden: true` components are rendered via `<MakeswiftComponent>`, not the picker.** The concrete mechanism behind global slots (header, site theme) is: register normally with `hidden: true`, then somewhere in server-rendered layout code call `client.getComponentSnapshot(id, { siteVersion })` and render `<MakeswiftComponent snapshot type label />` with `type` matching `registerComponent`'s `type`. If you're adding a *new* global slot (not a new draggable component), this is the pattern to follow — see `references/runtime-and-components.md`.
- **Tailwind v4 + Makeswift CSS reset conflict.** If Tailwind classes seem to do nothing specifically inside Makeswift-rendered content, check `RootStyleRegistry`'s `enableCssReset` prop — it defaults to `true` and its unlayered normalize.css reset silently beats Tailwind v4's `@layer`-based reset. Set `enableCssReset={false}` where `RootStyleRegistry` is set up.
- **Layout shift right after hydration** that correlates with a context value differing between server and client render (locale, session, feature flag) → wrap that value with `useDeferredValue` rather than letting it interrupt hydration.
- **Slots inside conditionally-rendered/collapsed content** (e.g. an accordion panel) aren't reachable to drag components into while collapsed — the builder user has to switch to Interact mode, open it, switch back to Build mode, then drag into the slot.

## Reference

- `references/controls.md` — every control type, its options, and its resolved prop shape.
- `references/runtime-and-components.md` — `ReactRuntime` setup, `<Page>`/`<ReactRuntimeProvider>`/`<MakeswiftComponent>`/`<RootStyleRegistry>`, `MakeswiftClient` methods, `MakeswiftApiHandler` + custom font registration, and a troubleshooting quick-reference for the builder's own connection/rendering error messages.
