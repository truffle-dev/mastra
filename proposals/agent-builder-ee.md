# Agent Builder EE — Proposal

**Status:** Draft for team review
**Scope:** Shape-only PR. No builder behavior.
**Asking for:** Sign-off on the config shape, class boundary, and license gate.

---

## TL;DR

We're adding **Agent Builder** as an EE-gated feature of the Editor. This PR
lands the _shape_ only — types, class skeleton, license check, package
wiring. Zero behavior. Follow-up PRs add the actual builder logic.

Three decisions we want buy-in on:

1. **Config:** single top-level `builder` key on `MastraEditorConfig`, typed
   as `AgentBuilderOptions` (a plain POJO). Users pass options, not a class.
2. **Access:** two methods on `MastraEditor` — `hasEnabledBuilderConfig()`
   (sync) and `resolveBuilder()` (async). No `editor.builder` property.
3. **Licensing:** boot-time check in `ServerAdapter.init()`, same pattern as
   RBAC. License check runs **before** any `@mastra/editor/ee` code loads.

```ts
// What users write
new Mastra({
  editor: new MastraEditor({
    builder: {
      enabled: true,
      features: { agent: {} },
      defaults: { agent: { memory: { /* SerializedMemoryConfig */ } } },
    },
  }),
});

// What consumers (future PRs) use
if (editor.hasEnabledBuilderConfig()) {
  const builder = await editor.resolveBuilder();
  // builder: IAgentBuilder | undefined
}
```

---

## Why This Feature

Mastra needs an EE-licensed path for agent creation that can be:

- **Configured by admins** — feature toggles, default memory shape, etc.
- **Permission-gated** — different roles see different surfaces (future RBAC).
- **Deployed without surprise** — unlicensed prod fails loudly at boot.

This replaces the existing standalone `@mastra/agent-builder` package,
which is being removed. The new surface lives inside the Editor where
admin configuration, licensing, and permissioning all belong.

---

## The Three Decisions

### 1. Config Shape: Single `builder` Key, Plain Options

```ts
// packages/core/src/editor/types.ts
export interface MastraEditorConfig {
  // ...existing fields
  builder?: AgentBuilderOptions;
}

// packages/core/src/agent-builder/ee/types.ts
export interface AgentBuilderOptions {
  enabled?: boolean;           // default: true
  features?: {
    agent?: {};                // placeholder shape
  };
  defaults?: {
    agent?: {
      memory?: SerializedMemoryConfig;
    };
  };
}
```

**Why this shape:**

- **One key, not five scattered fields.** Matches the ticket: "consolidate all
  builder options under a key."
- **Plain POJO, not a class instance.** Users don't instantiate anything.
  Editor handles construction. Fewer import paths for users to navigate.
- **Options live in `@mastra/core`, not `@mastra/editor`.** Lets
  `MastraEditorConfig` type the field without a reverse dep on the editor
  package. Mirrors RBAC's `IRBACProvider` split.
- **`enabled: boolean` is explicit**, separate from "is the key present?"
  Reason: we use key-presence elsewhere (`features.*`) for a different
  semantic ("show this section"). Conflating the two invites bugs.
- **`SerializedMemoryConfig`, not `MemoryConfig`.** Full `MemoryConfig`
  contains functions, Zod schemas, and class instances that don't round-trip
  through JSON. All `AgentBuilderOptions` fields must be JSON-safe — this is
  a hard rule for the whole type.

All top-level fields are optional. Sub-fields under `features` and
`defaults` are additive — new ones land as consumers need them without
breaking existing configs.

---

### 2. Access: Two Methods, Not a Property

```ts
class MastraEditor implements IMastraEditor {
  // Sync. OSS-safe. Does NOT import @mastra/editor/ee.
  hasEnabledBuilderConfig(): boolean;

  // Async. Dynamic-imports @mastra/editor/ee on first call. Caches.
  resolveBuilder(): Promise<IAgentBuilder | undefined>;
}
```

**Why not a simple `editor.builder` property?**

We tried that first. Two things broke:

1. **License-check ordering.** If `editor.builder` is a getter that loads
   the EE class, the validator calling `editor.builder` forces the import
   *before* the license check can fail. Unlicensed prod would surface a
   missing-module error or constructor error instead of the correct
   license error.
2. **Core → editor dependency.** Typing `editor.builder` as the concrete
   `EditorAgentBuilder` class in `IMastraEditor` (core) requires core to
   depend on editor. Wrong direction.

The split solves both:

- **`hasEnabledBuilderConfig()`** is sync and reads only private state. The
  validator uses it. No EE code is touched until the license is verified.
- **`resolveBuilder()`** is async and returns `IAgentBuilder` — a small
  structural interface in core. Callers (routes, follow-up behavior)
  trigger the dynamic import only after boot-time gates have passed.

**Structural interface in core:**

```ts
// packages/core/src/agent-builder/ee/types.ts
export interface IAgentBuilder {
  readonly enabled: boolean;
  getFeatures(): AgentBuilderOptions['features'];
  getDefaults(): AgentBuilderOptions['defaults'];
}
```

Purely a type. No runtime machinery. Lets `IMastraEditor` reference the
builder's shape without pulling in the EE class.

---

### 3. Licensing: Boot-Time Check in `ServerAdapter.init()`

Mirrors RBAC's existing pattern at
`packages/server/src/server/server-adapter/index.ts:485`.

```ts
async validateAgentBuilderLicense(): Promise<void> {
  const editor = this.mastra.getEditor();
  if (!editor?.hasEnabledBuilderConfig()) return;

  try {
    const { isEEEnabled } = await import('@mastra/core/auth/ee');
    if (!isEEEnabled()) {
      throw new Error('[mastra/auth-ee] Agent Builder requires a valid EE license…');
    }
  } catch (err) {
    if (err instanceof Error && err.message.startsWith('[mastra/auth-ee]')) throw err;
    throw new Error('[mastra/auth-ee] EE module could not be loaded…');
  }
}
```

**Guarantees:**

| scenario | behavior |
|---|---|
| `builder` omitted | validator returns; no license check; no EE import |
| `builder.enabled: false` | same |
| `builder` configured, licensed prod | builder available via `resolveBuilder()` |
| `builder` configured, unlicensed prod | hard boot error (license) |
| `builder` configured, `@mastra/core/auth/ee` missing | hard boot error |
| Dev environment | license check bypassed (existing `isDevEnvironment()` rule) |

Same hard-boot-error pattern as RBAC. Same message shape. Same escape hatch
for dev.

---

## Class Skeleton (What Ships in This PR)

```ts
// packages/editor/src/ee/agent-builder.ts
import type { AgentBuilderOptions, IAgentBuilder } from '@mastra/core/agent-builder/ee';

export class EditorAgentBuilder implements IAgentBuilder {
  constructor(private readonly options: AgentBuilderOptions = {}) {}

  get enabled(): boolean {
    return this.options.enabled !== false;
  }

  getFeatures(): AgentBuilderOptions['features'] {
    return this.options.features;
  }

  getDefaults(): AgentBuilderOptions['defaults'] {
    return this.options.defaults;
  }
}
```

Getters only. No methods. Behavior lands in follow-up PRs without breaking
this surface.

**Class name: `EditorAgentBuilder`.** The name signals that this class is
scoped to the Editor's builder surface — the thing admins configure and
users interact with through the Editor. Keeps room for other "builder"
concepts to exist later without name contention.

> Context: the standalone OSS `@mastra/agent-builder` package is being
> removed. We could use plain `AgentBuilder` once that lands, but
> `EditorAgentBuilder` is a clearer name on its own merits and we don't
> want to couple this PR's naming to the removal timeline.

---

## What's Out of Scope (Deliberately)

Shape-only means we're _not_ shipping:

- Any builder behavior (`createAgent`, `preview`, etc.)
- RBAC enforcement (flagged below as a known follow-up)
- UI enforcement of features/visibility
- Storage namespace for builder state
- Non-agent artifact types (skills, workflows)

All of these are additive and non-breaking on top of the shape.

---

## Open Questions / Known Follow-Ups

Nothing below is blocking this PR. Called out so nobody is surprised later.

### RBAC Vocabulary

The generated `agent-builder` RBAC resource and its HTTP-derived verbs
(`:read`, `:write`, `:execute`, `:*`) exist today only because of the OSS
`@mastra/agent-builder` package's server handlers. When that package is
removed, those generated entries go with it.

That's actually convenient: when we wire RBAC for Agent Builder EE in a
follow-up, we're working from a blank slate rather than fitting into a
generated model that didn't match our needs.

This PR does not lock any vocabulary. It just leaves the integration point
clean.

### Builder Lifecycle

Right now `EditorAgentBuilder` is a config holder — no dependencies.
Once behavior lands, we'll decide whether it gets:

- The `Mastra` instance reference (for storage/tools/RBAC)
- A dedicated `registerWithMastra` hook
- A narrower injection

Not decided here.

### Resolver Simplification

`hasEnabledBuilderConfig()` + `resolveBuilder()` is intentionally minimal.
Once real consumers exist, we may collapse to a single boot-time resolution
(e.g., server init resolves and caches for the process). The current shape
doesn't prevent that.

---

## Risks & Mitigations

| risk | mitigation |
|---|---|
| EE code leaks into OSS runtime graph | Dynamic import only; no static `./ee` imports from root entry. Same pattern as RBAC. |
| License validator imports EE class before license check | Sync `hasEnabledBuilderConfig()` reads only raw config. No EE import in the validator path. |
| Bundlers still emit an EE chunk for OSS consumers | Bundler-dependent. Matches RBAC's posture. "Not in runtime graph" is the guarantee; "not in build artifact" is best-effort. |
| Future RBAC work locks into a vocabulary that later doesn't fit | No vocabulary shipped in this PR. Generated `agent-builder` resource goes away with the OSS package, leaving a clean slate. |
| `SerializedMemoryConfig` drift | Type is owned by `@mastra/core/memory` and already used by stored agent snapshots. Shared definition, not a fork. |

---

## Files Touched

**New:**
- `packages/core/src/agent-builder/ee/types.ts` — `AgentBuilderOptions`, `IAgentBuilder`
- `packages/core/src/agent-builder/ee/index.ts` — barrel
- `packages/editor/src/ee/agent-builder.ts` — `EditorAgentBuilder`
- `packages/editor/src/ee/index.ts` — barrel
- Tests for shape, `enabled`, resolver behavior, license gating

**Modified:**
- `packages/core/src/auth/ee/license.ts` — add `'agent-builder'` to feature list (payload only)
- `packages/core/src/editor/types.ts` — `builder?` on `MastraEditorConfig`; two methods on `IMastraEditor`
- `packages/editor/src/index.ts` — implement the two methods; store raw options
- `packages/server/src/server/server-adapter/index.ts` — `validateAgentBuilderLicense()` call in `init()`
- `packages/editor/package.json` — `./ee` export + build entry
- `packages/core/tsup.config.ts` — explicit entry for `src/agent-builder/ee/index.ts`

---

## Test Plan

Critical cases covered:

- `builder` omitted → no EE import; validator passes
- `builder.enabled: false` → no EE import; validator passes
- `builder.enabled: true` + licensed → resolver returns instance; import fires once; cached
- `builder.enabled: true` + unlicensed prod → license error at boot, **before** any EE import
- Missing `@mastra/core/auth/ee` when builder enabled → hard boot error (matches RBAC)
- Missing `@mastra/editor/ee` at resolve time → clear runtime error
- `@mastra/editor/ee` throws during evaluation → `resolveBuilder()` rejects, doesn't cache bad value
- Validator reads `mastra.getEditor()`, not `mastra.getServer()?.editor` (regression guard)
- Boot ordering: editor set before server; validator sees populated editor

---

## What We're Asking

Sign-off on:

1. The three decisions above (config shape, access methods, license gate).
2. The `EditorAgentBuilder` class name.
3. The deferred items as legitimately out of scope for a shape PR.

If any of those feel wrong, better to push back now than after the
follow-up behavior PRs land against this surface.
