# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) policy for the `linked-fw` package fleet.

`default.json` is the whole policy. Every package repo carries a three-line `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>linked-fw/renovate-config"]
}
```

Renovate resolves `local>linked-fw/renovate-config` to **this repo's `default.json` on its default
branch**, at every run. So one edit here changes all 22 repos, the same way `linked-fw/.github@v1`
holds one workflow definition behind 22 thin callers. Do not pin a SHA in the stubs — that hands
back the 22-file edit this exists to avoid.

`default.json` is required as the filename: Renovate has deprecated `renovate.json` as a *preset*
filename. (`renovate.json` as a consuming repo's own config is current and correct.)

## Why this exists

Six packages' `package-lock.json` pinned `@_linked/core` far below what their range allowed — `xsd`
and `primitives` on **2.6.0** under a `^2.0.1` range. `pr.yml` runs `npm ci`, which installs the
lock and ignores the range, so those packages built against a stale core for months. Core 2.6.0
predates the arch-02 IRI scheme, so `xsd`'s CI emitted
`https://data.lincd.org/module/-_linked-xsd/shape/boolean` instead of
`https://linked.cm/shape/xsd/Boolean`: **a published package's shape IRIs depended on which core
the consumer happened to install.**

A stock Renovate would not have opened a single PR about it. Its normal update flow only fires when
a new version *escapes* the declared range, and `^2.0.1` contains 2.6.0 forever. The one control
that catches it is **`lockFileMaintenance`**, which regenerates the lock within the existing ranges
— and it is **disabled by default**. It is enabled here. That is the point of this repo.

## The policy in one screen

| | |
|---|---|
| **Schedule** | Weekly, Monday 00:00–06:00 `Europe/Amsterdam`. Renovate defaults to UTC, so naming the zone is what stops the window drifting with CET/CEST. |
| **Grouping** | All `@_linked/**` in one PR per repo. `react` + `react-dom` + their `@types` together. GitHub Actions together. Everything else per-package. |
| **Never grouped** | Majors (one breaking change must not block nineteen safe ones), `typescript` (a minor is a compiler change), security fixes. |
| **Automerged** | `@_linked/*` non-major on green CI, and `lockFileMaintenance`. |
| **Held for review** | Every external dependency, every major, `typescript`, GitHub Actions. |

The automerge split is the one asymmetry worth stating plainly: an `@_linked/*` update PR exists
*because someone deliberately released that version*, so a human is already upstream of it. An
external bump is a third party's change that nobody reviewed. Green CI means different things
either side of that line — several ontology packages have no tests at all (`require-tests`
defaults to `false` in `pr.yml` for exactly that reason), so a passing check there often means
"nothing ran".

Majors stay manual even for `@_linked`, because the incident's failure mode is invisible to CI: the
build passed, and only the emitted output was wrong.

## Scope

The 22 repos with a `package.json`, a `package-lock.json` and a `pr.yml` caller. Deliberately out:

- **`livekit`, `live-sessions`** — no contents on the default branch.
- **`app-template`** — has a `package.json` but **no lockfile and no `pr.yml`**, so nothing would
  verify a Renovate PR against it. Its ranges are already behind (`@_linked/core: ^2.12`), and
  every app CN generates starts from whatever they resolve to that day. Real problem, wrong tool:
  it needs a `pr.yml` caller and a committed lockfile first, after which it is an ordinary 23rd
  repo.
- **`.github`** — no `package.json`, so the npm manager finds nothing; it is worth onboarding for
  GitHub Actions digests alone, which is a separate call.
- **CN** — yarn, private, a different org and a different release model. When it moves to npm it
  becomes the *most* important consumer of the `linked framework` group rather than just another
  member, and it will want its own preset file beside this one (a wider schedule, and
  `rangeStrategy: bump`, since nothing consumes an application's `package.json`).

## Peer dependencies

The invariant, and the one thing to not break: **Renovate must resolve locks under the same peer
rules CI installs with.** Renovate regenerates `package-lock.json` in its own sandbox; if it
resolves strictly while CI installs with `--legacy-peer-deps`, its artifact step hits `ERESOLVE`
and — rather than failing loudly — opens a PR with `package.json` updated and the lock left
untouched. That is a PR that drifts the lock further: the original defect, produced by the tool
meant to prevent it.

Measured today, both sides resolve strictly: the shared `pr.yml` runs a plain `npm ci`, no caller
passes `--legacy-peer-deps`, and no repo commits an `.npmrc`. So the alignment is structural and
needs no config. **If `--legacy-peer-deps` ever comes back to `pr.yml`, add `npmrc` +
`npmrcMerge: true` here in the same commit**, or Renovate will start opening PRs whose locks it
silently failed to regenerate.

## Background

`docs/ideas/renovate-rollout.md` in the Create Now repo.
