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


## Not in the study, and why

- **Nextcloud / ownCloud** — evaluated and passed over in the 2026-07-08
  exploration; groupware breadth is the opposite of the narrow ask, and its sync
  leg is the part every source rates weakest.
- **Dropbox / Google Drive** — closed source. They set the UX expectation this
  study is measured against, not the implementation evidence.
- **Yjs** — Automerge is pinned as the CRDT data point; Yjs is the same bet with a
  different encoding. Pin it too only if the CRDT tier stops being deferred.
- **git-annex, DVC, Perkeep, ostree** — real prior art on content-addressed
  large-file storage. Held back to keep the question narrow; add them if the
  binary-sibling problem (PDFs in a corpus) turns out to be the hard part.
