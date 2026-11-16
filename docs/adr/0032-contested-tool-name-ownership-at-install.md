# ADR-0032 — A contested agent-layer tool name is taken back at install, not won by ordering

Status: Accepted (fix for [#123](https://github.com/Rianico/dsh-better-edit/issues/123), lane `i123`).
Related records: [ADR-0004](0004-orphaned-serve-healing.md) (the self-heal watcher this decision reuses),
[ADR-0015](0015-shadow-str-replace-editor.md) (the governed shadow that was silently missing),
`docs/handoff-dsh-sol-pi-tool-ownership.md` (the live reproduction and the `dsh-sol-pi` half).
Downstream records: `src/index.ts` (`installAgentTools`, `own`, `installOnce`),
`src/self-heal.ts` (`reclaimToolEntry`), `test/core/self-heal.test.ts` (the pin).

## Context

The hashline tools register on the agent's own scope layer, where the nearest layer wins a name. dsh's
registry gives no public replace: registration lands in the calling scope's layer
(`packages/core/tools/src/index.ts:117`), a second registration of the same name **in one layer throws**
(`NamedEntries.insert`, `packages/core/scope/src/store.ts:43-46`), a global re-registration of a
preset-provided name is refused outright, and the disposer `register()` returns deletes **by name**.
`dsh-sol-pi` fuses `edit` from its own listener, so both plugins race for `edit`, and `agent/created` is
emitted after the agent scope is composed, awaited serially — which plugin lands first is not a stable
lever. Both orderings were tried against the live harness and both lost: bundle order (better-edit loads
first, sol-pi still held `edit`) and an `["agents"]` inject pass (inert during composition, because
`child.agents.list()` is empty before the agent is registered).

Measured on the tree at `009b922`, three defects compounded in `installAgentTools`:

1. The install body ran inside `agent.ctx.effect(async ...)`, which starts only after the `agent/created`
   emit settles, so every registration landed after a peer plugin's synchronous install.
2. The steps were one straight sequence. A duplicate name threw, the effect rejected, Cordis logged it,
   and every step after the throw never ran — which is why `str_replace_editor`, `undo_last_edit`, the
   `write` hook, the watcher, and guidance were all absent, name-for-name, in the live GUI.
3. A taken name stayed taken: nothing reclaimed it, so re-registration could not succeed.

## Decision

1. **Register synchronously inside the `agent/created` listener.** The agent scope is already composed by
   the time the harness emits the event, so the tools are installed while the listener runs. Guidance stays
   async (it needs preset composition and file reads) and lands after the tools. Everything subscribes on
   `agent.ctx`, so agent-scope disposal is the cleanup; the manual disposer list is gone.
2. **Isolate every step.** `own(label, register, takeName?)` catches, and a failure warns with the label
   instead of aborting the rest of the install. A dropped tool is a warning, never a silent cascade.
3. **Take the contested name back and re-register once.** `reclaimToolEntry(toolsSvc, agent, name)`
   deletes whatever currently holds the name on the agent layer, then `register()` runs again. Contested
   names: `read`, `edit`, `str_replace_editor`, `undo_last_edit`. The helper is shared with the self-heal
   watcher, so install and heal use one mechanism (ADR-0004's heal, same delete-then-re-register).
4. **The built-in `write` is not reclaimed**, and the inject pass is deleted as inert.

Ownership is therefore order-independent: it survives any bundle order, any preset, and any peer plugin.

## Consequences

- A peer plugin's variant of a contested name is taken over at install. Against the preset's built-in that
  is the intended behaviour; against a peer's variant it is a takeover, which is a behaviour question, not
  a bug fix — narrowing the heal to built-in intruders is deferred to its own ticket.
- Reaching into `layers.scoped.get(agent).tools.data` is a private Cordis map. There is no public
  `tools.take(name, definition)` or `tools.ownedIn(scope, name)`; filing one upstream is the upgrade path
  for this seam (see the handoff doc, "Worth filing upstream").
- `then_run` remains sol-pi's contract, not ours: this plugin does not fuse it, so a fused hashline `edit`
  cannot report a mutation path it does not have.
