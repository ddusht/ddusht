# Agent-Native Image Building: Buildpacks, Bazel, and a Scion for Builds

Oct 20, 2026 · @ddusht

## Summary

Image building was designed for humans shipping a few releases a day; agents need thousands of short-lived environments in under a second. Cloud Native Buildpacks (CNB) already has the right data model for that world (semantic layers, SBOMs, rebase), but its execution model is batch, sequential and registry-centric.

The opportunity is a "Scion for builds": a shared, always-online build service that serves environments from a company-wide content-addressed cache, the way Anthropic's Scion turned source control into a service. Bazel and BuildBuddy prove the model for declared builds; nobody yet combines it with buildpacks' zero-config detection and governance.

## What Cloud Native Buildpacks are

CNB is a CNCF specification plus a reference implementation that turns source code into an OCI image without a Dockerfile. It grew out of Heroku's and Pivotal's buildpacks. The current lifecycle line, 0.21.x, supports Platform APIs up to 0.15 and Buildpack APIs up to 0.12.
 
The lifecycle runs five phases in order:

1. **Analyze**: read metadata from the previous image to see which layers already exist.
2. **Detect**: each buildpack's `/bin/detect` inspects the source and the lifecycle resolves a build plan of what each provides and requires.
3. **Restore**: bring cached layers back.
4. **Build**: each buildpack's `/bin/build` writes named layers flagged `build`, `launch` or `cache`.
5. **Export**: assemble run image + launch layers + app layer, then write SBOMs and metadata.

A *builder* bundles a build image, buildpacks and the lifecycle. Platforms that drive it include `pack`, kpack, Tekton, Heroku and Google Cloud (Cloud Run, App Engine, Cloud Functions). Heroku extended CNB to its Cedar generation in September 2026.

| Strengths | Weaknesses |
| --- | --- |
| Rebase: patch the OS under thousands of images without rebuilding | Slow first builds; phases run in sequence |
| Semantic layers: cache invalidation follows meaning, not Dockerfile line order | Writing a buildpack is heavy |
| Native SBOM and provenance | Custom system packages are awkward |
| Platform team owns the builder; app teams just push code | Registry-centric: build, push, pull, run |
| Unprivileged, daemonless builds | Produces slim run images, not dev environments |

## Why agents break CI-era assumptions

CNB was designed for one build per commit with a human release at the end; agent workloads invert almost every assumption.

| Dimension | Classic CI build | Agent workload |
| --- | --- | --- |
| Volume | About 1 build per commit | Hundreds to thousands of environments per hour, many forks of one repo |
| Latency budget | Minutes | Sub-second; a model loop or a person is waiting |
| Output | Slim run image | Build environment: compilers, package managers, test tools |
| Mutability | Immutable after build | Agent installs and edits at runtime; the image is a starting point |
| Trust | Trusted Dockerfile | Agent-authored, partly untrusted instructions and code |
| Error reader | Human | LLM that must self-repair |

Sandbox vendors now compete on start time, and the fastest get there by skipping image builds at request time: Daytona restores snapshots from warm pools, and Modal turns a prepared sandbox's filesystem into a reusable image. The image is becoming an environment snapshot, not a release artifact.

## Bottlenecks in agent image building today

The biggest single cost is moving bytes nobody uses: image download is about 76% of container startup, yet only 6.4% of the data is needed to start doing useful work.

1. **Pull and unpack dominate startup.** Lazy loading helps but has limits: SOCI found lazy loading wins below 80% access density and a parallel full pull wins above it. In Grab's tests eStargz pushed Airflow startup from 5 s to 25 s, while SOCI kept it at 5 s.
2. **Dependency installs repeated at scale.** Thousands of sandboxes run the same `npm install` or `pip install`; caches are per host, so fleet fan-out means misses and upstream rate limits.
3. **Thundering herd.** Fifty parallel forks of one repo trigger fifty identical builds; few systems dedupe in-flight builds by content hash.
4. **Non-determinism kills caching.** `apt-get update`, `latest` tags and copying the whole repo before installing deps invalidate every later layer.
5. **Wrong image shape.** CNB ships a slim run image, but agents need toolchain + deps + test tools: a third image type nobody models.
6. **Runtime drift is lost.** Packages an agent installs mid-session vanish or get captured as an opaque snapshot with no SBOM.
7. **Registry round-trips and sprawl.** Build, push, pull, boot for a 10-minute environment; millions of throwaway images to garbage-collect.
8. **Secure builds are slow builds.** Untrusted instructions need rootless or microVM builders; privileged BuildKit is unsafe for multi-tenant use.
9. **Errors are not machine-actionable.** Hundreds of interleaved log lines cost the agent tokens and turns, and it often fixes the wrong thing.

## Landscape

Railpack has the faster execution model, CNB the richer data model, and Bazel with BuildBuddy shows what a shared build service looks like when every step is declared.

| Tool | What it is | Plan known up front | Shared cache | Best at | Main cost |
| --- | --- | --- | --- | --- | --- |
| [Cloud Native Buildpacks](https://github.com/GoogleCloudPlatform/buildpacks) | CNCF spec; Paketo, Heroku, Google builders | No: buildpacks discover work as they run | Per builder or cache image | Rebase, SBOMs, governance, unprivileged builds | Sequential, slow cold builds |
| [Railpack](https://railpack.com/architecture/overview) | Railway's Nixpacks successor; Go, BuildKit LLB | Yes: a JSON build plan compiled to a BuildKit graph | BuildKit cache, shareable | Speed, small images, simple config | Single vendor; needs a BuildKit daemon; no rebase |
| Bazel | Google's build system (internal name Blaze) | Yes, per action, with declared inputs and outputs | Remote cache + remote execution | Incremental, hermetic monorepo builds | Everything must be declared; steep learning curve |
| [BuildBuddy](https://buildbuddy.io/docs/remote-runner-introduction) | Hosted backend for Bazel and Buck2 | Uses Bazel's plan | Action cache, RBE, warm Firecracker runners | Shared build factory with build visibility | Only helps already-declared builds |
| [Scion](https://anthropic.com/engineering/redesigning-source-control) | Anthropic's in-house version control | Not a builder; source control as a service | Daemon, regional cache, server; 99%+ hit rates | 30,000+ merges a day; API access without a checkout | Always-online; not released publicly |

**Railpack vs CNB.** Railpack emits a plan where each step declares its inputs, so BuildKit runs independent steps in parallel and skips cached ones. Railway reported 38% smaller Node and 77% smaller Python base images than Nixpacks. CNB wins on fleet patching (rebase), per-buildpack SBOMs, daemonless multi-tenant builds and a vendor-neutral ecosystem.

**How Google uses both.** Internally Google builds with Blaze. Externally it ships CNB-compatible buildpacks for Cloud Run, App Engine and Cloud Functions, because customers upload unprepared code. The buildpacks repo itself is built and tested with Bazel: Bazel compiles each Go buildpack to a tarball, assembles builder images and runs per-builder acceptance tests.

**Agent sandbox platforms** (E2B, Daytona, Modal, Vercel Sandbox, Blaxel) each have their own image or template builders and compete on cold start through snapshots and warm pools.

## Why CNB cannot run a parallel DAG

Parallelism requires the build to be described as data before it runs; Railpack's and Bazel's builds are data, CNB's are programs.

Four properties of the CNB spec force sequential execution:

1. **Ordering is semantic.** A later buildpack relies on an earlier one's layers and environment being present, e.g. `npm install` needs Node already on `PATH`.
2. **Inputs and outputs are unknown in advance.** The lifecycle learns what a buildpack read and wrote only after `/bin/build` exits.
3. **Cache decisions live inside each buildpack.** Each compares its own layer metadata, so the engine cannot skip or reorder steps.
4. **Shared mutable filesystem.** All buildpacks write into one `/layers` tree, so running two at once would race.

This is a deliberate tradeoff: CNB optimized for buildpacks from many vendors composing without coordination; Railpack controls all its providers, so it can require declared inputs.

In kitchen terms, Railpack is a head chef with the full recipe written down, assigning independent tasks to different cooks at once. CNB is a relay of guest chefs in one small kitchen, each discovering what to do as they go.

**Paths to partial parallelism:** concurrent layer fetch and push inside phases; opt-in declared dependencies in a future Buildpack API; engine-level cache keys (lockfile hash, version) so the lifecycle can skip a buildpack without running it; and compiling layer declarations to BuildKit LLB.

## Proposal: a Scion for CNB

Turn builds into a shared, always-online service that answers "give me an environment for this code" from a company-wide cache, keeping CNB's semantic layers, SBOMs and rebase. Fittingly, a scion is the cutting grafted onto a rootstock, which is what rebase already does to app layers.

| Scion idea for source | Equivalent for builds |
| --- | --- |
| Repo lives on the server, not in every clone | Builds run in a shared service, not `pack build` on every runner |
| Diffs, merges, rebases happen server-side | Image assembly is a server-side metadata operation over layer digests |
| Every query is an API; no checkout needed | "Which images contain log4j 2.14?" answered without pulling anything |
| Lazy file fetch with a bounded local cache | Lazy layer fetch (SOCI-style): only bytes a process touches |
| Daemon, regional cache, authoritative server | Node daemon, regional layer cache, content-addressed layer store |
| No packfiles to maintain | Virtual images, materialized only when shipped to production |
| Directory-level permissions | Layer-level policy: which buildpacks and dependencies are allowed |

&#91;embedded content: Scion for builds · 3 cache tiers\]

A request falls through daemon, regional cache and build server only on a miss; the server reads just the files it needs from the source control service.

### Mechanics

1. **Every layer is a pure function of declared inputs**, e.g. JDK layer = Zulu 21.0.5 + buildpack v12. A layer built anywhere is reusable everywhere.
2. **Identical requests are deduplicated**: 50 agents asking for the same environment trigger one build.
3. **Rebase generalizes to any layer**: swapping a layer nothing depends on is a metadata operation.
4. **Memory snapshots sit on top** as a cache tier keyed to image digest; the image stays the auditable source of truth.
5. **Agents author the build plan** directly as a validated `plan.toml`, skipping heuristic detection.
6. **Structured results**: per-phase JSON with the failing buildpack, command, exit code and a remediation class.
7. **Runtime drift is captured**: an agent's `pip install` is routed through the buildpack, so it becomes a layer with an SBOM entry.

### Tradeoffs

- **Centralization**: if the service is down, nobody builds. Acceptable when users are always-online agents.
- **Hermetic buildpacks required**: declared inputs, no surprise network fetches; a large migration for Paketo and Heroku buildpacks.
- **Pays off only at scale**: overkill for a few builds a day, essential at tens of thousands.

Closest prior art: Bazel remote execution (Blaze/Forge internally, BuildBuddy and EngFlow commercially), Nix binary caches and Depot. None pairs that service model with buildpacks' zero-config detection.

## Practical note: Paketo Java reinstalling the JDK and Maven

When Paketo's Azul Zulu and Maven buildpacks reinstall every build, the cache is almost always not surviving between builds. Check the log: `Reusing cached layer` means the cache worked; `Contributing to layer` means a reinstall. `BP_LOG_LEVEL=DEBUG` shows why a layer was rejected.

| Cause | Fix |
| --- | --- |
| Ephemeral CI runners lose pack's local cache volume | `--publish --cache-image registry/myapp-cache`, or a cache PVC on kpack and Tekton |
| Image name changes every build (`myapp:${GIT_SHA}`) | Build to a stable name, add tags after, or use `--cache-image` |
| Previous image missing, so the JRE launch layer cannot be reused | Keep the last image in the daemon or registry under the same name |
| Builder pulled fresh, bringing a new JDK patch | Pin `BP_JVM_VERSION=21`, pin the builder digest, `--pull-policy if-not-present` |
| `--clear-cache` set in a CI template | Remove it |

To speed up reinstalls that do happen, point `BP_DEPENDENCY_MIRROR` at an internal Artifactory or Nexus, and bind a Maven `settings.xml` that uses the internal repository.

## Sources

- [Redesigning source control at Anthropic](https://anthropic.com/engineering/redesigning-source-control)
- [Simon Willison on Scion](https://simonwillison.net/2026/oct/13/redesigning-source-control-at-anthropic/)
- [Buildpacks lifecycle](https://github.com/buildpacks/lifecycle/wiki)
- [Heroku: CNB for Cedar-generation apps](https://devcenter.heroku.com/changelog-items/3803)
- [Railpack architecture overview](https://railpack.com/architecture/overview)
- [Railway: moving on from Nix](https://dev.to/sarah-railway/why-were-moving-on-from-nix-3cb1)
- [GoogleCloudPlatform/buildpacks](https://github.com/GoogleCloudPlatform/buildpacks) and its [CONTRIBUTING.md](https://github.com/GoogleCloudPlatform/buildpacks/blob/main/CONTRIBUTING.md)
- [BuildBuddy remote runners](https://buildbuddy.io/docs/remote-runner-introduction)
- [SOCI snapshotter](https://github.com/replit/soci-snapshotter) and [SOCI paper](https://arxiv.org/abs/2607.06868)
- [Grab: Docker lazy loading](https://engineering.grab.com/docker-lazy-loading)
- [LogRocket: comparing AI agent sandbox platforms](https://blog.logrocket.com/comparing-ai-agent-sandbox-platforms-e2b-modal-daytona-and-more/)
