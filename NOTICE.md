# NOTICE — ZONTAL fork of raft-cs

This repository is ZONTAL's fork of [marcschier/raft-cs](https://github.com/marcschier/raft-cs), the source of the
`RaftCs` packages. The original work is Copyright (c) marcschier and licensed under the MIT License (see
[LICENSE](LICENSE)); the licence and copyright notices are unchanged. ZONTAL's changes are released under the same
licence.

## Base

- Upstream release tag `v1.1.3` (commit `df44fde`, 2026-10-12), the latest upstream release. The branch `zontal/1.2.0`
  is that tag plus the patches below.
- The `RaftCs` 1.1.x package metadata names the repository `github.com/marcschier/crdt`; the source of `RaftCs` is
  `github.com/marcschier/raft-cs` (the commit recorded in `RaftCs` 1.1.2, `cc3305c`, is its tag `v1.1.2`).

## Packages and version

Every package keeps its upstream id and is versioned **`1.2.0-zontal.3`** by Nerdbank.GitVersioning
([version.json](version.json)):

| Package | Assembly |
| --- | --- |
| `RaftCs` | `Raft` |
| `RaftCs.Storage.File` | `Raft.Storage.File` |
| `RaftCs.Transport` | `Raft.Transport` |
| `RaftCs.Transport.NanoMsg` | `Raft.Transport.NanoMsg` |

The packages are published by [.github/workflows/zontal-pack.yml](.github/workflows/zontal-pack.yml) on a tag
`v1.2.0-zontal.*` to the zontal GitHub Packages feed and to the static feed
`https://zontal.github.io/nuget-feed/index.json`, which restores without credentials.

`RaftCs.Storage.File` depends on `System.IO.FileSystem.AccessControl` 5.0.0 for its `netstandard2.0` and
`netstandard2.1` targets only; the `net8.0`, `net9.0`, and `net10.0` targets use the in-box file access control APIs.

## Strong naming

The assemblies are signed with the repository's committed key `eng/Raft.snk` (public key token `dfdc46ace5e226f1`), and
the assembly version stays `1.1.0.0` (`assemblyVersion` in version.json), so the assembly identity is the one of the
upstream `RaftCs` 1.1.x packages, for example `Raft, Version=1.1.0.0, Culture=neutral, PublicKeyToken=dfdc46ace5e226f1`.
Assemblies compiled against upstream `RaftCs` 1.1.x bind to these without binding redirects.

## Patches

Each patch is a branch from `v1.1.3` with failing-first tests, merged into `zontal/1.2.0`, and proposed upstream.
`1.2.0-zontal.3` contains each branch up to the head below.

| # | Branch | Head | Change | Upstream |
| --- | --- | --- | --- | --- |
| 1 | `fix/committed-entry-index` | `9ee9b3a` | `RaftNode.CommittedItems`: each committed command with its log index and term, so a state machine knows the log position of its state. `Committed` is a payload view of the same queue. `RaftNode.AppliedIndex` advances before the commands of an applied batch are queued. | [marcschier/raft-cs#9](https://github.com/marcschier/raft-cs/pull/9) |
| 2 | `fix/file-storage-compaction` | `153c629` | `FileRaftStorage.Compact` stages the retained log beside the log file and swaps it in, durably and retryably, so compaction bounds the file. The file is rewritten only when its owner, group, and access control can be preserved exactly; otherwise the compaction is logical (recorded in the log, which keeps its access and does not shrink). | [marcschier/raft-cs#10](https://github.com/marcschier/raft-cs/pull/10) |
| 3 | `fix/snapshot-delivery` (on patch 1) | `fa6e6a1` | Application snapshots: `IRaftSnapshotStorage` (implemented by `MemoryStorage` and `FileRaftStorage`), `RaftNode.CreateSnapshotAsync`, a `CommittedItemKind.Snapshot` item when a follower installs the leader's snapshot and first after a restart, and no compaction past the latest snapshot. The committed membership is stored with the index it reflects (`IRaftConfStateIndexStorage`), so a restart replays exactly the configuration changes after it. `FileRaftStorage` writes point-in-time snapshot files and installs snapshots crash-atomically. Membership records are record kind 7 (kind 4 is patch 2's compaction-base record) and follow a compaction record at index 2^64−1 (a rollback fence) that earlier releases, which ignore membership records, fail on. | [marcschier/raft-cs#11](https://github.com/marcschier/raft-cs/pull/11) |

Patches 2 and 3 both change `FileRaftStorage`; the fork's merge of the two keeps the behaviour of both and adds what
neither branch can carry alone: a rewritten compacted log starts with the log generation and keeps the rollback fence
and the membership records, so the snapshot still names the log that continues it and the membership stored in the log
survives the rewrite. The record numbering and the rollback fence, first added by this merge in `1.2.0-zontal.2`, are
part of patch 3 since `fa6e6a1`.

## Changes since 1.2.0-zontal.2

The on-disk format is unchanged: stores written by `1.2.0-zontal.2` and `1.2.0-zontal.3` open with either version.

- **Snapshot installation crash window:** installing a snapshot writes the snapshot file before it replaces the log.
  When the log holds no rollback fence yet, it now gets one, flushed to disk, before the snapshot file is written. If
  the process stops between the two, an earlier release refuses the store instead of replaying the superseded log after
  the new snapshot.
- **Tests and CI gates:** the upstream branches now carry the test fixes that `1.2.0-zontal.2` applied in its merge
  (wrapped long lines, camel-cased local functions,
  `CreateSnapshotAsync_AfterAStorageWriteFailure_FailsInsteadOfWaiting` waiting for the first applied entry, and the
  compaction rewrite checked by its retained entries), and new tests cover the fence before the first membership
  record and the installation crash window over a log of an earlier release.

## Changes in 1.2.0-zontal.2 (since 1.2.0-zontal.1)

`1.2.0-zontal.1` contained the first revisions of patches 2 and 3 (`7bd15e4`, and `03e5334`, the change that `410d7db`
carries on top of patch 1); `1.2.0-zontal.2` contains the revisions after upstream review.

- **Compaction** is staged and durable: the retained log is written beside the log file and swapped in only after the
  write succeeded, a failed swap leaves the log retryable, and with `Fsync` on Unix the directory is synced before the
  next write. The log file is rewritten only when its owner, group, and access control can be preserved exactly on the
  new file; otherwise the compaction is logical only.
- **Dependency:** `RaftCs.Storage.File` adds `System.IO.FileSystem.AccessControl` 5.0.0 for its `netstandard2.0` and
  `netstandard2.1` targets only.
- **Snapshot delivery** adds the optional `IRaftConfStateIndexStorage` interface, which `MemoryStorage` and
  `FileRaftStorage` implement, next to the optional `IRaftSnapshotStorage` of `1.2.0-zontal.1`. No public API is
  removed or changed; `IRaftStorage` and `IRaftWritableStorage` are as in upstream `v1.1.3`.
- **On-disk additions** in `FileRaftStorage`: the snapshot file ends with a marker and the generation of the log that
  continues it; a log written whole starts with a log generation record; the membership is kept in the log as
  configuration records with the index they reflect, after a rollback fence; a rewritten compacted log holds a
  compaction-base record and a guard record that older readers refuse.
- **Rollback:** stores written by `1.2.0-zontal.1` open with `1.2.0-zontal.2` and later. After one of them rewrites a
  compacted log, installs or creates a snapshot, or records a membership in the log, `1.2.0-zontal.1` cannot open the
  store: it fails closed rather than misread it.

## History

`1.2.0-zontal.1` (2026-10-14) contained patch 1 including commit `9ee9b3a`, which advances `RaftNode.AppliedIndex`
before the commands of an applied batch are queued. The packages first published as `1.2.0-zontal.1` on 2026-10-14
lacked that commit; they were replaced on the same day by a build that includes it, under the same version. A restore
that still holds the first build in its NuGet global packages folder keeps it until that folder entry is removed
(`dotnet nuget locals global-packages --clear`). Later versions are not affected.

Other changes in this fork, not proposed upstream: this notice, the README section, version.json, the `zontal-pack`
workflow, and a guard that keeps the upstream `ci` workflow from publishing to the upstream author's feed.
