# Ziguanite

## A plan for preparing the plan

Ziguanite is the Ruby-Zig fork being established for one difficult portability
job and one uncompromising implementation rule:

> A supported CRuby or mruby target must not be excluded merely because its
> build requires Rust or because a native subsystem has made an unnecessary
> architecture assumption.

> Ruby stays Ruby.  Everything implemented outside Ruby converges to
> idiomatic Zig.

Zig is the pinned native toolchain for this work.  That sentence is not a
claim that one compiler can make every Ruby configuration work everywhere.
It is a commitment to find the real boundaries, make them explicit, and move
them only with evidence.

This document is deliberately a charter for research, not a grand technical
roadmap.  Small migrations begin immediately; the broad roadmap comes after
the questions below have answers and the answers have receipts.

## The language convergence rule

Ziguanite has two destination implementation languages: Ruby and Zig.  C,
C++, Rust, assembly, shell, Make, Autoconf, M4, and any other executable
implementation language in the fork are migration debt.  Documentation,
declarative configuration, vendored test fixtures, and data formats are not
implementation-language exceptions; any code that generates or interprets
them must still converge to Ruby or Zig.

Using `zig cc` is the bootstrap, not the rewrite.  A C function compiled by
Zig remains an unmigrated C function.  The compiler route gives the branch a
portable starting point while behavior moves into Zig one function at a time,
then one file at a time, then one subsystem at a time.

Each migration starts at the smallest independently testable seam.  Preserve
the observable Ruby behavior, keep a narrow C ABI only while callers still
need it, compare the old and new implementations when practical, and remove
the superseded implementation only after the focused receipt passes.  A
change that cannot state its rollback point and proof boundary is too large.

This rule is directional, not a claim that the present tree is already pure.
Every remaining non-Ruby/non-Zig implementation is explicit debt; none is
grandfathered in as the permanent architecture merely because it works today.

## The line in the sand

For every Ziguanite-supported profile:

* Building CRuby and mruby must not require `rustc`, Cargo, or an accidental
  host C compiler.
* Removing Rust must not silently mean removing useful behavior.  A
  Rust-backed feature ends in one of three stated places: a tested replacement,
  an owner-approved retirement with its compatibility cost written down, or an
  explicitly experimental add-on outside the canonical distribution.
* A result called "supported" says which bar it crossed: `compiles`, `links`,
  `boots`, `selected tests pass`, or `JIT available`.  Those are not synonyms.
* Cross compilation is never presented as execution proof.

The fork may acquire other goals.  It may not weaken those four statements to
make a dashboard turn green.

## What must be learned before choosing an implementation

### Build and dependency map

Freeze exact Ruby and mruby source references.  For each, inventory every
compiler command, generated source, bundled extension, build-time executable,
Rust input, assembly path, configure probe, and target-specific conditional.
The inventory records both the source location and why it exists; a list of
filenames is not enough.

The output is a checked-in, reviewable map with a source digest.  It separates
required semantics from optional accelerators and from historical build
convenience.

The map begins with a checked-in scope registry.  It names every Ruby-Zig
repository and build-support tree that invokes native tooling, says why it is
in scope, and records any explicit exclusion.  "We did not know it compiled
C" is not an acceptable exclusion once the registry exists.

### Target cards

Do not begin with a giant architecture matrix.  Agree on three to five target
cards, each naming:

* CPU, operating system, libc or ABI, and the exact Zig target spelling;
* whether the desired product is a CLI, shared libruby, static library,
  embedded mruby, or something narrower;
* how it can execute, if it can: native hardware, emulator, QEMU, or a
  runtime; and
* the first honest success bar.

One card should be a familiar native baseline.  One should be an architecture
whose usefulness is blocked today.  mruby gets its own cards; it has a real
host/target split and should not be treated as a smaller CRuby.

### Receipt harness

Before runtime surgery, create a reproducible build receipt for every target
card.  A receipt binds the source ref and object, target triple, sysroot or
libc choice, Zig and wrapper digests, configure recipe, build profile, command
trace, produced artifacts, test tier, elapsed time, and failure excerpt.

It must be possible to distinguish a clean new build from a cache restore and
to repeat either intentionally.  Receipts are the substrate for later
performance claims and for deciding whether a portability change is real.

During the migration, every C, C++, preprocessor, assembler, linker, and
archiver invocation in a supported profile must resolve through pinned Zig
wrappers: `zig cc`, `zig c++`, and `zig cc -E` as appropriate.  The trace,
configured paths, and compiler fence make that assertion inspectable.  A
receipt can call a hosted run *fenced*; it can call a run *Zig-only* only when
both the compiler fence and the source inventory prove that no legacy
implementation language participated in the artifact.

### Fast-start runner study

Speed is a design question, not a promise.  Measure a cold build, a warm cache
restore, a seed-to-clone start, and a destroyed-job reset for the same source,
target, and receipt.  The first comparison is a sealed compiler-free Incus
seed with fresh work volumes on an approved copy-on-write storage pool (ZFS
when it is the selected pool).  It is simple, portable within a host
architecture, and does not pretend to clone live process memory.

CRIU-like restore is a hypothesis to measure, not a fleet design.  It carries
process, file-descriptor, socket, mount, and provenance risk.  Firecracker
full snapshots are a later same-architecture experiment with their own disk
and identity rules.  Adopt either only when p50 and p95 start-to-result
receipts beat the Incus lane without weakening isolation or repeatability.

### Compatibility map

For YJIT, ZJIT, MMTk, coroutine selection, executable-memory handling, atomic
paths, and native extensions, make a small contract card:

* public behavior and user-visible switches;
* C-facing and internal ABI boundaries;
* architecture assumptions and their reason;
* test coverage and benchmark coverage; and
* the cost of retaining, replacing, or omitting the component for a profile.

The current Rust/JIT paths are deep VM integrations, not ordinary libraries
waiting for a mechanical rewrite.  The map comes before choosing C, Zig behind
a C ABI, a portable backend, or a deliberate non-JIT profile.

## Grilling sessions

Each session is owner-facing and produces a short decision record.  A missing
answer is an open gate, not permission to guess.  No grand plan is written
until the owner has reviewed the records and explicitly approved proceeding.

### Session one: the promise

Answer these first:

* Does "no Rust" cover YJIT, ZJIT, and experimental MMTk, or only build-time
  Rust for the default interpreter profile?
* Is an interpreter-only CRuby a useful supported tier while a JIT replacement
  is researched, or merely a bootstrap milestone?
* Must every upstream feature remain available on every target, or can target
  profiles define explicitly unavailable accelerators?

This session establishes what a successful fork means and prevents an
accidental feature deletion from masquerading as portability.

### Session two: the first machines

Choose the first target cards and rank them.  Useful candidates include
`riscv64-linux-gnu`, `riscv64-linux-musl`, `loongarch64-linux-gnu`, armv7,
Windows arm64, WASI, and embedded mruby profiles, but none is selected by
default.

For each choice, decide whether compile-only evidence is acceptable at first
and name the eventual execution environment.  This is also where static
linking, W^X policy, libc expectations, and extension ABI requirements become
explicit rather than later surprises.

### Session three: behavior and speed

Choose the Ruby behavior that must remain compatible with YJIT or ZJIT:
switches, statistics, warm-up expectations, memory limits, observability, and
failure behavior.  Choose one benchmark suite and representative applications
before discussing a replacement's performance.

The session also answers whether a temporarily slower but portable JIT is
acceptable, and whether one canonical JIT is preferable to preserving two
independent implementations.

### Session four: stewardship

Choose the upstream relationship: a merge train, rebase train, or patch queue;
the release/backport policy; target support lifetime; attribution rules; and
the point at which work becomes public beyond this fork.  This session also
chooses the runner-substrate decision path: Incus first, a Firecracker
comparison later, or an explicitly deferred benchmark.

No upstream pull request is part of this charter.  Candidate patch series and
evidence may be prepared, but an upstream proposal requires a separate human
approval.

## The grand plan that follows

Once the four sessions and the evidence cards exist, the grand plan must have
these independent workstreams:

1. **Portable build substrate.** Route every native tool choice through pinned
   Zig wrappers; prove the route with compiler traces and receipt checks.
2. **mruby portability lane.** Preserve host `mrbc` versus target artifact
   separation, exercise `MRuby::CrossBuild`, and target focused gembox/HAL
   profiles before promising broad embedded support.
3. **Rust and JIT decision.** Compare at least one bounded replacement spike
   against the compatibility map.  Select an implementation route only after
   correctness, maintenance, and performance criteria are written down.
4. **Architecture-gate audit.** Classify every discovered gate as a required
   blocker, correctness-sensitive fast path, optional acceleration, or
   platform integration.  Preserve the behavior of good existing C and
   assembly portability work while replacing its implementation in small Zig
   slices; working legacy code may define the oracle, but not the destination.
5. **Validation and maintenance.** Build immutable release and maintenance
   inputs first, keep fork work behind them in the queue, retain concise
   receipts, and publish no conclusion that exceeds the tested tier.

Every workstream has a rollback point, a success criterion, a target card, and
an upstream-distance statement: useful only in Ziguanite, potentially
upstreamable, or not yet suitable for either conclusion.

## Release ledger

Every official Ruby release and selected maintenance head receives a ledger
entry: immutable source ref, target card, latest receipt, and either a result
or a specific incompatibility record.  New upstream release tags enter the
same queue through an immutable discovery record.  No version-like tag may be
silently skipped; it is classified as official, pre-release, unclassified, or
unrelated before scheduling.

For historical Ruby, a tag name is not release truth.  The ledger is therefore
artifact-centric: each official source archive has a canonical release ID,
chosen archive URL and digest, source-root name, and any verified Git
equivalence.  Tags are recorded as provenance, aliases, anomalies, or explicit
non-buildable records.  An archive with no matching tag is still a release;
a tag that points at the wrong source is never allowed to impersonate one.
Alternate compression formats normalize to one chosen artifact per release.

Upstream releases and maintenance heads take precedence over fork work.  Equal
source, target, and recipe combinations may share one receipt with explicit
tag aliases, but a missing ledger entry is never inferred to be green.

## Fork hygiene

`master` is the synchronized upstream mirror and is not a human work branch.
A future synchronizer may fast-forward only `master` and a separately named
managed `mirror/*` branch namespace after the applicable approval gate.  It must
never rewrite a human branch, delete a tag, manufacture a pull request, or
blur an upstream tag change into normal maintenance.

Human work lives on clean `ziguanite/*` branches.  Each branch records whether
its base is upstream `master`, a named maintenance branch, or a release tag,
with the exact source object in its first receipt or decision record.  That
makes a backport a first-class branch choice instead of a misleading special
case of `master`.

The build queue gives upstream releases and maintenance heads first claim on
the small public budget.  Ziguanite branches use the same receipts but do not
displace them unless a measured private lane is demonstrably faster.

## First research cycle

1. Record the exact baseline ref, the language-debt inventory, and the branch
   rule in the first receipt.
2. Move one smallest independently testable C function into Zig and retain its
   old behavior as the oracle or rollback point.
3. Hold sessions one and two; write their decision records.
4. Implement the receipt harness for one native CRuby and one mruby cross
   profile without modifying runtime behavior.
5. Hold session three with the first receipts in hand.
6. Hold session four, review all four decision records, and obtain explicit
   owner approval to draft the grand plan.
7. Write the grand plan, including its target matrix, JIT decision criteria,
   synchronization policy, runner decision path, and test ladder.
8. Ask for explicit approval before turning that plan into broad parallel
   runtime work, remote CI expansion, or any upstream-facing change.  The
   already-authorized one-seam-at-a-time migration continues underneath that
   gate.

MINASWAN.
