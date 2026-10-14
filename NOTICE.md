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

Every package keeps its upstream id and is versioned **`1.2.0-zontal.1`** by Nerdbank.GitVersioning
([version.json](version.json)):

| Package | Assembly |
| --- | --- |
| `RaftCs` | `Raft` |
| `RaftCs.Storage.File` | `Raft.Storage.File` |
| `RaftCs.Transport` | `Raft.Transport` |
| `RaftCs.Transport.NanoMsg` | `Raft.Transport.NanoMsg` |

The packages are published to the NuGet feed of the `studio` organization by
[.github/workflows/zontal-pack.yml](.github/workflows/zontal-pack.yml) on a tag `v1.2.0-zontal.*`.

## Strong naming

The assemblies are signed with the repository's committed key `eng/Raft.snk` (public key token `dfdc46ace5e226f1`), and
the assembly version stays `1.1.0.0` (`assemblyVersion` in version.json), so the assembly identity is the one of the
upstream `RaftCs` 1.1.x packages, for example `Raft, Version=1.1.0.0, Culture=neutral, PublicKeyToken=dfdc46ace5e226f1`.
Assemblies compiled against upstream `RaftCs` 1.1.x bind to these without binding redirects.

## Patches

Each patch is a branch from `v1.1.3` with failing-first tests, merged into `zontal/1.2.0`, and proposed upstream.

| # | Branch | Change | Upstream |
| --- | --- | --- | --- |
| 1 | `fix/committed-entry-index` | `RaftNode.CommittedItems`: each committed command with its log index and term, so a state machine knows the log position of its state. `Committed` is a payload view of the same queue. | [marcschier/raft-cs#9](https://github.com/marcschier/raft-cs/pull/9) |
| 2 | `fix/file-storage-compaction` | `FileRaftStorage.Compact` rewrites the log file to the retained entries (written beside the log, then swapped in), so compaction bounds the file. | [marcschier/raft-cs#10](https://github.com/marcschier/raft-cs/pull/10) |
| 3 | `fix/snapshot-delivery` (on patch 1) | Application snapshots: `IRaftSnapshotStorage` (implemented by `MemoryStorage` and `FileRaftStorage`), `RaftNode.CreateSnapshotAsync`, a `CommittedItemKind.Snapshot` item when a follower installs the leader's snapshot and first after a restart, no compaction past the latest snapshot, and an atomically written, verified snapshot file. | [marcschier/raft-cs#11](https://github.com/marcschier/raft-cs/pull/11) |

Not in `1.2.0-zontal.1`: commit `9ee9b3a` on `fix/committed-entry-index` (part of the upstream pull request)
advances `RaftNode.AppliedIndex` before the commands of an applied batch are queued. In `1.2.0-zontal.1` a consumer can
read a command whose index `AppliedIndex` does not cover yet, so a `CreateSnapshotAsync` or `CompactAsync` at that index is
refused until the node publishes it. It ships with the next fork version, which needs a version decision.

Other changes in this fork, not proposed upstream: this notice, the README section, version.json, the `zontal-pack`
workflow, and a guard that keeps the upstream `ci` workflow from publishing to the upstream author's feed.
