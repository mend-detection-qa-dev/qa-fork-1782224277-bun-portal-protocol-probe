# bun-portal-protocol-probe

Mend SCA detection probe — Tier 4, entry #20.

## Pattern

S4 — `portal:` protocol (yarn-compat, symlinked, no-copy, no install hooks).

Source: `docs/BUN_COVERAGE_PLAN.md §3.2`, feature S4.

The `portal:` protocol is a yarn-compatible local-source declaration that Bun adopted for
cross-compatibility with yarn-based projects. It declares a dependency as a symlink into a local
directory, with no file copying and no install-hook execution. The protocol token `portal:` is
preserved verbatim in the manifest and in the `bun.lock` workspaces section.

## Why standalone

`portal:` has a distinct lockfile shape from `file:` and `link:`, and introduces a third
protocol token that Mend's parser must recognize as a local-source type. The failure modes
are qualitatively different from the `file:`/`link:` pair covered in probe #7:

- A parser that knows `file:` and `link:` but not `portal:` will drop `my-portal-pkg` entirely
  (most likely real-world behavior given Bun is not in the UA resolver list).
- A parser that maps `portal:` to `file:` will emit `editable=false` instead of `editable=true`
  (wrong — `portal:` is a symlink).
- A parser that maps `portal:` to `registry` will attempt an npm lookup and fail (wrong source
  classification is a hard failure for any dep that is not actually on the registry).

These failure modes are different enough from the `file:`/`link:` taxonomy to warrant a
standalone probe with its own ReportPortal step.

## Relationship to probe #7

Probe #7 (`bun-local-sources-probe`, Tier 2, entry #7) covers `file:` and `link:` as a bundled
pair. This probe extends the local-source protocol coverage with `portal:` as a third case:

| Protocol | Install behavior | `editable` | Probe |
|---|---|---|---|
| `file:` | copy into `node_modules` | `false` | #7 |
| `link:` | symlink into `node_modules` | `true` | #7 |
| `portal:` | symlink into `node_modules`, no install hooks | `true` | #20 (this probe) |

`portal:` and `link:` share the same `editable=true` flag and symlink behavior. The only
observable difference in the dep tree is the `source_detail.protocol` field (`"portal:"` vs
`"link:"`). A Mend parser that conflates the two will still produce a correct `editable` flag
but will lose protocol identity.

## Mend config

No `.whitesource` is emitted for this probe. Bun is NOT in Mend's `install-tool` supported list —
`scanSettings.versioning` cannot pin a Bun toolchain, and there is no UA pre-step toggle for Bun.
Detection is lockfile-driven only: Mend must parse `bun.lock` (text JSONC, Bun 1.2+ format)
statically without running `bun install`.

The UA `javascript.md` resolver file (fetched 2026-05-15) contains zero references to Bun or the
`portal:` protocol. The resolver selection table covers `yarn.lock`, `package-lock.json` /
`npm-shrinkwrap.json`, and `pnpm-lock.yaml` only. `bun.lock` is not listed. This is a documented
limitation (`BUN_COVERAGE_PLAN.md §4`, row "Bun not in install-tool list").

## Project layout

```
bun-portal-protocol-probe/
├── package.json                <- root: portal: dep declared
├── bun.lock                    <- JSONC lockfile (Bun 1.2+), portal: protocol preserved
├── index.ts                    <- minimal stub
├── vendor/
│   └── my-portal-pkg/          <- portal: package (symlinked on install, no hooks)
│       └── package.json        <- name: my-portal-pkg, version: 1.0.0, no deps
├── expected-tree.json
└── README.md
```

## Dep tree (expected)

```
bun-portal-protocol-probe@0.1.0
  my-portal-pkg@1.0.0   [source: local, editable: true, protocol: portal:]
    (no deps)
```

- Direct deps: 1 (`my-portal-pkg`)
- Transitive deps: 0
- Total: 1

## Protocol comparison (all local-source protocols)

| Protocol | Token in lockfile | Source | Editable | Install behavior |
|---|---|---|---|---|
| `file:` | `file:./path` | `local` | `false` | Bun copies the directory (like npm `file:`) |
| `link:` | `link:./path` | `local` | `true` | Bun symlinks (yarn-compat) |
| `portal:` | `portal:./path` | `local` | `true` | Bun symlinks, no install hooks (yarn-compat) |

## Key failure modes

| Failure | Symptom in Mend output | Mend risk |
|---|---|---|
| `portal:` not recognized | `my-portal-pkg` dropped from tree | High — Bun is not in UA resolver list |
| `portal:` classified as `file:` | `editable=false` instead of `true` | Medium — parser pattern-matches on colon suffix |
| `portal:` classified as `registry` | npm lookup attempted, resolution fails | Medium — naive parsers treat unknown protocols as registry |
| `portal:` classified as `link:` | `editable=true` correct, but `protocol` field wrong | Low — functionally acceptable but wrong metadata |
| `my-portal-pkg` in `workspaces` not read | dep dropped (packages-only parser) | High — same risk as `link:` and `file:` in probe #7 |
| Ghost transitive emitted | spurious dep in tree | Low — `my-portal-pkg` has no transitive deps |
| JSONC parse failure | empty tree | High — lockfile has trailing commas and comments |

## Resolver notes (UA — javascript.md)

The UA `javascript.md` resolver file (fetched live 2026-05-15) contains zero references to Bun
or the `portal:` protocol. The resolver selection table covers `yarn.lock`,
`package-lock.json` / `npm-shrinkwrap.json`, and `pnpm-lock.yaml` only. `bun.lock` is not listed.

The Yarn 1.x resolver DOES document `link:./path` syntax for local packages in `yarn.lock`
format. The `portal:` token is not mentioned in the UA resolver file — it is absent from
both the npm and yarn resolver sections.

Because this pattern targets something the resolver file does not mention, the comparator must
treat this probe as **exploratory** (not regression-bound against a documented Mend behavior).
A passing result indicates Mend added Bun support including portal: recognition.
A failing result documents the gap for a Jira bug report.

## Tracked in

`docs/BUN_COVERAGE_PLAN.md` §11.4 entry #20
