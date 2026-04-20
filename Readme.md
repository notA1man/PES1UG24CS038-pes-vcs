# PES-VCS Lab Report

---

## Screenshots

### Phase 1: Object Storage

**Screenshot 1A** — `./test_objects` output showing all tests passing:

![Screenshot 1A](screenshots/1a.png)

**Screenshot 1B** — `find .pes/objects -type f` showing sharded directory structure:

![Screenshot 1B](screenshots/1b.png)

### Phase 2: Tree Objects

**Screenshot 2A** — `./test_tree` output showing all tests passing:

![Screenshot 2A](screenshots/2a.png)

**Screenshot 2B** — `xxd` of a raw tree object (first 20 lines):

![Screenshot 2B](screenshots/2b.png)

### Phase 3: Staging Area

**Screenshot 3A** — `pes init` → `pes add` → `pes status` sequence:

![Screenshot 3A](screenshots/3a.png)

**Screenshot 3B** — `cat .pes/index` showing the text-format index:

![Screenshot 3B](screenshots/3b.png)

### Phase 4: Commits and History

**Screenshot 4A** — `pes log` output with three commits:

![Screenshot 4A](screenshots/4a.png)

**Screenshot 4B** — `find .pes -type f | sort` showing object store growth:

![Screenshot 4B](screenshots/4b.png)

**Screenshot 4C** — `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain:

![Screenshot 4C](screenshots/4c.png)

---

## Phase 5 & 6: Analysis Questions

### Branching and Checkout

**Q5.1: Implementing `pes checkout <branch>`**

To implement `pes checkout <branch>`, these changes are needed:

- Read `.pes/refs/heads/<branch>` to get the target commit hash.
- Update `.pes/HEAD` to `ref: refs/heads/<branch>` (unless supporting detached mode explicitly).
- Resolve target commit → root tree, then materialize that tree into the working directory.
- Rewrite `.pes/index` so staged state matches the checked-out snapshot.

Working directory actions:

1. Create files/dirs present in target tree but missing locally.
2. Update files whose blob hash differs.
3. Remove tracked files that exist in current branch but not in target tree.

Why checkout is complex:

- It is a three-way consistency problem across current `HEAD`, target branch tree, and working directory state.
- It must avoid data loss from local staged/unstaged changes.
- Path conflicts (file↔directory changes across branches) require careful ordering of deletes/creates.
- The operation should be crash-safe because it mutates refs, index, and working tree together.

**Q5.2: Detecting dirty working directory conflicts**

Use only index + object store with this approach:

1. Load current `HEAD` tree and target branch tree.
2. For each path that differs between current and target trees, check if local path is dirty:
   - Compare working file metadata (`stat`) against index entry (`mtime`, `size`) for quick dirty detection.
   - If metadata differs, optionally re-hash file and compare to index blob hash for certainty.
3. Also detect staged-but-uncommitted changes by comparing index hash vs current `HEAD` tree hash.

Refuse checkout if a path is both:

- dirty in working directory or index, and
- changed by target branch checkout.

Also refuse if an untracked local file would be overwritten by a target tracked path.

**Q5.3: Detached HEAD behavior and recovery**

Detached HEAD means `.pes/HEAD` stores a raw commit hash, not `ref: refs/heads/<name>`.

If commits are made in this state:

- New commits are still created normally.
- Parent links are valid.
- But no branch file advances to reference those commits.

After switching back to a branch, those detached commits become unreachable by branch traversal (unless remembered).

Recovery options:

- Create a branch pointing to the detached commit hash (if known).
- Checkout that hash and reattach by creating/updating a branch ref.
- In full Git, reflog helps recover such hashes; a minimal PES without reflog requires user-recorded hashes.

### Garbage Collection and Space Reclamation

**Q6.1:** _TBD_

**Q6.2:** _TBD_
