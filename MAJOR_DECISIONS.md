# Major decisions

The choices a reviewer would otherwise have to reverse-engineer, and the ones
whose first impression is "that looks wrong". Each says what was decided, what
it costs, and what would make it worth revisiting.

Not a changelog and not a status page. What is blocking right now lives on the
wiki's **Needs Human Work**; questions deferred until the monorepo can be seen
whole live in `MONOREPO_TODO.md`.

---

## Each `itest/` suite is its own Go module

*Status: [#4](https://github.com/fil-forge/forge-2/pull/4), in review. Everything
below describes that PR, not `main` as it stands.*

The presenting problem was that `hilt/Dockerfile` needed the **repository root**
as its build context: `hilt/go.mod` carried `replace => ../smelt`, and Go
resolves every replace target before downloading anything, so a context scoped
to `hilt/` died in `go mod download` reading `../smelt/go.mod`.

The real problem was that nothing hilt's binary links has ever lived outside
`hilt/`. The smelt dependency belonged to `itest/` alone, and `//go:build itest`
was doing a module boundary's job — hiding files from the parent module's build.
A module boundary does that properly, so the tag is gone.

**What it cost.** A new module re-opens the version-skew question that unifying
the pins closed: a fresh `go mod tidy` in `ingot/itest` resolved `versitygw` to
a commit still declaring the upstream module path and picked a newer `libforge`
than the unified pin; `hilt/itest` picked a newer `ucantone`. All pinned back
by hand. Expect this every time a module is added.

**What it revealed.** Two of the five sibling `replace` directives were
test-only, not one: both `hilt → smelt` and `ingot → smelt` moved into the
nested `itest` modules. Three remain genuinely linked and keep the root build
context: `ingot → hilt`, `piri → delegator`, `piri → piri-signing-service`.

## Module paths say `forge`, not `forge-2`

Every module is `github.com/fil-forge/forge/<svc>`, though the repository is
currently `fil-forge/forge-2`. Deliberate: `forge-2` becomes `forge` by rename
or by pushing its `main` onto it, so the paths are already correct on the far
side of that.

**The cost is real but bounded.** An external `go get
github.com/fil-forge/forge/piri` resolves at the *old* repository and fails
there. Nothing we build is affected — per-module `GOWORK=off` builds, the
workspace build, and every `replace ../<svc>` edge are relative and never
consult the network for an in-repo module.

## Sibling services are wired with `replace ../<svc>`

Not with extracted `internal/client/<svc>` packages, which
`docs/consolidation/build-readiness.md` recommended. A `replace` is Go-native,
always resolves to the matching commit, and needs no new package designed
before anything can build.

Extraction is a real alternative and a better long-term shape — it is Phase 3's
kind of work (module boundaries), not a Phase 0 prerequisite. Revisit when
`libforge` dissolves and the boundaries are being drawn anyway.

## Six peer images stay on a mutable `:main` tag on purpose

`smelt/pkg/stack/options.go` pins eighteen images by digest and then, at lines
420–425, leaves six on `ghcr.io/fil-forge/<svc>:main` — `piri`, `hilt`,
`ingot`, `sprue`, `delegator`, `piri-signing-service`. This looks exactly like
a sweep that missed six entries. It is not.

Those six are the **published peers** an `itest` run tests against. Each suite
compiles the working tree's service, mounts it over that service's published
image, and takes every other service from the registry — HEAD-of-one-service
against the published network. Pinning them would convert a compatibility check
into a test against a frozen snapshot, which is the one thing it must not be.

**This has already bitten once, and the bite was the point.**
`TestForgeVersity/UploadPartCopy/invalid_part_number` went red here with
nothing in the tree changed: `hilt:main` had moved (#70, #72) while our ingot
sat at `a80bea0`, a pairing that never existed upstream. The fix was to resync
the subtrees, not to pin the tag. A mutable tag is how that class of skew
becomes visible at all.

## `-count=1` on every live-stack test invocation

`setup-go` restores `GOCACHE`, which holds Go's **test-result** cache. Per `go
help test`, a result is reused when the files and environment variables the
test consulted are unchanged — and a Docker stack is neither. The first green
`itest hilt` took 26 seconds and started no containers:

```
ok  github.com/fil-forge/forge/hilt/itest  (cached)
```

So `-count=1` is on the CI invocations *and* on the commands the docs tell
people to paste. The CI paths cannot produce a silent green build; the
documented ones can produce a silent green developer.

## MinIO comes from our own fork

`minio/minio` images were **deleted from Docker Hub** on 2026-09-11 and the
project archived; Quay is affected too. Pinning would not have helped — the
whole repository is gone, so every tag in it is gone.
[`fil-forge/minio`](https://github.com/fil-forge/minio) builds from source and
publishes `ghcr.io/fil-forge/minio:<upstream tag>`, currently
`RELEASE.2025-10-15T17-29-55Z`, which carries a security fix that was never
published as a container. See the wiki's **MinIO Image Removal**.

## The history is full of merges, and is meant to be

`main` carries ~1000 commits and ~60 merges because seven services' full
upstream histories are in it, imported by unsquashed `git subtree add`, and
each `git subtree pull` adds that service's new commits behind another merge.

**`--squash` is never used, on any prefix.** A rebase flattens imported history
after the fact; `--squash` declines to import it at all; both leave a monorepo
that cannot say where its code came from.

The consequence for contributors: when the base moves under a branch carrying
subtree merges, that branch is **rebuilt** — replayed onto the new base with
each `git subtree pull` re-run — not merged and not plainly rebased.
`git rebase --rebase-merges` does *not* do this; it recreates the topology but
re-runs a plain merge that knows nothing about the subtree prefix. The wiki's
**Current State**, rule 7, has the full mechanics and the sweep it requires.

## What is deliberately not here

- **Outward-facing libraries**, which have consumers beyond Forge and want
  their own versions and cadence: `ucantone`, `automobile`.
- **Forks of upstream software** we patch or repackage — `minio`,
  `storetheindex`, `did-method-plc`, `filecoin-localdev`, `versitygw`,
  `filecoin-services`. Folding these in would destroy what makes them useful:
  upstream history, provenance, and the ability to take upstream changes.
- **Things being retired**: `guppy` is being dismantled and archived rather
  than moved.

`indexing-service` and `forgectl` are in scope and pending. `swarf` is landing
via [#3](https://github.com/fil-forge/forge-2/pull/3) and loses its image pin
on arrival, like everything else that moves in.
