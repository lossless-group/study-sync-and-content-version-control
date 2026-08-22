# Sync and Content Version Control — Study

## The question

> For a folder of documents shared **bidirectionally** between an operator and a
> **non-technical** client, backed by **blob storage**, at which layer does
> history live — the transport, the store, the application, or the document?

The interesting bits are not "which sync tool is fastest." The interesting bit
is that the ask people actually voice — *"a Dropbox that agents can work in, with
version history, per directory"* — is **three problems wearing one coat**:

| Layer | The question it answers | Failure if you get it wrong |
|---|---|---|
| **Transport** | how bytes get from one machine to another | corruption, silent divergence, orphaned edits |
| **History** | what did this look like before, and what changed | unbounded storage, or no undo at all |
| **Legibility** | what changed, why should a human care | you ship sync and still can't show progress |

Every entry below picks a different layer to be opinionated about and inherits
the other two. Reading them side by side is the point.

## Why this study exists

Three prior explorations in the Lossless tree circled this and none closed:

- `self-host-stack/context-v/explorations/Instantly-Synced-Team-Folder-Nextcloud-vs-Alternatives.md`
  (2026-07-08) — compared Nextcloud / Syncthing / Seafile, leaned Syncthing, left `Open`.
  Also flagged the footgun this study keeps running into: **continuously syncing a live
  `.git` working tree with any file-sync tool is a well-known way to corrupt a repository.**
- `ai-labs/augment-it/context-v/explorations/Syncthing-For-Collaborator-Access-To-The-Corpus.md`
  (2026-07-17) — chose asymmetry (`Receive Only` collaborators) precisely to make
  conflicts structurally impossible. Never built.
- `ai-labs/corpora-builder/context-v/plans/Corpora-Builder-MVP-R2-Native-With-Checkpoint-History.md`
  (2026-08-08) — the **HISTORY** decision: R2 has no object versioning, therefore
  version control must be application-level, therefore *content-addressed objects +
  checkpoint manifests*. "Git's data model, minus git, on a substrate that has neither."

Six days after HISTORY, `ai-labs/flave-ai/context-v/specs/Master-Flave-An-Agent-Native-Document-Format-and-Publisher.md`
§8.2 resolved the opposite way for `.flave` documents: **every bundle is a `jj` repo.**
Two resolved decisions, two repos, six days apart, and one syncbox now expected to
serve both. **That contradiction is what this study is for.**

## The reading checklist

Apply to every entry:

- **Unit of sync.** File? Directory? Library? Repository? Document? What is the
  smallest thing that can be shared independently, and can two of them nest?
- **Where history lives.** In the transport, the store, the app, or the document
  itself? Can you get history *without* the tool that wrote it?
- **Content addressing.** Is the store content-addressed? At what granularity —
  whole file, fixed block, content-defined chunk? What does dedup actually buy?
- **Blob-storage story.** Can it target S3/R2 *natively*, or does it need a POSIX
  filesystem and a syncer underneath? Does it need object versioning (R2 has none)?
- **Conflict semantics.** Merge, rename-and-keep-both, last-writer-wins, refuse,
  or represent-as-first-class? **What does a non-technical person see?**
- **Asymmetry.** Can one side be read-only *structurally*, or only by agreement?
- **Hand-recoverability.** With the tool gone, is the data still plain files a
  human can open — or is it an opaque pack format needing a restore step?
- **Labels and legibility.** Can a checkpoint carry a human-written reason? Is
  there any surface that renders "what changed" for someone who won't read a diff?
- **Operational cost.** Daemon? Server? Database? Or a single binary against a bucket?

## The design space at a glance

| Bet | Entry |
|---|---|
| History in the VCS — five pluggable stores, cloud-shaped traits, no cloud implementation | [jj (Jujutsu)](./jj) |
| All three layers in one product — library-scoped commit DAG + CDC blocks + daemon + web UI; S3 behind the paid tier | [Seafile](./seafile) |
| Transport only — peer-to-peer, no server, no store, structural asymmetry via four folder types | [Syncthing](./syncthing) |
| History only — write-once content-addressed snapshots natively into a bucket, no sync, no UI | [restic](./restic) |
| History only, with the legibility layer — a description on every snapshot and policies that inherit per directory | [Kopia](./kopia) |
| History in the document — CRDT merge, so concurrent edits have an answer instead of a second filename | [Automerge](./automerge) |
| A *surface*, not an engine — the GUI over rclone, and the only entry that solves running-on-a-schedule cross-platform | [Rclone UI](./rclone-ui) |
| Content is *optional* — pointers in git, bytes fetched on demand, with the bookkeeping that makes deleting them safe | [git-annex](./git-annex) |

## The checklist, filled in

Index only — the interpretation lives in
`ai-labs/context-v/explorations/A-Syncbox-For-Client-Document-Folders.md`.

| | jj | Seafile | Syncthing | restic | Kopia | Automerge | Rclone UI |
|---|---|---|---|---|---|---|---|
| **Unit** | repo | library | folder | repo | **directory** | document | rclone path pair |
| **History lives in** | VCS | store | *(local archive)* | store | store | document | **nowhere** |
| **Content-addressed** | yes | yes + CDC | transfer only | yes + CDC | yes + CDC | changes only | inherits rclone |
| **Native S3/R2** | no | **Pro only** | impossible | **yes** | **yes** | not in repo | **yes, all backends** |
| **Needs object versioning** | n/a | no | n/a | **no** | **no** | n/a | n/a |
| **Conflicts** | recorded, first-class | rename-both | rename-both | n/a | n/a | **merged** | surfaces bisync's |
| **Structural read-only** | no | server-side | folder type | credential | **storage wrapper** | protocol hint | not modelled |
| **Plain files on disk** | **yes** | client only | **yes** | no | no | **no** | **yes** |
| **Human reason per version** | commit desc | **`Commit.Desc`** | **none** | **none** | **`Description`** | change `message` | **none** |
| **Retention policy** | manual `gc` | per-library days | thinning curve | `ExpirePolicy` | **per-dir + pins** | **none possible** | log rotation only |
| **Ops cost** | one binary | **server + DB + web** | one binary | one binary | binary (+server) | library + glue | desktop app |

Bold marks where an entry is the outlier — best or worst — on that row.

**Rclone UI is a surface over a transport, so most rows read as rclone's.** It is pinned
for one row the others do not have — see its entry — and for correcting the claim that
rclone cannot run as a service.

**Nothing in the table does all three layers on a bucket.** The two entries that
speak S3 natively (restic, Kopia) do not sync and do not leave a plain tree. The
two that leave a plain tree (jj, Syncthing) cannot reach a bucket. The one that
does everything (Seafile) puts the bucket behind a paid licence and keeps no plain
tree server-side. That gap is the finding, not a gap in the reading.

## In the study now

### [jj (Jujutsu)](./jj)

Git-compatible VCS, Rust, Apache-2.0. Pinned `9d905d5` (v0.44.0, 2026-08-21).
Upstream: <https://github.com/jj-vcs/jj>. Maintained by the `jj-vcs` org;
originated at Google, now community-governed.

**Why it's here.** `flave-ai`'s spec §8.2 resolved that every `.flave` bundle is
a `jj` repo. This entry is the source of truth for whether the two claims that
decision rests on — retained git compatibility, and the operation log as the
agent-undo primitive — survive contact with the code. Also the study's only
entry where *conflicts are first-class recorded values* rather than a second
file with a longer name.

**The thing to read it for.** `StoreFactories` (`lib/src/repo.rs:434-440`) is
five backends, not one; only `GitBackend` is production-grade and `SimpleBackend`
is labelled a proof of concept by the maintainers. And a colocated repo splits in
half: content in `.git/`, but change IDs, predecessors, and the whole operation
log in `.jj/` — so a git-only recipient gets the document and loses the undo.

Deep write-up: [`context-v/profiles/Profile__Jujutsu.md`](./context-v/profiles/Profile__Jujutsu.md)

### [Seafile](./seafile) + [seafile-server](./seafile-server)

Two repos, both needed. Client daemon: `2ebf6ac8` (v9.0.21, 2026-08-07), GPLv2.
Server: `8c47d5f` (2026-08-11), AGPLv3. Upstream: <https://github.com/haiwen/seafile>,
<https://github.com/haiwen/seafile-server>. Maintained by haiwen (Seafile Ltd.),
open-core — Community Edition is pinned here, Professional is closed.

**Why it's here.** It is the closest existing system to the question this study
asks, and it is where the search actually landed: the [seasync](https://github.com/seasync)
org publishes *unofficial Seafile clients*, not a separate engine. Pinning it makes
the 2026-07-08 exploration's pass-over an informed one instead of a repeat.

**The thing to read it for.** Git's object model reimplemented — commit → SeafDir →
Seafile (an ordered list of `block_ids`) → blocks — with Rabin content-defined
chunking underneath (`common/cdc/cdc.c:25-32`, 1 MiB average). `Commit.Desc` is a
human-written reason on every version, which is the legibility field the rest of the
study should be measured against. And `fileserver/objstore/objstore.go` is 56 lines
that show a clean four-method storage seam with **exactly one open-source
implementation** — the S3 backend is Professional.

Deep write-up: [`context-v/profiles/Profile__Seafile.md`](./context-v/profiles/Profile__Seafile.md)

### [Syncthing](./syncthing)

Peer-to-peer continuous file sync, Go, MPL-2.0. Pinned `38ac58c8` (nearest tag
`v2.1.4-rc.1`, 2026-08-20). Upstream: <https://github.com/syncthing/syncthing>.
Maintained by the Syncthing Foundation.

**Why it's here.** The Lossless tree has leaned toward it twice without installing
it, and `augment-it`'s exploration ended with five open questions about it. Pinning
the source answers two of them outright.

**The thing to read it for.** `lib/model/folder_recvonly.go:29-57` settles whether a
receive-only device silently orphans a collaborator's edit: **it does not.** The edit
is flagged `FlagLocalReceiveOnly`, sent outward with an empty Version so nobody is
told they are behind, counted by the database's per-flag size accounting, and offered
back as a **Revert**. Also here for `receiveencrypted` — an untrusted always-on device
holding a replica it cannot read — and for `lib/versioner/staggered.go:49-54`, a
four-line progressive-thinning retention curve worth stealing outright.

**And what it can never be.** There is no shared history, only a per-device local
archive of whole-file copies under `.stversions/` carrying no author and no reason.
A peer is always a daemon on a real machine, so it cannot target a bucket at all.

Deep write-up: [`context-v/profiles/Profile__Syncthing.md`](./context-v/profiles/Profile__Syncthing.md)

### [restic](./restic)

Content-addressed snapshot backup, Go, BSD-2-Clause. Pinned `a80be147` (nearest
tag `v0.19.1`, 2026-08-01). Upstream: <https://github.com/restic/restic>.

**Why it's here.** To answer honestly whether `corpora-builder`'s unbuilt Phase 4
is just restic. The HISTORY decision proposed *"a content-addressed object store
plus checkpoint manifests… git's data model, minus git."* restic has shipped that
since 2014, and its layout is `data/<ab>/<sha256>` + `snapshots/<id>` — the same
scheme, the same two-hex fan-out.

**The thing to read it for.** `doc/design.rst:28-120` next to the HISTORY bucket
diagram. Then `internal/backend/` — eleven backends including `s3` and `rclone`,
and **restic needs no object versioning**, which is precisely the R2 gap that
forced HISTORY in the first place. Then `internal/data/snapshot_policy.go:14-28`,
a retention grammar where `Tags` can pin a snapshot out of expiry entirely.

**Where it stops.** No browsable `live/` tree — a repository is encrypted packs,
so constraint 1 (files-as-truth, hand-recoverable) does not survive. And
`Snapshot` carries `Tags` and computed statistics but **no free-text reason**,
which is the one field the make-progress-visible goal actually needs.

Deep write-up: [`context-v/profiles/Profile__Restic.md`](./context-v/profiles/Profile__Restic.md)

### [Kopia](./kopia)

Content-addressed snapshots with a per-directory policy tree, Go, Apache-2.0.
Pinned `4ebb2be6` (nearest tag `v0.23.1`, 2026-08-21). Upstream:
<https://github.com/kopia/kopia>.

**Why it's here.** Read immediately after restic: same bet, and the differences
land exactly where this study's question is. Kopia's `snapshot.Manifest` has the
**`Description`** field restic has no equivalent for, plus key-value `Tags`,
manual `Pins` that hold a snapshot out of expiry, and `RetentionReasons`
explaining why one survived. And its policies — retention, compression, file
selection, chunking algorithm, before/after hooks — **attach per directory and
inherit**, which is the literal shape of "per-dir sync with version control."

**The thing to read it for.** `snapshot/manifest.go:18-40` against restic's
`internal/data/snapshot.go:17-34`. Then `snapshot/policy/`, where every effective
policy field records *which level supplied it*. Then
`repo/blob/readonly/readonly_storage.go` — structural read-only one layer below
where Syncthing and Seafile put it.

**Where it stops.** Same two walls as restic: it does not sync, and the bucket
holds packed encrypted contents rather than a plain markdown tree.

Deep write-up: [`context-v/profiles/Profile__Kopia.md`](./context-v/profiles/Profile__Kopia.md)

### [Automerge](./automerge)

CRDT library — Rust core with WASM/JS and C bindings, MIT. Pinned `47908d6c`
(nearest tag `js/automerge-3.4.1`, 2026-08-17). Upstream:
<https://github.com/automerge/automerge>, from Ink & Switch.

**Why it's here.** To make `flave-ai`'s §8.4 v3 deferral a decision rather than an
omission, and because it is the **only entry in this study where two people
editing the same paragraph has a defined answer.** Every other entry resolves a
conflict by renaming and keeping both.

**The thing to read it for.** `src/transaction/commit.rs:3-8` and
`src/change.rs:47-93` — history as a DAG of authored, timestamped, optionally
captioned changes. Then `automerge.rs:1318`: `diff(before_heads, after_heads) ->
Vec<Patch>`, which is *structured* change data and the best raw material in the
study for a progress feed.

**What it costs.** A document is a `Map`/`List`/`Text` object graph, not a
directory — so `rg`, Obsidian, rclone, and every agent that reads files stop
working without an export step. And there is **no truncate, compact, or GC
anywhere in the API**: full history is retained by construction, which is what
makes the merge guarantees hold. Storage and network adapters are not even in
this repo; they live in [automerge-repo](https://github.com/automerge/automerge-repo).

Deep write-up: [`context-v/profiles/Profile__Automerge.md`](./context-v/profiles/Profile__Automerge.md)

### [Rclone UI](./rclone-ui)

Desktop GUI over rclone. Tauri + React + Rust, Apache-2.0. Pinned `da6e134`
(v3.7.3, 2026-08-21). Upstream: <https://github.com/rclone-ui/rclone-ui> ·
<https://rcloneui.com/>.

**Why it's here, and it's different from the rest.** Every other entry is an
engine; this is a *surface* over one. It earns the pin for two things nothing
else in the study supplies.

**One: it corrects the record on rclone.** rclone has a full HTTP remote-control
daemon (`rclone rcd`) and this app is built entirely on it —
`/core/stats`, `/core/transferred`, `/job/status`, `/job/batch`, `/sync/sync`,
`/sync/bisync`. The 2026-08-22 exploration claimed rclone was batch-only. That is
true of *watching* and wrong about running as a service. `/core/transferred`
is a per-item record of what moved, grouped by job — a change record for the
transport layer, ready-made.

**Two: `src-tauri/src/scheduler/` is 5,167 lines solving "run this on a schedule,
cross-platform."** A `SchedulerBackend` trait with four backends — per-user
**LaunchAgents** on macOS user-mode, **crontab** for Linux and macOS system-mode,
**Task Scheduler** via `schtasks.exe` on Windows — plus append-only JSONL run
history and an `interrupted` flag *synthesized* from a started event with no
finished one. That last bit is the cheapest available answer to "did my
automation quietly die." Read `launchd.rs:1-12` first: it explains the TCC and
Keychain reasoning for choosing a LaunchAgent over cron, which is precisely the
kind of detail a study exists to pin.

**What it does not have.** No file version history at all — and
`lib/rclone/versions.ts` is a false lead, managing *rclone binary* versions, not
file ones. No `--backup-dir` use anywhere. The UI layer is React, which the tree
prohibits, so only the Rust half is liftable. Its embedded PDF/DOCX/XLSX viewers
are a non-finding — a reader already owns viewers, and a client-facing feed only
has to hand over the file.

Deep write-up: [`context-v/profiles/Profile__Rclone-UI.md`](./context-v/profiles/Profile__Rclone-UI.md)

### [git-annex](./git-annex)

Large-file manager layered on git. Haskell, AGPL-3+. Pinned `6bc17908` (nearest
tag `10.20260717`, 2026-08-21). Upstream: `https://git.joeyh.name/git/git-annex.git`
— **no GitHub mirror exists.** By Joey Hess.

**Why it's here: the study's own trigger fired.** The list below said to add it
*"if the binary-sibling problem (PDFs in a corpus) turns out to be the hard
part."* It did — a corpus is 1,715 text files and 78 binaries, and **the binaries
are 90.5% of the bytes.** They broke jj via LFS, they dominate sync cost, and
they are why a 4-year-old laptop is a constraint.

**The thing to read it for.** `doc/git-annex-drop.mdwn:25-36` — it verifies a
copy exists elsewhere *before* deleting locally, and says so when it won't.
`numcopies` is the piece an in-house design skips and regrets. Then
`doc/internals.mdwn:62-95`: location tracking lives in an orphan branch whose
files are **"designed to be auto-merged by simply concatenating them together"**,
each line timestamped. That is a CRDT in a text file, solving multi-peer metadata
merge without a CRDT library.

**Why read rather than adopt.** Its symlink mechanics leak, it is Haskell, and
`CorpusStore.exists()` + `stat()` already provide the verification primitive
git-annex builds by hand.

Deep write-up: [`context-v/profiles/Profile__Git-Annex.md`](./context-v/profiles/Profile__Git-Annex.md)


## Not in the study, and why

- **Nextcloud / ownCloud** — evaluated and passed over in the 2026-07-08
  exploration; groupware breadth is the opposite of the narrow ask, and its sync
  leg is the part every source rates weakest.
- **Dropbox / Google Drive** — closed source. They set the UX expectation this
  study is measured against, not the implementation evidence.
- **Yjs** — Automerge is pinned as the CRDT data point; Yjs is the same bet with a
  different encoding. Pin it too only if the CRDT tier stops being deferred.
- **automerge-repo** — where Automerge's storage and network adapters live. Not
  pinned because the CRDT question is answered without it; pin it the moment the
  v3 tier is reopened.
- ~~**git-annex**~~ — **pinned 2026-08-22**; the trigger below fired.
- **DVC, Perkeep, ostree** — further prior art on content-addressed large-file
  storage. Still held back; git-annex covers the model, and a second example
  would restate rather than extend it.
