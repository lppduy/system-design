# M5-11: Git Internals

## Core Insight: Content-Addressable Filesystem

Everything stored by SHA-1 hash of its content. Same content = same hash = stored once.

```
content → SHA-1 hash → stored at .git/objects/<first-2-chars>/<rest>
```

Same principle as: CDN cache keys, Docker image layers, IPFS.

---

## 4 Object Types

### Blob — file content (no filename, no path, just bytes)
```bash
git cat-file -p <hash>   # show content
git cat-file -t <hash>   # show type
```

### Tree — directory listing
```
100644 blob ce0136...  greeting.txt
040000 tree a1b2c3...  src/
```
Points to blobs (files) and other trees (subdirectories). Recursive, like filesystem inodes.

### Commit — snapshot + metadata
```
tree 8f4a3b...          ← root tree (entire project state)
parent a1b2c3...        ← previous commit
author / committer / message
```
A commit is a snapshot, not a diff. Points to a tree representing ALL files at that moment.

### Tag — named pointer to a commit (annotated tags)

---

## What Happens When You Update a File

```
greeting.txt changed from "hello" to "hello world"

1. New blob created (different content = different hash)
2. New root tree created (points to new blob, but SAME tree for unchanged dirs)
3. New commit created (points to new tree, parent = old commit)

Unchanged files/directories → SAME hash → reused (structural sharing)
```

Only what changed gets new objects. Everything else shared by reference.

---

## How git diff Works

Git doesn't store diffs. Computes them on the fly:
1. Get root tree of commit A and commit B
2. Compare tree entries — same hash? Skip. Different? Compare blob contents.

---

## Refs: Branches and HEAD

```
.git/refs/heads/main    → file containing a commit hash
.git/refs/heads/feature → file containing a commit hash
.git/HEAD               → "ref: refs/heads/main" (which branch you're on)
```

A branch = a file with 41 bytes (commit hash). Creating a branch = writing one file. Cheap.

---

## DAG (Directed Acyclic Graph)

Commits form a DAG. Each commit points to parent(s). Merge commits have two parents. Acyclic = can never loop back. Same structure as blockchain.

**Hash cascade:** Change any byte in history → every descendant commit gets a new hash (integrity guarantee / tamper detection).

---

## Packfiles — Efficient Storage

### Problem
100 commits changing 1 line each = 100 full copies of the file. Wasteful.

### Solution
`git gc` packs loose objects into packfiles with delta compression:
- Latest version stored in full (accessed most often)
- Older versions stored as deltas (tiny diffs from newer version)

Like video encoding: keyframes (full) + delta frames (differences).

**When packing happens:** `git gc`, `git push`, or auto when >~6700 loose objects.

---

## Merge Strategies

### Fast-forward
Main didn't change since branch. Just move pointer forward. No new commit.
```bash
git checkout main && git merge feature
# main pointer moves to feature's tip
```

### Three-way Merge
Both branches changed. Uses 3 inputs: common ancestor + both tips.
- Same in both → keep
- Changed in one → take the change
- Changed in both → CONFLICT (human resolves)

Creates merge commit with two parents.

```bash
git merge feature
# CONFLICT → edit file → git add → git commit
```

### Rebase
Replay your commits on top of another branch. Creates NEW commits (different hashes).
```bash
git checkout feature && git rebase main
# feature's commits replayed on top of main's tip
# feature changes, main untouched
```

**Golden rule:** Never rebase commits others have already pulled.

### Merge main into feature (alternative to rebase)
```bash
git checkout feature && git merge main
# Creates merge commit on feature. Safe for shared branches.
```

### When to use which
- Solo/unpushed → rebase (clean linear history)
- Shared branch → merge (safe, no rewrite)
- Merging completed feature into main → merge or PR

### Key mental model
```
rebase = I move myself on top of them (my branch rewrites)
merge  = I pull them into me (merge commit on my branch)
```

---

## Distributed Model

Every clone = full repo (all objects, all history). No central authority.

```
git push  = send objects remote doesn't have + update refs
git fetch = download objects you don't have + update remote-tracking refs
git pull  = fetch + merge
```

All ops except push/fetch are 100% local. Works offline.

---

## System Design Patterns from Git

| Git Concept | Seen elsewhere |
|---|---|
| Content-addressable storage | Docker layers, CDN, IPFS |
| DAG history | Blockchain, dependency graphs |
| Structural sharing | Immutable data structures, React virtual DOM |
| Delta compression | Video encoding, rsync, incremental backups |
| Distributed replication | CRDTs, eventual consistency |
| Three-way merge | Google Docs conflict resolution |

---

## Interview Angle

1. **Content-addressable** — hash = address. Dedup, integrity, tamper detection.
2. **4 objects** — blob/tree/commit/tag. Commit → tree → blobs. Structural sharing.
3. **Packfiles** — newest = full base, old = deltas. Efficient storage.
4. **Merge strategies** — fast-forward, three-way (common ancestor), rebase (replay).
5. **Distributed** — every clone is complete. Push/fetch exchange packfiles.
6. **Patterns** — content-addressable (CDN), DAG (blockchain), delta compression (video).
