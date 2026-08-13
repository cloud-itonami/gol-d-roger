# gol-d-roger — gold asset management (design document + UI scaffold, not an implementation)

The name says nothing about the subject, so: **this repo is about gold** — spot
and futures pricing, physical/paper/digital holdings, vault inventory, tax lots.
`gol-d-roger` is a One Piece reference that happens to contain "gold"; it is a
discovery alias, not a description. `CLAUDE.md` is the design document.

`README.edn` remains the canonical machine-readable metadata
(`:canonical-metadata :edn`). This file is prose for a human arriving cold, and
does not override it.

## What is actually in here

Sixteen files. Fourteen were lifted verbatim out of `etzhayyim/root` at
`60-apps/etzhayyim-project-gol-d-roger`; two (`README.edn`, `migration.edn`)
were added by the extraction itself. That claim is machine-checkable — see
[`docs/check-migration-identity.cljs`](docs/check-migration-identity.cljs).

| | |
|---|---|
| `CLAUDE.md` (8.6 kB) | The design document: control plane, risk engine, Arrow tables, XRPC fan-out to 12 ISIC/ISCO actors, six Matrix rooms. |
| `appview/etzhayyim-wasm-gol-d-roger-wy2zvdvd/svelte/` | A Vite + Svelte 5 scaffold. `App.svelte` renders one `<h1>` and the sentence "Vite entry scaffold after SvelteKit cleanup." |
| `PROJECT.jsonld`, `kotodama.jsonld`, `migration.edn`, `NOTICE` | Metadata and provenance. |

**Nothing in the design document is implemented here.** There is no `proto/`,
no `src/` at the repo root, no service, no Arrow table, no XRPC client, no test.
The gap between `CLAUDE.md` and the tree is the single most important thing to
know before planning work against this repo — read the document as an intent,
not as a description of code that exists.

## Declared hosts — 4 of 5 do not exist (measured 2026-08-13)

The metadata names hosts the app would talk to. Re-measure with
[`docs/check-surface.cljs`](docs/check-surface.cljs); it extracts the hosts from
the files that declare them rather than carrying a hardcoded list.

| Host | DNS | Declared in (config and design files) |
|---|---|---|
| `br8bojxp.etzhayyim.com` | NXDOMAIN | `CLAUDE.md` |
| `etzhayyim.com` | resolves | `CLAUDE.md`, `kotodama.jsonld` |
| `gol-d-roger.etzhayyim.com` | NXDOMAIN | `CLAUDE.md`, `PROJECT.jsonld`, `kotodama.jsonld` |
| `resources.etzhayyim.com` | NXDOMAIN | `PROJECT.jsonld` |
| `wy2zvdvd.etzhayyim.com` | NXDOMAIN | `PROJECT.jsonld` |

Control host `registry.npmjs.org` resolved, so the NXDOMAINs mean absence and
not a broken resolver. If this table and the script ever disagree, the script is
right and this table is stale.

The script's own "declared in" column is longer than this one: it scans `.md`
too, so it also attributes every host to this README and to the quickstart,
which name them in prose. That is the script being literal, not a sixth
declaration site — the hosts originate in the three files above.

## The UI does not build from this repo alone

Measured 2026-08-13 with node v26.3.0 / npm 11.16.0 / pnpm 10.26.2. Three
*independent* reasons, each of which must be fixed:

1. `npm install` fails `EUNSUPPORTEDPROTOCOL` — `package.json` declares two
   `workspace:*` dependencies, a protocol npm does not implement.
2. `pnpm install` fails `ERR_PNPM_WORKSPACE_PKG_NOT_FOUND` — pnpm implements
   `workspace:`, but neither package is present, because the workspace root was
   left behind in `etzhayyim/root`.
3. Even with both removed, `@sveltejs/vite-plugin-svelte@4.0.4` peer-requires
   `vite@^5.0.0` while `package.json` asks for `vite@^6.4.2`. The scaffold is
   internally inconsistent regardless of the workspace question.

Of the two `workspace:*` dependencies, only one is real:

- **`@etzhayyim/design-system` is genuinely required** — `tailwind.config.js`
  imports `etzhayyimUIKit` from `@etzhayyim/design-system/plugin`. Its `content`
  globs also point at `../../../../../packages/ts/design-system/dist`, which
  resolved to the monorepo root before extraction and now dangles two levels
  above this repo. **The package still exists in this workspace, renamed and
  re-orged: [`kotoba-lang/svelte-design-system`](https://github.com/kotoba-lang/svelte-design-system)
  is `@etzhayyim/design-system@0.1.0` and exports `./plugin`.**
- **`@etzhayyim/vite-plugin-safe-builder` is referenced nowhere** in the tree —
  declared but unused.

[`docs/operator-quickstart.md`](docs/operator-quickstart.md) walks a build that
was actually run to completion (28.30 kB bundle, served 200) by pointing the
dependency at that repo. That path is a **local probe, not a committed fix** —
which of the available forms the durable fix should take is an owner decision
and is recorded there as an open question, not silently chosen here.

## Verifying this README

Both checkers use three-valued exit codes on purpose: "could not measure" is
never reachable from the same exit code as "measured, all fine".

```bash
nbb docs/check-migration-identity.cljs   # 0 intact · 1 tampered · 3 cannot answer
nbb docs/check-surface.cljs              # 0 all resolve · 1 some NXDOMAIN · 3 cannot answer
```

On 2026-08-13 the first exits 0 and the second exits 1. Both were also observed
failing: editing a migrated file drives the first to 1, running either outside a
checkout drives them to 3. A checker that has only ever been seen passing has
not been shown to discriminate.

`check-migration-identity.cljs` reports files added after the extraction without
failing — this README, `docs/`, and the two checkers are such additions, so
expect it to name them.
