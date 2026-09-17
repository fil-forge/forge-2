# Monorepo TODO

Two kinds of thing the consolidation turns up and cannot deal with where it
finds them.

**Questions for the whole repository**, below, which only make sense once the
monorepo is built and can be looked at as a whole. Each needs a team decision
rather than an implementation. Add to it when a branch turns up a question it
cannot answer by itself: say what was found, why it is not decidable yet, and
what someone would have to choose — not what you would do.

**Findings in the imported code**, at the end: bugs noticed while moving a
service in, which are not the migration's to fix. Add to it when a review of
an import turns up something that was already true upstream.

Neither is a place for ordinary work, which belongs in issues. What is
blocking or waiting on a person right now lives on the wiki's
**Needs Human Work** page.

---

# Questions for the whole repository

## Restore the s3-compat report pipeline

Upstream `ingot` publishes an S3 compatibility report to GitHub Pages
(`2365944c`, [ingot#131](https://github.com/fil-forge/ingot/pull/131)). The
consolidation brought the test suite across and left the reporting behind:
there is no s3compat step here, no `s3-compat-report` artifact, and no `pages`
job anywhere in the repository.

**Why it waits.** A report pipeline has to publish *somewhere*, and where that
is depends on decisions the monorepo has not made yet — whether services keep
per-service Pages sites or share one, and what the release flow looks like
once Phase 1 adds tags and publishing. Rebuilding ingot's version now would
bake in the polyrepo's answer to a question the monorepo gets to ask fresh.

**The choice.** One Pages site per service, one shared site, or drop the
published report and keep the suite's own output.

## Turn Renovate on

`renovate.json` is in the repository as of
[#6](https://github.com/fil-forge/forge-2/pull/6), covering the 18 images the
stack pulls plus the Go and Actions dependencies. It does nothing until the
Renovate GitHub App is installed on the organisation.

**Why it waits.** Installing it is an org-level action with org-level
consequences: it opens PRs across every repository it is granted, so the
scope, the schedule, and who triages the PRs are decisions for whoever owns
that surface — not for the branch that happened to write the config.

**Until then**, the pins are frozen rather than maintained. That is a real
cost and a deliberate one: a frozen pin is still reproducible, which an
unpinned tag is not, and #6 existed because a mutable `:main` tag broke a
conformance test.

**The choice.** Install it and decide the scope, or adopt something else
(Dependabot covers Docker and Go but reads `docker-compose` files less
willingly), or keep bumping by hand and accept the drift.

## Narrow hilt's Docker build context again

`hilt/Dockerfile` builds from the repository root, because hilt links swarf
through `replace => ../swarf` and Go resolves every replace target before it
downloads anything — a context scoped to `hilt/` dies in `go mod download`.
piri and ingot are the same. Recorded in the Dockerfile itself.

**Why it waits.** Narrowing it is not a Dockerfile change. An in-repo module
reached by a `replace` always lives outside `hilt/`, so no restructuring
helps — not even extracting an `internal/client/swarf`, which would just be a
different sibling directory. The only thing that narrows the context is
consuming swarf as a published, tagged module through the proxy, which is
Phase 1 work and gives up same-commit co-development in exchange.

**Agreed 2026-09-16**: keep it as it is for now, resolve before the
consolidation is finished. So this one has a decision already; what it needs
is Phase 1 to happen.

## Decide whether to cover smelt/systems/stress-tester

It is a module in its own right — `smelt/systems/stress-tester/go.mod`, with
its own `Dockerfile`, `compose.yml` and `cmd/` — and nothing builds, vets or
tests it. smelt's own `go build ./...` does not reach it because it is a
separate module, it is absent from `go.work`, and no workflow references it.
It is the only Go in this repository in that position.

**Why it waits.** It is not a check that was dropped: upstream's `go-check`
walked every `go.mod` in a repository, so this module lost its coverage at the
consolidation, when the root workflow started enumerating modules by name.
[#9](https://github.com/fil-forge/forge-2/pull/9) briefly added it and it was
taken back out, because restoring what was removed and covering something that
was never covered are two different changes.

It is also not free. Its `go.mod` says `go 1.24.0` where every other module
says `1.27.0`, which is below staticcheck's own minimum — adding it to the
matrix is what turned #9 red, and the fix had to move the toolchain source
from the job's module to `go.work`.

**The choice.** Add it to the matrix and to `go.work` and keep it building —
noting that adding it to `go.work` shifts workspace-mode resolution for every
other module (measured: `modernc.org/cc/v3` 3.40.0 → 3.41.0, `modernc.org/ccgo/v3`
3.16.13 → 3.17.0, plus a gorm and sqlite subtree) — or decide it is a
development tool that does not need CI, and say so where someone will find it.

It is clean on build, vet, tidy, test and staticcheck as it stands, so
whichever way this goes, it is not currently broken.

## Decide what to do about the macOS test run

Each service's own CI ran its tests on macOS as well as ubuntu: the shared
go-test workflow defaults to `["ubuntu", "windows", "macos"]` and every
service's config skipped only Windows. The monorepo runs ubuntu only, and
[#9](https://github.com/fil-forge/forge-2/pull/9) restored the other four
lost checks while deliberately leaving this one alone.

**Why it waits.** It is not clear the macOS jobs were ever green. Several
modules' tests boot containers through testcontainers, and GitHub's macOS
runners have no Docker daemon — so either those jobs were failing upstream,
or something not visible from here supplied one. Reproducing a job that was
already red buys nothing, and macOS runners bill at a higher multiplier than
ubuntu, so this is not free to find out by trying.

**The choice.** Establish what those runs actually did — one look at a recent
`Go Test` run on any of the eight upstream repositories settles it — then
restore macOS, restore it only for the modules that do not need Docker, or
decide ubuntu-only is what the monorepo wants and say so.

## Turn `SA4006` back on

`staticcheck.conf` at the repository root says `checks = ["inherit",
"-SA4006"]`, added by [#10](https://github.com/fil-forge/forge-2/pull/10). It
is off everywhere, for one finding in one generated file.

**What fires.** `indexing-service/pkg/service/queryresult/json_gen.go:231`.
`dag-json-gen` emits `written++` after each field and guards the *next* field
with `if written > 0 { WriteComma() }`, so the increment after the last field
has no reader. A true positive, and the JSON is correct.

**Why it waits.** Three ways out, none of them this branch's:

- **Fix the generator.** The durable answer, and it is upstream:
  `github.com/alanshaw/dag-json-gen`, pinned at `v0.0.9`, which is also the
  latest published version — so there is no newer release to take instead.
  Someone has to open that PR.
- **Pin staticcheck back to what the polyrepo ran.** Not available.
  indexing-service was `go 1.25.7`, so the shared workflow's version table gave
  it 2025.1.1 and its `Go Checks` at `bcb63ec` was green. Unifying the libforge
  pin moved it to `go 1.27.0`, and both older versions — 2025.1.1 (`v0.6.1`)
  and 2026.1 (`v0.7.0`) — fail on *every* package with `export data version 4
  is greater than maximum supported version 2`. Only 2026.2.1 reads Go 1.27
  export data, and the libforge pin requires `go >= 1.27.0`. Verified by
  installing both.
- **Narrow the suppression instead of widening it.** staticcheck's config is
  directory-scoped and walks up, so a `staticcheck.conf` in
  `indexing-service/pkg/service/queryresult/` would cover one directory rather
  than thirteen modules. That is where it started, and it was moved out: that
  directory is inside a subtree prefix, so a file there is permanent local
  divergence every future `git subtree pull` of indexing-service has to carry,
  and it would still cover the three hand-written files beside `json_gen.go`.
  A narrower scope for a permanent conflict is a real trade, not an obvious
  one.

**The cost of leaving it.** SA4006 fires exactly once across thirteen modules
today, so nothing is lost yet. What is lost is future findings: a genuine dead
assignment written tomorrow, anywhere in the repository, goes unreported. The
longer this stays, the less true "nothing is lost" gets.

**The choice.** Fix `dag-json-gen` upstream and drop the conf; or narrow it to
the one directory and accept the subtree divergence; or decide SA4006 is not
worth carrying and say so here rather than leaving the conf reading as
temporary.

## Decide whether CI should run only what a change affects

Nothing in `.github/` filters by path: no `paths:`, no `paths-ignore:`, no
changed-files detection. Every push runs all four workflows and every job in
them. `ci.yml`'s own header says why:

> Unfiltered by path on purpose: one job per module and no filter list means a
> new shared module cannot fall out of one and go silently green.

**What it costs**, measured rather than estimated. [#8](https://github.com/fil-forge/forge-2/pull/8)
changed **one Markdown file**. Its last push still ran:

| workflow | wall clock |
|---|---|
| `images` | 7m00s |
| `ci` | 8m40s |
| `e2e` | 9m57s |
| `itest` | **26m27s** |

26 minutes and 22 jobs for a documentation edit, and that repeats on every
push to every branch.

**Why it waits.** The obvious mechanism is the wrong one, in two separate
ways:

- **A hand-written `paths:` list is the same silent-green shape this
  consolidation keeps deleting.** The itest suites compiled behind a build tag
  and discarded, `SMELT_WORKSPACE` masking broken Dockerfiles, the indexer
  pulled from a published digest — each was green because something was not
  looked at. A filter list that goes stale when a module gains a dependency
  fails the same way, and looks identical while doing it.
- **A skipped job never satisfies a required status check.** GitHub reports a
  path-filtered job as *skipped*, not *success*, so any such job that is also
  required blocks the merge permanently. The usual answer is to start the job
  always and exit early inside it — which means the useful version of this
  filters *what a job does*, not *whether it runs*.

**The shape a good answer probably has**, if it is worth doing: derive the
affected set from the module graph rather than typing it. Rule 3 already says
this — `go list -deps` on the build target, not `go.mod` — and it is how the
`ingot → indexing-service` and `delegator → forgectl` edges were found, both
of which `go.mod` alone did not show. A derived set cannot go stale when
someone adds an import; a typed list silently can.

**The cheap part is already done.** `ci.yml` had no `concurrency:` block while
the other three did, so its runs piled up on a re-push instead of cancelling;
that was fixed separately and is not what this entry is about.

**The choice.** Build the derived gate, accept the cost as the price of not
having a stale filter list, or find a third thing — perhaps splitting the
slowest suites onto a different trigger.

## Decide how much more to spend making `itest ingot` fast

Sharding is done and **measured**, and it did less than predicted: ~25% off the
wall clock, not the "roughly half" this entry first claimed. The measurement
also moved where the remaining time is, so read the numbers before the options.

### The A/B, run within four minutes of each other

Both on `ubuntu-24.04`, same runner pool, same hour, so this is a real
comparison rather than a comparison against last night:

| | unsharded ([#19](https://github.com/fil-forge/forge-2/pull/19), run 35239716032) | sharded ([#21](https://github.com/fil-forge/forge-2/pull/21), run 35240022926) |
|---|---|---|
| `itest` workflow, start to finish | **30m43s** | **23m07s** |
| slowest job, start to finish | 29m29s (`itest ingot`) | 17m07s (`itest ingot 1/3`) |
| test binary | `ok …/ingot/itest` **1267.853s** | **628.841s** + **462.731s** + **204.654s** = 1296.226s |
| runner-minutes, ingot only | 29m29s | 42m04s |

**The total test work did not change** — 1296.2s against 1267.9s, +2.2%, which
is noise plus three process startups. Nothing got cheaper; it got spread. That
was the intent, and it is worth stating plainly because it bounds every option
below.

### Where the predicted half went

Three costs, none of which the "13 uniform boots" model had:

1. **The shards are not balanced: 629s / 463s / 205s.** The critical path is
   the *slowest* shard, not the mean. A perfectly balanced 3-way split of
   1296s would be 432s, so imbalance alone costs **~197s (3m17s)**.

   The round-robin splits by test *name order*, which silently assumes every
   test costs about the same. It does not. Shard 1 drew `TestForgeVersity` —
   the Versity S3-compatibility suite, hundreds of subtests, dozens of them
   3-second object-lock retention waits — plus `TestForgeMultipartExpiryShred`
   and `TestForgeReadAfterEviction`, which both wait on TTLs. Shard 3 drew four
   cheap ones and finished in 205s.

2. **Queue wait: 2m25s to 4m47s per shard** (4m47s on the critical one). Four
   concurrent jobs where there were two, and the runners did not all start at
   once. This is pure loss, and it grows with shard count.

3. **The image build, ~6 minutes, is per job and did not move.** Sharding
   triplicated it in runner-minutes.

### So the options have reordered again

- **Turn on a Docker layer cache in `itest` and `e2e`** — the cheapest thing on
  this list and, as far as the tree shows, never actually decided against.
  `itest.yml:143` and `e2e.yml:118` both shell out to plain
  `docker build`, which cannot use `--cache-from type=gha` at all; only
  `images.yml` uses buildx, via `docker/build-push-action@v6`. The
  "No caching, deliberately (2026-09-11)" comment lives in `images.yml` and
  gives a reason specific to *that* workflow — "the point is to provoke build
  failures, not to avoid them". That reason does not obviously carry: in
  `itest` and `e2e` the build is a means to running the tests, and `images.yml`
  is already proving the cold build works, on the same commit, on every pull
  request, in parallel. **Keeping `images.yml` uncached as the canary and
  caching the other two would preserve the whole point of the decision** and
  take up to ~6 min off each of four jobs. Needs
  `docker/setup-buildx-action` plus `buildx build --cache-from/--cache-to
  type=gha --load`; the risk is a stale cache masking a Dockerfile fault,
  which is exactly what the canary is for.
- **Build the images once and load them** — the same ~6 min, attacked from the
  other side: `images.yml` already builds all 8 and could hand them over as an
  artifact. Unmeasured, and the uncertainty is real: 8 images is likely 1–2 GB
  of round-trip. A registry would be faster but `images.yml` deliberately takes
  no `packages: write` so fork PRs work. **Try the cache first** — it is a
  handful of lines, needs no cross-job plumbing, and if it works this option
  stops mattering.
- **`lockWaitTime` in our own versitygw fork is 3 seconds, and it is
  self-imposed.** `tests/integration/utils.go:2654` in
  `github.com/fil-forge/versitygw` (pinned at
  `v0.0.0-20260914113944-a628e2cc628c`) sets

      lockWaitTime time.Duration = time.Second * 3

  and `cleanupLockedObjects` uses it twice: it sets
  `RetainUntilDate: time.Now().Add(lockWaitTime)` and then sleeps the same
  duration waiting for the lock it just created to lapse. Nothing external
  requires 3 seconds — the teardown picks the retention date itself.

  **38 call sites** across `Access_Control.go`, `CopyObject.go`,
  `CreateMultipartUpload.go`, `GetObject*.go`, `PutObject*.go`,
  `WORM_protection.go` and `versioning.go`, so at one firing each that is
  **~114s of pure `time.Sleep`** — roughly a sixth of shard 1's 629s, and
  `TestForgeVersity` is the test that bounds the whole job.

  3s → 1s would save ~76s. **1s is the safe floor until someone checks**
  whether sub-second `RetainUntilDate` round-trips through ingot's object-lock
  path; the S3 header is a timestamp, and second granularity is the
  conservative assumption rather than a verified one. versitygw is a fork we
  control, so this is a one-line change — but it is **not** in this
  repository, so it lands upstream first and arrives here as a pin bump.
- **Balance the shards by measured duration**, worth ~3m17s. The obvious
  implementation is a hand-written grouping, which is the silent-green shape
  this repository keeps deleting — a new test falls out of the list unnoticed.
  A derived version works: carry the previous green run's per-test durations,
  bin-pack against them, and give an unknown test the mean so it still runs and
  merely unbalances slightly. That degrades safely, which the list does not.
- **One test bounds everything.** `TestForgeVersity` is most of shard 1's 629s.
  Until it is split or made internally parallel, no shard count gets the job
  below roughly its own duration plus the ~6 min build plus queue. More shards
  now buy very little.
- **Share one stack across tests** — the estimate this entry carried twice, now
  known to have been built on a wrong decomposition. The "~80s per boot,
  1040s booting / 217s working" split assumed 13 uniform boots. It cannot be
  right: shard 3 ran **four** tests, and so four boots, in 204.654s total, so a
  boot is at most ~51s and the real split is far more evenly divided between
  booting and working than assumed. **The shared stack is therefore worth less
  than the 3–4 minutes previously estimated, not more** — and it still changes
  test isolation in the one job that exists to catch flakiness. Parking it.

**The honest next step is to measure the artifact round-trip**, which is the
one number nobody has and the one lever that is clearly largest. Everything
else here is now measured.

**Also worth keeping in view:** the trade sharding actually made was **~43% more
runner-minutes for ~25% less wall clock**. That was the right trade while PRs
are the bottleneck; it is the wrong one if runner-minutes ever become the
constraint, and every service brought in-repo adds another image build to the
fixed half.


---

# Findings in the imported code

Problems noticed while bringing a service in, and deliberately not fixed by
the branch that found them: changing behaviour inside a commit whose job is to
move code makes a regression and a migration fault indistinguishable. Recorded
here so they are not lost with the review thread. They belong in issues
against the owning code once someone picks them up — nobody has filed them
yet.

Each was checked against the tree rather than taken on the reviewer's word.

## swarf

Raised by the Copilot reviewer on
[#3](https://github.com/fil-forge/forge-2/pull/3), and present at `c43af97`,
the commit the subtree imported — so none of these are migration damage.

### The revocation lookup is served as immutable for a year, and is not

`swarf/pkg/fx/app.go:348` sets `public, max-age=31536000, immutable` on the
by-delegation lookup. But the route is mutable: the schema permits several
rows per `revoked_delegation`, and `Get` returns the newest of them —
`swarf/pkg/store/postgres/store.go:83` is `ORDER BY recorded_at DESC, id DESC
LIMIT 1`. A cache may therefore serve a superseded record for a year, and
`immutable` tells it not even to revalidate.

On a revocation endpoint that fails in the wrong direction: a client holding
the cached older record sees a *narrower* revocation than the one actually
recorded.

It interacts with the next finding, and the two want deciding together. If
"newest matching" is the intended contract then the caching is wrong; if the
endpoint is meant to be immutable then it should be content-addressed by cause
CID, and `Get`-by-delegation is the wrong route shape.

### The memory and PostgreSQL stores disagree about what `Get` returns

`swarf/pkg/store/memory/store.go:64` keys records by the revoked delegation, so
a second revocation of the same delegation overwrites the first. PostgreSQL
keeps every row and returns the newest. The two backends therefore answer the
same question differently after a repeated revocation, and a memory-backed
stream cannot satisfy the interface's "all revocation records" contract.

The overwrite is the small part. The divergence is the real one: a test that
passes against the memory store can be wrong about production.

### The firehose client drops oversized events and then hangs

`swarf/pkg/client/client.go:259` builds a default `bufio.NewScanner`, which
caps a token at 64 KiB, and line 286 is `_ = scanner.Err()` under a comment
saying the caller reconnects. A valid event can exceed the cap, because
`api.FirehoseRevocation.Path` may carry many CIDs. `ErrTooLong` is then
discarded and `Stream` reconnects at the same cursor indefinitely, never
yielding the record and never returning an error — it presents as a hang
rather than a failure, which is the worse of the two.

**The fix already exists in this repository, in the wrong place.** The CLI
raises the limit and checks the error — `cmd/swarf/stream.go:79` is
`scanner.Buffer(make([]byte, 64*1024), 4*1024*1024)` and line 101 is
`if err := scanner.Err(); err != nil` — while the library every other service
consumes does neither.

### The stream's settle window assumes a bound on transaction duration

`swarf/pkg/store/postgres/store.go:22` defines `streamSettleWindow = 10 *
time.Second`, documented as bounding "how long an insert may take between its
`recorded_at` (NOW() at transaction start) and its row becoming visible".

That is an assumption, not a bound. PostgreSQL assigns `NOW()` at transaction
start, so an INSERT that blocks for longer commits with a `recorded_at`
already behind the cursor, and the stream misses that revocation permanently.
The code is aware of the shape of the problem — the comment at line 111
explains that rows can become visible out of `recorded_at` order — but ten
seconds is a guess at how far out of order. A monotonic database sequence
would make publication and cursor advancement independent of how long a
transaction took.
