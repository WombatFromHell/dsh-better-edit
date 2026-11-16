# Handoff: `dsh-sol-pi` and `dsh-better-edit` both claim `edit`

Defect: on dsh `0.2.2-rc.1`, `dsh-better-edit`'s hashline `edit` never lands and three of its tools are
missing entirely. Reproduced with a smoke against the real installed packages. Root cause is split across
the two plugins; the better-edit half is applied and installed, and the remaining work is a `dsh-sol-pi`
design change (R3 below).

Mechanism claims below are cited against the harness source checkout `~/Projects/deepseek-harness`
(the `packages/core/{tools,scope,agent}` sources) and against the installed 0.2.2-rc.1 packages.

## Symptom (live deployment, profile `web`)

Before any fix:

| tool                 | what the agent resolves                      |
| -------------------- | -------------------------------------------- |
| `read`               | better-edit hashline read — correct          |
| `edit`               | **dsh-sol-pi's fused pi-style edit** — wrong |
| `str_replace_editor` | absent                                       |
| `undo_last_edit`     | absent                                       |

After restart 1 (better-edit's synchronous, per-step install): all four names present, `edit` still
sol-pi's fused literal contract (`file_path`/`old_string`/`new_string`/`then_run`). Not a
schema/validation problem: the same definitions register fine when nothing holds the name first (smoke
scenario A).

## Mechanism (verified against the harness source)

- The registry is scope-layered. `ToolsService.register()` runs
  `layers.effect(ctx, layer => layer.tools.insert(name, …))` (`packages/core/tools/src/index.ts:117`), so a
  registration lands in **the calling scope's own layer**. The agent's scope key is the `Agent` object
  itself: `scopeTarget(agent, agent)` (`packages/core/agent/src/index.ts:479`) and
  `createScope(ctx, key)` / `scopeOf(ctx)` (`packages/core/scope/src/index.ts:137,154`). The agent layer is
  the only slot for a per-agent variant.
- A second registration of the same name **in one layer throws**: `NamedEntries.insert`
  (`packages/core/scope/src/store.ts:43-46`) raises `tool "edit" is already registered in this scope`. A
  global re-registration of a preset-provided name is refused outright —
  `tool "edit" is already registered (for a per-agent variant, register through that agent's \`agent.ctx\`
  instead)` (`packages/core/tools/src/index.ts:69`). So neither plugin can fuse or shadow globally, and the
nearest layer wins a name (`ScopedLayers.merge`, `store.ts:203`).
- **There is no public replace/override.** `NamedEntries` exposes `insert/get/has/keys/entries` only;
  `ToolsService.layers`, `ScopedLayers.scoped`, and `NamedEntries.data` are all private fields. The disposer
  `register()` returns deletes **by name** (`store.ts:48-53`), so a peer plugin's disposer removes whatever
  currently holds that name — a taken-back name is only recoverable by re-registration, which is exactly what
  this plugin's self-heal watcher does.
- `agent/created` is emitted **after** the agent scope is composed, awaited serially:
  `await this.ctx.serial(entry.carrier, 'agent/created', …)` (`packages/core/agent/src/index.ts:564`). Both
  plugins install from that listener.

So ownership of `edit` is "whoever registers in the agent layer first", and that order is **not** a stable
lever. Two theories were tried and both failed in the live harness:

- _Listener order_: better-edit's bundle row loads first, but restart 1 still showed sol-pi's fused `edit`.
- _Inject-pass order_: an `["agents"]` inject pass does resolve before listeners on a plain root context
  (`.scratch/smoke/order-probe.mjs`: `better pass -> sol-pi pass -> better listener -> sol-pi listener`), but
  during agent composition `child.agents.list()` is empty — the agent is not yet in the registry — so the pass
  never installed anything and restart 2 was unchanged. That pass has been **deleted** as inert.

The deterministic fix is therefore not ordering: whoever installs takes the contested name **back** and
re-registers. That is order-independent and survives any bundle order, any preset, and any peer plugin.

## Root cause 1 — `dsh-better-edit` (applied, installed)

Three defects in the install path:

1. **Deferred install.** The `effect(async …)` wrapper pushed every registration past the `agent/created`
   emit, so a peer plugin's synchronous install always claimed the agent-layer names first. Registrations now
   run synchronously inside the listener; only the guidance sections stay async (they need preset composition
   and file reads). Everything subscribes on `agent.ctx`, so agent-scope disposal is the cleanup — the manual
   disposer list is gone.
2. **One straight sequence.** A duplicate registration threw, the async effect rejected, Cordis only logged
   it, and every step after it never ran — which is why `str_replace_editor`, `undo_last_edit`, the `write`
   hook, the watcher, and guidance were all missing. Each step is now isolated by
   `own(label, register, takeName?)` (`src/index.ts:164`).
3. **A taken name stayed taken.** `own()` now reclaims and retries once:
   `reclaimToolEntry(toolsSvc, agent, name)` then `register()` again. The helper is shared with the self-heal
   watcher (`src/self-heal.ts`, same delete-then-re-register the watcher already shipped for #43), so install
   and heal use one mechanism. Contested names: `read`, `edit`, `str_replace_editor`, `undo_last_edit`; the
   `write` hook and guidance sections are not reclaimed (the built-in `write` is deliberately left alone).

Removed: the `["agents"]` inject-pass install described above — inert, and its comment claimed an ordering
that the harness source contradicts.

Checks: `pnpm run typecheck`, `pnpm test` (163 files, 1705 passing), `pnpm exec oxlint src/index.ts
src/self-heal.ts` (0 errors), `pnpm run build` — all green. New test: _"reclaimToolEntry drops a peer
plugin's entry so re-registration lands ours"_ (`test/core/self-heal.test.ts`).

## Root cause 2 — `dsh-sol-pi` (applied in its working tree)

`attachActionFusion` resolves the visible `edit`/`write`, wraps them with `then_run`, and re-registers on the
**agent's own layer** — which is forced, since a global re-registration is refused (see Mechanism). Applied:

- The guidance section is evaluated **per assembly scope** and advertises `then_run` only for names whose
  visible definition actually carries that parameter, so a shadowed `edit` is no longer promised to the model
  (`src/sol-dsh/index.ts`, `src/sol-dsh/host.ts` widened `PromptSection.text` to take the assembly context).
  Test: _"advertises then_run only for names the assembly scope can actually call"_.
- Fusion is **deferred into an async effect body** (`src/sol-dsh/action-fusion.ts:198`), so it registers after
  the `agent/created` emit settles instead of overwriting a contract another plugin claimed. It does not win
  the name (better-edit takes it back); it still earns its keep by not clobbering a peer's contract on a later
  layer change. Without an `effect` (tests, older hosts) the loop runs inline.
- `agent/session-start` no longer triggers a second fusion pass (it produced duplicate-registration warns).

Still open — **the wrap assumes the harness's literal contract.** `mutationPath`
(`src/sol-dsh/action-fusion.ts:47-50`) reads `path` / `file_path` / `filePath`. better-edit's `edit` takes
`file` + `edits[].anchor_from`, so a fused hashline `edit` resolves no path and every call reports
`then_run skipped: mutation path is unavailable; the command was not run`. Harmless today because sol-pi's
wrap is refused; wrong the moment it wins the name.

## Remaining changes to make in `dsh-sol-pi` (ranked)

**R3 — stop piggy-backing on mutation tools** (the honest fix): register `then_run` as **its own transport
tool** instead of widening someone else's schema. This removes the name race entirely, survives any plugin
that replaces `edit`, and removes the disposer hazard: today sol-pi's fusion effect owns a registration keyed
by `edit`, and that disposer deletes _by name_, so if sol-pi ever holds the name and better-edit has taken it
back, sol-pi's scope teardown removes better-edit's definition until the watcher heals it. Cost is a changed
model-facing contract, which is the plugin's value proposition.

**R1 — fuse only a contract you understand** (small interim, if R3 is not taken yet):

```ts
function isLiteralMutation(name: string, base: ToolDefinition): boolean {
  const p = (base.parameters?.properties ?? base.parameters) as Record<string, unknown> | undefined;
  return name === "write"
    ? p?.path !== undefined
    : p?.old_string !== undefined && p?.new_string !== undefined;
}
// in attachActionFusion, after `if (!base) continue;`:
if (!isLiteralMutation(name, base)) {
  ctx.logger?.warn?.(
    `dsh-sol-pi: ${name} is not the built-in literal contract (owned by another plugin); then_run not fused`,
  );
  continue;
}
```

**Not needed:** an ownership probe (`layers.peek(agent).tools.has(name)`) — private Cordis internals, and the
try/catch warn plus the per-scope prompt text already cover it. **Do not rely on bundle order** as the
arbiter: `dsh.profile.bundles` lists `dsh-better-edit` before `dsh-sol-pi`, but the live restarts showed order
does not decide the winner. Reordering the bundles changes nothing that matters.

**Worth filing upstream (dsh-tools):** a public way to own or replace a scoped tool name — e.g.
`tools.take(name, definition)` or `tools.ownedIn(scope, name)`. Both plugins currently reach into
`layers.scoped.get(agent).tools.data`, which is a private map and will break on any internal refactor.

## Paired change in `dsh-better-edit` (recommended, **not** applied)

The watcher deletes whatever currently holds the name and re-registers (`reclaimToolEntry`,
`src/self-heal.ts:31`). Against the preset's built-in that is the intended heal (#43); against a **peer
plugin's** variant it is a takeover. Narrow it to built-in intruders — skip the heal when the current
definition carries a `then_run` parameter or is not the preset's own definition — so sol-pi's variant is not
retaken mid-session. Needs its own ticket since it changes this plugin's behaviour. Note that install-time
takeover now fires regardless of order, so this narrowing is a behaviour question, not a bug fix.

## Verification

Smoke — real 0.2.2-rc.1 packages (`@deepseek-ai/cordis`, `dsh-tools`, `dsh-scope`) plus the plugin's built
`lib`, one agent scope per scenario, snapshotting that scope's own view after every step:

```
cd .scratch/smoke && rsync -a --delete ../../lib/ lib/ && DSH_HOME=/tmp/dsh-smoke-home node register-smoke.mjs
```

Current run (patched pair):

```
A clean          after install:          edit read str_replace_editor undo_last_edit | [edits,file,mode]
B sol-pi first   after install:          edit read str_replace_editor undo_last_edit | [edits,file,mode]
B sol-pi first   after a later change:   edit probe read str_replace_editor undo_last_edit | [edits,file,mode]
C better-edit first after sol-pi wrap:   edit read str_replace_editor undo_last_edit | [edits,file,mode]
PASS A / PASS B (install takes the taken edit back) / PASS B (hashline edit survives a later change) / PASS C (wrap refused)
```

Before the fix, scenario B was `edit read` only — matching the live GUI name-for-name — and B's install step
left sol-pi's fused contract in place (`!hashline`). C is the live deployment's ordering; it keeps the hashline
`edit` and all four tools, with sol-pi logging `could not shadow edit`.

`dsh-better-edit`: `pnpm run typecheck`, `pnpm test` (163 files, 1705 passing), oxlint 0 errors,
`pnpm run build` — all green. `dsh-sol-pi`: `npm run typecheck`, `npm test` (20 files, 201 passing),
`npm run build:dsh` — all green (unchanged this turn; rebuilt 10:53).

## Status

- `dsh-better-edit` rebuilt (`lib/`), repacked to `~/.dsh/local-patches/dsh-better-edit-1.2.1.tgz` (11:27),
  and `pnpm install` in `~/.dsh/profiles/web` materialized it: the installed
  `node_modules/dsh-better-edit/lib/index.js` contains the takeover call and **no** `["agents"]` pass.
- `dsh-sol-pi` is a `link:` dependency, so its rebuilt `dist/sol-dsh/index.js` (10:53, effect-deferred fusion)
  is what the profile loads.
- **Restart 1** (`Tool.listTools`): `read`, `str_replace_editor`, `undo_last_edit` present — crash-order defect
  gone — `edit` still sol-pi's fused literal contract.
- **Restart 2** (same list, with sol-pi's deferral loaded and better-edit's `["agents"]` pass installed):
  `edit` _still_ fused. That falsified the pass/listener ordering theory; the pass was inert and is deleted.
- **Restart 3** (with the pass installed, before the takeover build): unchanged, as expected — the takeover
  build was installed after it.
- **Needs one more harness restart.** Then the live agent should resolve `edit` as the hashline contract
  (`file`, `edits`), keep `read`/`str_replace_editor`/`undo_last_edit`, and sol-pi should log
  `could not shadow edit`.

## Not verified

- **Live behaviour after the next restart.** Everything above is verified in the smoke, the harness source, and
  unit tests — not yet in the running GUI.
- **0.2.1-rc.2 vs 0.2.2-rc.1 registry delta.** Only `0.2.2-rc.1` is installed anywhere on this machine, so the
  "worked before, broke now" claim is unconfirmed.
