# Substrate Image Updates and Snapshot Semantics

> Date: 2026-07-07

## Question

What should happen when an actor is suspended and its `ActorTemplate` image changes? Related: what happens when the gVisor/runsc or worker runtime image changes underneath existing snapshots?

## Short Answer

A process/rootfs snapshot should be treated as bound to the compatibility set that existed when it was created: workload image/rootfs, pause image, runsc/gVisor behavior, OCI bundle layout, mount/runtime flags, and Substrate's atelet/ateom restore protocol. Updating the image is not a safe "resume into the new image" mechanism. It is a snapshot invalidation or migration boundary.

## Current Substrate Behavior

- **Actor state stores a template reference, not a frozen template spec.** The public `Actor` proto stores `actor_template_namespace`, `actor_template_name`, and snapshot URI fields, but not resolved workload image digest, pause image digest, template generation/hash, or runsc hash.
- **`CreateActor` records only the template reference.** It validates that the referenced `ActorTemplate` exists, then stores an actor in `SUSPENDED` state with the template namespace/name.
- **`ResumeActor` reloads the latest `ActorTemplate`.** The resume workflow fetches the actor, then fetches the current `ActorTemplate` by name.
- **Restore rebuilds OCI bundles from the current workload spec.** `atelet.Restore` downloads checkpoint files, calls `prepareOCIBundles`, then asks `ateom-gvisor` to run `runsc create` and `runsc restore`.
- **The CRD comments already encode the invariant.** Both `PauseImage` and `Container.Image` require pinned references and say: "All images must be pinned (changing the image invalidates snapshots)."
- **The `ActorTemplate` spec is now immutable.** The CRD validates `spec` with `self == oldSelf`, so an in-place `kubectl patch` that changes the image (or any other spec field) is rejected by the API server. This closes the most obvious way to create an old-checkpoint/new-rootfs mismatch. The remaining risk is deleting and recreating a template under the same name with a new image while snapshots created under the old image still exist at a reused snapshot location.

## GitHub Findings

There are no GitHub Discussions in [agent-substrate/substrate](https://github.com/agent-substrate/substrate) at the time of this search. Relevant public issues/PRs exist, though.

### Directly Relevant

- [Issue #10 — Prevent user from creating ActorTemplates with un-pinned images](https://github.com/agent-substrate/substrate/issues/10): explicitly states that if a tag points to a different image, it invalidates all gVisor snapshots based on the old image. Proposed practice: require pinned images and prevent ActorTemplate mutation.
- [Issue #16 — Some improvements to ActorTemplate](https://github.com/agent-substrate/substrate/issues/16): says updating an `ActorTemplate` makes the golden snapshot stale and causes actors using the template to diverge. Proposed direction: treat `ActorTemplate`s as conceptually immutable and enforce that.
- [Issue #148 — Mutating ActorTemplate doesn't result in regeneration](https://github.com/agent-substrate/substrate/issues/148): concrete repro where editing an image after the template is Ready increments Kubernetes generation but does not regenerate the golden actor/snapshot.
- [Issue #119 — Actor State Machine](https://github.com/agent-substrate/substrate/issues/119): distinguishes memory, rootfs, and homedir layers. It says OCI image updates invalidate process/rootfs snapshots, while homedir state can survive image changes. It also describes devolution: gVisor/hardware changes can force memory purge while retaining rootfs/homedir, and image upgrades can devolve to homedir only.
- [Issue #166 — Avoid re-untarring actor image rootfs on every restore](https://github.com/agent-substrate/substrate/issues/166): confirms the current restore path spends time rebuilding OCI rootfs before `runsc restore`. It proposes caching extracted rootfs by immutable image digest, reinforcing that image digest is the right compatibility key.

### Related Signals

- [PR #184 — expose actor identity via `/run/ate/actor-id`](https://github.com/agent-substrate/substrate/pull/184): explains why actor identity should not be injected via env var. Env vars live in checkpointed process memory/overlay, so restoring multiple actors from one golden snapshot would expose the golden actor's ID. This is a concrete example of "values captured in process/rootfs state are not late-bound unless mounted/recreated at restore time."
- [PR #150 — re-enable `-direct` and `-background` runsc flags](https://github.com/agent-substrate/substrate/pull/150): shows restore behavior depends on runsc flags and checkpoint file lifecycle; runtime configuration is part of compatibility.
- [PR #96 — GPU passthrough via gVisor nvproxy + cuda-checkpoint](https://github.com/agent-substrate/substrate/pull/96): states that NVIDIA driver version must match across checkpoint and restore, another example that restore is constrained by runtime/environment compatibility.
- [Issue #121 — pluggable ateom backend](https://github.com/agent-substrate/substrate/issues/121) and [Issue #123 — MicroVM support](https://github.com/agent-substrate/substrate/issues/123): both point toward checkpoint/restore behavior being backend-specific. The state policy should not assume all runtimes have the same compatibility boundary as gVisor.

## gVisor Checkpoint/Restore Mental Model

gVisor `runsc` restore is not "run old memory in an arbitrary new image." The raw flow is: create a new container from an OCI bundle, then run `runsc restore --image-path=<checkpoint-dir>`. That means restore still depends on a compatible `config.json`, rootfs, mounts, runtime flags, CPU features, and filesystem view.

gVisor documentation also separates checkpoint/restore from rootfs tar snapshots. If filesystem changes need to be carried across as a portable artifact, rootfs snapshotting is a separate mechanism. For Substrate, that maps well to issue #119's separation between memory, rootfs, and homedir layers.

## User-facing Practice

The simplest user-facing rule is: treat `ActorTemplate` as an immutable compatibility contract. If the workload image changes, create a new template revision instead of patching the existing template image in place.

### Development / throwaway actors

- Delete or abandon old actors and their snapshots.
- Delete/recreate the `ActorTemplate`, or create a new template name.
- Boot fresh actors from the new pinned image.

### Production without state migration

- Create a new template, for example `counter-v2`, with the new pinned image digest.
- Let the controller create a fresh golden snapshot for that template revision.
- Create new actors from the new template and cut traffic over at the application/routing layer.
- Drain, suspend, archive, or delete actors using the old template only after they are no longer needed.
- Keep old images/runtimes available until no actor snapshot depends on them.

### Production with state preservation

- Do not expect gVisor process/rootfs snapshots to migrate across image versions.
- Preserve state at an app-level durable layer: database, object store, homedir/app-data layer, or explicit export/import.
- Boot actors under the new template and restore/migrate that app-level state explicitly.
- If the same actor ID must survive, the platform needs an explicit upgrade/migration API; silently changing the template behind the actor is the wrong primitive.

Substrate now enforces `ActorTemplate` spec immutability at the CRD level (`spec` is validated with `self == oldSelf`), so a plain `kubectl patch actortemplate ... image=...` is rejected rather than silently producing an old-checkpoint/new-rootfs pairing. Changing the image requires creating a new template revision (a new template object). Resume still reloads the current `ActorTemplate` and rebuilds OCI bundles from it, so the residual risk is deleting and recreating a template under the same name with a different image — or reusing a snapshot location across revisions — which can still form an unsafe pair.

## Practical Policy Options

| Policy | Behavior | When it fits |
| --- | --- | --- |
| Reject on mismatch | Snapshot metadata is compared to current template/runtime metadata; mismatch fails clearly. | Safest first production behavior. Prevents silent corruption or confusing partial success. |
| Use recorded old compatibility set | Restore with the image digest/runsc version/template revision recorded when the snapshot was created. | Good for preserving suspended actors across template changes, but requires old images/runtimes to remain available. |
| Boot fresh from new image | Discard/ignore process/rootfs snapshot and start from the latest template image. | Good when the user wants upgrade semantics and can tolerate losing in-memory/rootfs state. |
| Devolve to homedir | Preserve only a durable app-data layer, not process/rootfs state. | Matches issue #119's model for image upgrades: image changes invalidate memory/rootfs, but homedir can survive. |
| Explicit migration | Run a versioned migration tool or app protocol from old state to new image. | Best long-term UX for stateful products, but requires app/runtime-specific design. |

## Recommended Direction

- **Treat `ActorTemplate` as immutable for process/rootfs snapshots.** This is now enforced: the CRD rejects any `spec` mutation (`self == oldSelf`), so image/config changes require a new template revision rather than an in-place patch.
- **Record snapshot metadata.** Include template revision/spec hash, workload image digests, pause image digest, runsc hash/version, ateom protocol/state-format version, relevant runtime flags, and created time.
- **Compare on resume.** If metadata does not match the selected restore policy, fail with a clear "snapshot invalidated by image/runtime change" error.
- **Regenerate golden snapshots on compatibility-set changes.** If a new template revision/image is desired, create a new golden actor/snapshot for that revision rather than mutating the old one in place.
- **Separate mutable app data from process/rootfs snapshots.** The state-machine issue's memory/rootfs/homedir layering is the right direction: image upgrades should degrade from process snapshot to homedir/app-data restore unless an explicit migration exists. Substrate now provides a first building block for this: `DurableDir`-typed `volumes` on the `ActorTemplate`, plus a per-snapshot `SnapshotScope` (`snapshotsConfig.onPause` / `onCommit` = `Full` vs `Data`), where `Data` captures only durable-volume contents and excludes process memory and the rest of rootfs.
- **Cache immutable rootfs by digest for performance.** Issue #166's extracted-rootfs cache is compatible with this model: immutable image digest is a cache key and part of restore compatibility.
- **Be conservative with runsc/worker upgrades.** Changing runsc, ateom image, GPU driver, CPU feature set, runtime flags, or backend implementation should be modeled as a compatibility boundary until metadata says otherwise.

## Open Questions

- Should an actor track the template revision it was created from, or always point to a mutable template name?
- Should the first implementation reject mismatches, or support restoring with recorded old image/runsc immediately?
- Where should homedir/app-data live so it can survive image upgrades cleanly?
- What is the operator policy for old image/runtime retention? How long must old digests remain pullable?
- How should WorkerPool/ateom rolling upgrades drain or preserve actors with incompatible snapshots?
- How should GPU driver and CPU feature compatibility be represented in snapshot metadata?

## Bottom Line

The existing repo discussion already converges on the same rule: process/rootfs snapshots are not image-upgrade artifacts. Image updates should either create a new template revision/golden snapshot, reject old snapshots, restore with the old recorded compatibility set, or devolve to a lower state layer such as homedir. Silent restore of an old checkpoint against a new image is the behavior to avoid.
