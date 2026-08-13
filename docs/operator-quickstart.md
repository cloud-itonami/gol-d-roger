# Operator quickstart

Every command below was run to completion on 2026-08-13 and the observed output
is recorded next to it. Where a step fails, it fails here too — that is the
finding, not an omission. Nothing in this file is aspirational.

Measured with: macOS (darwin 25.3.0), node v26.3.0, npm 11.16.0, pnpm 10.26.2,
`nbb` on the PATH.

```bash
git clone git@github.com:cloud-itonami/gol-d-roger && cd gol-d-roger
```

---

## 1. Is this repo still the extraction it claims to be? (≈1 s)

`migration.edn` asserts this tree is `60-apps/etzhayyim-project-gol-d-roger`
lifted verbatim out of `etzhayyim/root` — 14 tracked files, 27,274 bytes — plus
two files the extraction added. Nothing checked that until now.

```bash
nbb docs/check-migration-identity.cljs
```

Observed:

```
EXAMINED	16 files at root commit 8537862	MIGRATED	14	ALLOWED-ADDITIONS	2
tracked-files  recorded 14	measured 14	match
bytes          recorded 27274	measured 27274	match
migrated files changed since extraction: 0
Extracted subtree is intact and matches migration.edn.
```

Exit `0`. Files added *after* the extraction (this document, the README, the two
checkers) are listed and do not fail the check — later work is expected,
in-place editing of the lifted tree is not.

Exit `1` means a migrated file was changed, moved, or deleted. Confirmed by
appending one line to `appview/…/svelte/index.html`: the script named that exact
file and exited 1; `git checkout --` on it returned the exit to 0.

Exit `3` means it could not measure — no git, no `migration.edn`, no unique root
commit. Confirmed by running it against an empty directory:
`COULD NOT ANSWER: /tmp/… is not a git checkout. Refusing to report a pass.`

## 2. Do the hosts this repo names exist? (≈2 s, needs DNS)

```bash
nbb docs/check-surface.cljs
```

Observed — exit `1`:

```
SCANNED	13 files	DECLARED-HOSTS	5
control	registry.npmjs.org	resolves

   br8bojxp.etzhayyim.com  NXDOMAIN  <- CLAUDE.md, README.md, …
            etzhayyim.com  resolves  <- CLAUDE.md, README.md, …/kotodama.jsonld, …
gol-d-roger.etzhayyim.com  NXDOMAIN  <- CLAUDE.md, PROJECT.jsonld, …/kotodama.jsonld, …
  resources.etzhayyim.com  NXDOMAIN  <- PROJECT.jsonld, README.md, …
   wy2zvdvd.etzhayyim.com  NXDOMAIN  <- PROJECT.jsonld, README.md, …

4 of 5 declared hosts do not exist.
```

(The "declared in" column is abbreviated above. The script scans `.md`, so it
attributes each host to this file and the README as well — they name the hosts
in prose. The hosts originate in `CLAUDE.md`, `PROJECT.jsonld`, and
`kotodama.jsonld`.)

The control host is checked first: if `registry.npmjs.org` fails the script
exits `3` instead of reporting five NXDOMAINs, because on a machine with no DNS
those five results would mean nothing. Exit `3` also fires when zero hosts are
extracted — confirmed against an empty directory, so an empty scan cannot be
mistaken for a clean one.

**Consequence for step 3 onward: there is no deployed backend to talk to.** The
UI is a static scaffold; nothing here reaches a live service.

## 3. Build the UI as committed — this fails (≈30 s)

```bash
cd appview/etzhayyim-wasm-gol-d-roger-wy2zvdvd/svelte
npm install
```

Observed:

```
npm error code EUNSUPPORTEDPROTOCOL
npm error Unsupported URL Type "workspace:": workspace:*
```

`workspace:` is a pnpm protocol. Trying the package manager that implements it:

```bash
pnpm install
```

```
ERR_PNPM_WORKSPACE_PKG_NOT_FOUND  In : "@etzhayyim/vite-plugin-safe-builder@workspace:*"
is in the dependencies but no package named "@etzhayyim/vite-plugin-safe-builder"
is present in the workspace
Packages found in the workspace:
```

Zero packages found: the workspace root stayed behind in `etzhayyim/root` when
this subtree was extracted. Removing both `workspace:*` entries exposes a
*second, unrelated* fault:

```
npm error Found: vite@6.4.3
npm error Could not resolve dependency:
npm error peer vite@"^5.0.0" from @sveltejs/vite-plugin-svelte@4.0.4
```

So the scaffold has two faults that must both be fixed, and the second is not
caused by the extraction.

## 4. A build that does complete (≈2 min)

The real missing dependency is `@etzhayyim/design-system`, required by
`tailwind.config.js` (`import { etzhayyimUIKit } from '@etzhayyim/design-system/plugin'`).
It still exists — as `kotoba-lang/svelte-design-system`, whose `package.json`
is `@etzhayyim/design-system@0.1.0` with a `./plugin` export.

The other one, `@etzhayyim/vite-plugin-safe-builder`, is referenced nowhere in
the tree (`grep -rn safe-builder .` finds only the `package.json` line).

> **This section is a probe, not a committed fix.** It was run in `/tmp` copies;
> the repo is unchanged. See "Open question" below.

```bash
# a) build the design system from its own repo
git clone git@github.com:kotoba-lang/svelte-design-system /tmp/ds
cd /tmp/ds && npm install && npm run build     # svelte-package: src/lib -> dist
ls dist/plugin/tailwind.js                     # 10,307 bytes
```

```bash
# b) point the app at it, drop the unused dep, fix the peer range
cd <repo>/appview/etzhayyim-wasm-gol-d-roger-wy2zvdvd/svelte
#   "@etzhayyim/design-system": "file:/tmp/ds"
#   remove "@etzhayyim/vite-plugin-safe-builder"
#   "@sveltejs/vite-plugin-svelte": "^5.0.0"
npm install && npx vite build
```

Observed — exit `0`:

```
vite v6.4.3 building for production...
✓ 109 modules transformed.
dist/index.html                  0.42 kB │ gzip:  0.28 kB
dist/assets/index-C-zwCK5o.css   0.24 kB │ gzip:  0.21 kB
dist/assets/index-D82sGWgq.js   28.30 kB │ gzip: 10.90 kB
✓ built in 1.23s
```

Served and confirmed reachable:

```bash
npx http-server dist -p 8791 --silent &
curl -s -o /dev/null -w '%{http_code} %{size_download}B\n' http://127.0.0.1:8791/
# 200 419B
```

The bundle contains the app's own text (`etzhayyim-wasm-gol-d-roger-wy2zvdvd`),
so this is the scaffold and not an empty shell. What renders is one heading and
one sentence — the design document's application does not exist.

**Note on shared-machine builds.** In this workspace, run heavy builds through
the resource governor, which serialises them to one at a time:

```bash
node scripts/resource-guard.mjs run build -- npx vite build
```

It refused a build during this walkthrough (`build is already running (pid=…)`)
and the step succeeded on retry — that refusal is the guard working.

### Open question, deliberately left open

Three forms of the durable fix are available and they are not equivalent:
depend on `kotoba-lang/svelte-design-system` by git URL; publish it to a
registry and depend on a version; or restore a pnpm workspace spanning the
extracted repos. Choosing among them decides how every sibling repo extracted
from `etzhayyim/root` resolves its shared packages, which is broader than this
repo. It is recorded here rather than silently decided.

## 5. What you cannot do here

Not "not yet documented" — absent from the tree:

| Named in `CLAUDE.md` | State |
|---|---|
| `proto/etzhayyim/gol_d_roger/v1/gol_d_roger.proto` | No `proto/` directory. |
| `GoldQueryService`, `GoldCommandService` | No service implementation. |
| Six `gold_*` Arrow tables | No schema, no data. |
| XRPC to 12 ISIC/ISCO/entity actors | No client, and the hosts are NXDOMAIN (step 2). |
| Six Matrix rooms under `#gold-*` | No configuration beyond the prose table. |
| Tests | None. `npm run check` needs the failed install from step 3. |
