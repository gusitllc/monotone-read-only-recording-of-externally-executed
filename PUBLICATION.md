# Monotone, read-only recording of externally executed infrastructure deployments from versioned cloud-state snapshots

**Defensive Publication** - published to establish prior art.
**Publisher:** Gus IT LLC.

## Abstract

A deployment orchestrator records the progress of an infrastructure deployment that it does not run. The external run is observed only through read-only snapshots of cloud resource state that follow a strict versioned schema. Pure predicates map each snapshot to the orchestrator's canonical step keys with one of done, failed, running, pending, skipped or unproven; a fact the snapshot does not carry makes the step unproven and nothing is written for it. A failed validation step that belongs to a step overrides resource-level evidence. Writes begin with a conditional claim on the deployment's newest snapshot time, and every stage write is a compare-and-set that refuses older snapshots and refuses to move a finished stage back to running or pending. Snapshots with identical content are stored once. The overall status reaches succeeded only when the final deploy step and the deploy settings agree.

## Technical Field

Infrastructure deployment orchestration and observability, specifically recording the state of deployments executed outside the orchestrator from snapshots of cloud resource state.

## Background

Operators often build some clusters outside the orchestration tool that normally runs them, for example through a vendor portal template, yet still want those clusters to appear in the same per-step and per-node records as orchestrated ones. A common approach mirrors cloud state by overwriting the stored record with the latest read. That fails in practice. Reads are partial: throttling, expired tokens and missing permissions remove fields, and a mirror that treats an absent field as not done rewinds finished steps. Reads can target the wrong context, such as another subscription, and a not-found answer caused by authorization looks the same as a resource that is really absent. Two sync jobs can overlap and the older read can land last. Re-running a validation in the vendor portal resets its step statuses and would make a completed record look unfinished. Finally, a mirror that may also act on the resources risks the orchestrator interfering with a run it does not own.

## Summary of the Disclosure

The disclosure is a one-way recorder. A deployment record carries an immutable marker saying it is executed externally, and such a record is only ever written from snapshots. A snapshot is a JSON document with an exact schema version, a fixed set of typed top-level fields, a capture timestamp with an explicit time zone, and the identifiers of the subscription, resource group and cluster it describes; it must name the record's own target. A pure mapper turns the snapshot into one item per canonical step key and node with a status and a scrubbed evidence summary. Any predicate whose inputs are absent yields unproven, which is never written. The writer claims the record for the snapshot's time, stores the raw scrubbed snapshot once per distinct content, applies compare-and-set upserts that converge on the newest snapshot and never regress a stage, and then moves the rollup status and freshness stamp only while its claim still holds. Clusters found in the cloud but not yet tracked can be adopted as inert external records created from a fresh read.

## Detailed Description

Record marker. Each deployment record holds a recipe, fixed and hashed at creation, with a field stating that execution happens outside the orchestrator. Only such records accept snapshots; a retired record accepts none.

Snapshot intake. A separate read-only collector queries the cloud resource manager and assembles a document with resource group state, provider registrations, registered machines with extensions and edge-device details, the cluster resource, the deployment-settings resource with its validation and deployment step trees, key vaults, storage accounts and the scrubbed original response bodies. On receipt the recorder refuses the snapshot, and stamps the refused attempt with a check time and an error (never a success time), when it exceeds a size bound, has a different schema version, carries an unknown top-level field, misses a required identifier, has a field of the wrong JSON type, has a malformed timestamp, names a different subscription, resource group or cluster than the record, or is dated later than the present by more than a configured skew. A snapshot older than the last recorded one is refused as stale; one with the same timestamp is answered as a replay without writing.

Mapping. A pure function maps the snapshot to the canonical step catalog. Cluster-scoped predicates cover cloud preparation (resource group, storage accounts and provider registrations all succeeded), secret references (the deployment settings reference every required secret name and all point at one vault in the group), machine extensions and edge-device connectivity, edge-device validation (the deployment settings name exactly the record's machine set), the cluster resource shell, the validation run and the deploy run. Node-scoped predicates cover hardware serial number, firmware package state, operating system build, physical adapter names and link state, and machine registration and solution version. Any other key whose only evidence is a named vendor validation step is decided by those steps: all successful or skipped gives done, any failed gives failed, otherwise running or pending. A key with no observable evidence is unproven. Keys that the deployment's topology flavor does not include are never emitted.

Three-way proof. Every predicate first checks that the facts it needs are present. If a needed field is missing, the result is unproven with a reason, not pending and not done. A read failure such as a wrong subscription, an authorization fault or an expired token is never interpreted as resource-not-found; only a resource-level not-found counts as absent.

Validation override and updated builds. For keys decided from resource state that also own named validation steps, a failed validation step of the key's own overrides an otherwise good resource reading and fails the key. The operating system check treats the exact expected build, or any numerically later build on the expected platform edition, as done, because a cumulative update after the cluster formed moves the build forward on the same installation; an older build fails.

Writing. The writer first performs a single conditional update on the deployment record that sets a claimed snapshot time only when the record is not retired, the snapshot is strictly newer than the last recorded snapshot time, and no claim for a newer time is in flight. If the claim fails, nothing is written and the caller learns whether a newer snapshot won or the record was retired. A retry of the same snapshot after a crash re-claims and converges. The raw scrubbed snapshot is then stored with its schema version, unless a content hash computed over the snapshot without its timestamp equals that of the last stored snapshot; older stored snapshots beyond a retention count are pruned.

Stage writes. For each item that is not unproven, the writer reads the stored stage row. If moving from the stored status to the new one would rewind a finished stage to running or pending, or a running stage to pending, the write is refused, the row keeps its status, a warning event names the refused move and the move is listed as held. Otherwise an insert-or-update runs whose update half applies only when the stored row was not written by an earlier sync, has no snapshot time, or has a snapshot time not later than this snapshot's, and only when the same regression rule holds in the statement itself. Overlapping syncs therefore converge on the newest snapshot whatever order their writes land in. Approval data is carried over untouched; each status change appends an event.

Rollup and freshness. The overall status derived from the items is left unchanged when the final deploy step is unproven. It becomes succeeded only when the final deploy step is done and the deploy settings report a succeeded provisioning state and a successful deployment. A succeeded record can move only to failed, and only on a failed final step. Otherwise any running or done stage gives running. The status update and the freshness stamp, which records check time, success time, snapshot time and content hash, are each conditional on this snapshot still holding the claim or being the newest, so the success time moves only after every write above succeeded.

Adoption. A discovery pass with per-subscription reader identities reports clusters no record tracks. Adopting one takes only validated identifiers from the request, refuses if the cluster is tracked or no reader exists, reads the cluster and its deployment settings fresh, infers the topology flavor (refusing if it cannot), and creates an external record whose nodes are the machines the deployment settings name. A sync then runs.

## Disclosed Features (each disclosed alone and in combination)

1. A system that records an externally executed infrastructure deployment only from read-only snapshots of cloud resource state and never runs any of its steps.
2. The system validates each snapshot against a strict versioned schema with a fixed set of typed fields and a zoned timestamp, and refuses a snapshot that names a different target, is dated in the future beyond a bound, or is older than the last recorded one.
3. The system maps a snapshot to canonical step keys with pure predicates that return done, failed, running, pending, skipped or unproven, where any missing input yields unproven and an unproven result writes nothing.
4. The system lets a failed vendor validation step owned by a step key override resource-level evidence for that key.
5. The system treats an operating system build later than the expected build on the expected platform edition as the same installation, updated.
6. The method begins each write with a conditional claim on the record that succeeds only for a snapshot strictly newer than the last recorded one and not older than an in-flight claim.
7. The method writes each stage with a compare-and-set that applies only when the stored row is not from a newer snapshot and refuses moves from done to running or pending and from running to pending, logging each refused move as held.
8. The method stores the scrubbed raw snapshot with its schema version, skips storing a snapshot whose content without the timestamp equals the last one, and prunes beyond a retention count.
9. The method derives the overall status monotonically, reaching succeeded only when the final deploy step is done and the deploy settings agree, and leaving the status unchanged when the final step is unproven.
10. The system adopts an untracked cluster as an inert external record whose content comes from a fresh cloud read, not from the request body.

## Variations

- The snapshot may be pulled periodically by the recorder through read-only identities instead of being pushed by an operator tool.
- The compare-and-set may use a monotonically increasing sequence number issued by the collector instead of the capture timestamp.
- The content de-duplication may hash a canonical, key-sorted serialization rather than the snapshot as received.
- The regression rule may be extended to a full partial order over statuses, for example refusing failed to pending unless a new deployment record is created.
- The claim and stage writes may run in one database transaction where the data layer allows it, instead of a claim followed by conditional writes.
- Held regressions may raise a review item for an operator instead of only a warning event.

## Note on omitted parameters

The maximum snapshot size, the allowed clock skew, the snapshot retention count, the length cap on stored error summaries and the default provider and extension sets are omitted; they are implementation choices.
