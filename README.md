# PES-VCS: Custom Version Control System

**Name:** Ramith M R
**SRN:** PES1UG24CS367
**Repository:** PES1UG24CS367-PES-VCS
**Platform:** Ubuntu 22.04

---

## Phase 1 — Object Storage

### Implementation Details

In this phase, the main goal was to store data in a structured way using hashing.

* **object_write**

  * Adds a header (`type + size`) before storing data
  * Computes SHA-256 hash using OpenSSL
  * Avoids duplicate storage by checking existing hashes
  * Saves objects in `.pes/objects/XX/...` format
  * Uses safe write method (temp file + rename)

* **object_read**

  * Reads stored object using its hash
  * Verifies integrity by recomputing hash
  * Extracts type and size from header
  * Returns actual data portion

### Screenshots

* **1A:** Output of `test_objects` (all tests passing)
  <img width="968" height="216" alt="Test_object_p1_1" src="https://github.com/user-attachments/assets/a6fadd55-6759-4db7-9f00-3bb5aad17d7d" />

* **1B:** Object storage structure using `find .pes/objects`
  <img width="954" height="98" alt="Test_tree_p1_2" src="https://github.com/user-attachments/assets/fd3fc849-1ee6-4be1-b69b-511c5d9728b6" />


---

## Phase 2 — Tree Objects

### Implementation Details

This phase focuses on building directory structure from staged files.

* Converts flat index paths into hierarchical tree structure
* Handles nested directories like `src/main.c`
* Ensures entries are sorted to maintain consistent hashing
* Uses recursion to build subdirectories
* Writes tree objects into object store

Additional functions provided:

* `tree_parse` for reading tree data
* `tree_serialize` for writing tree data

### Screenshots

* **2A:** Output of `test_tree`
  <img width="832" height="178" alt="Test_object_p2_1" src="https://github.com/user-attachments/assets/3d42723a-6b09-4127-95e8-229356f45fe7" />

* **2B:** Raw tree object using `xxd`
  <img width="1515" height="183" alt="Test_tree_p2_2" src="https://github.com/user-attachments/assets/6e90e3cd-36ea-4d00-980a-382be101f1c1" />


---

## Phase 3 — Index (Staging Area)

### Implementation Details

The index acts as a staging layer before committing.

* **index_load**

  * Reads `.pes/index`
  * Creates empty index if file doesn’t exist
  * Parses entries using `sscanf`

* **index_save**

  * Sorts entries before saving
  * Uses atomic write (temp file → rename)
  * Ensures data consistency using `fsync`

* **index_add**

  * Reads file content
  * Stores it as blob using `object_write`
  * Updates or creates index entry
  * Saves updated index

### Screenshots

* **3A:** `pes init → pes add → pes status`
  <img width="822" height="589" alt="Index_status_check" src="https://github.com/user-attachments/assets/ba41c3f2-008b-4754-92e2-ccf5d3f042ca" />

* **3B:** Contents of `.pes/index`
  <img width="951" height="79" alt="file_text_info_cat_pesindex" src="https://github.com/user-attachments/assets/3a4a70a5-f33a-40f3-bb86-514aa3fa261b" />


---

## Phase 4 — Commits and History

### Implementation Details

This phase connects everything into a version history.

* **commit_create**

  * Generates tree from index
  * Reads previous commit as parent
  * Stores author, timestamp, message
  * Serializes commit data
  * Writes commit object to storage
  * Updates branch reference

Other helper functions:

* `commit_parse`
* `commit_serialize`
* `commit_walk`
* `head_read`
* `head_update`

### Screenshots

* **4A:** Output of `pes log`
  <img width="783" height="413" alt="word_commits_P4_1" src="https://github.com/user-attachments/assets/8e4f8d74-53b2-46c8-baac-325b787a5435" />

* **4B:** Object files using `find .pes -type f`
  <img width="931" height="298" alt="commit_object_growth_P4_2" src="https://github.com/user-attachments/assets/50531ff6-e64f-453d-a947-bc10ef1d037c" />

* **4C:** HEAD and branch reference files
  <img width="930" height="92" alt="References_P4_3" src="https://github.com/user-attachments/assets/6a2fca90-808a-4305-8b02-92b15a8cd381" />


---

## Integration Test

Final test was executed using:

```bash
make test-integration
```
<img width="714" height="994" alt="Intergration_test_p4_4" src="https://github.com/user-attachments/assets/d8f61c2e-58ec-4179-854d-ff4d60b0a5b1" />
<img width="718" height="851" alt="Intergration_test_p4_5" src="https://github.com/user-attachments/assets/ff3160d0-8555-4a6f-8bc3-7dcef16f67dd" />



All test cases executed successfully.

---

## Summary

This project demonstrates how a basic version control system works internally:

* Files are stored using hash-based addressing
* Directory structure is handled through tree objects
* Index acts as a staging layer
* Commits create a linked history

Overall, the implementation helped in understanding how systems like Git manage data efficiently.


## Phase 5 — Branching and Checkout Analysis

### Q5.1 — How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, the following steps are needed:

**Files that need to change in `.pes/`:**
1. `.pes/HEAD` — Updated to contain `ref: refs/heads/<branch>` pointing to the new branch
2. The working directory — All tracked files must be updated to match the target branch's tree

**Algorithm:**
1. Read `.pes/refs/heads/<branch>` to get the target commit hash
2. Read the commit object to get its tree hash
3. Recursively walk the tree object, collecting all `(path, blob_hash)` pairs
4. For each file in the target tree: read the blob from the object store and write it to the working directory
5. For files that exist in the current branch's tree but NOT in the target tree: delete them from the working directory
6. Update `.pes/HEAD` to `ref: refs/heads/<branch>`

**What makes this complex:**
- **Dirty working directory detection** — If a file has been modified but not committed, we risk losing changes. Must check before overwriting
- **File deletions** — Files present in the current branch but absent in the target branch must be deleted from the working directory
- **Untracked files** — Files not tracked by either branch should be left alone
- **Partial failures** — If checkout fails midway (e.g. disk full), the working directory could be in a mixed state. Git uses a "checkout plan" that validates the entire operation before making any changes
- **Directory creation/deletion** — Subdirectories may need to be created or removed

---

### Q5.2 — How to detect "dirty working directory" conflicts?

To detect whether a file would conflict during checkout, using only the index and object store:

**Algorithm:**
```
For each file tracked in the current index:
    1. Read the file's current content from the working directory
    2. Compute its SHA-256 hash
    3. Compare to the hash stored in the index entry

    If they differ → the file has been modified since last "pes add"
    
    Then check: does this file also exist in the TARGET branch's tree?
    And does the target branch have a DIFFERENT blob hash for it?
    
    If YES to both → CONFLICT → refuse checkout and report the file
```

**Fast-path optimization (like Git's index):**
- Before hashing the file, compare `mtime` and `size` from `stat()` against what's stored in the index entry
- If `mtime` and `size` match, the file is almost certainly unchanged — skip re-hashing
- Only hash the file if metadata differs (this avoids reading every file on checkout)

**Decision matrix:**

| Working dir == index? | Index == target tree? | Action |
|---|---|---|
| Yes (clean) | Any | Safe to update |
| No (dirty) | Same hash in target | Safe (target has same content) |
| No (dirty) | Different hash in target | **CONFLICT — refuse** |
| No (dirty) | Not in target | **CONFLICT — refuse** |

---

### Q5.3 — What happens in "Detached HEAD" state?

**What is detached HEAD?**  
Normally `HEAD` contains `ref: refs/heads/main` (a symbolic reference). In detached HEAD, `HEAD` directly contains a commit hash, e.g. `a1b2c3d4...`.

**What happens when you commit in detached HEAD?**
- New commits are created and written to the object store normally
- `HEAD` itself is updated to point to each new commit
- But **no branch file is updated** — `refs/heads/main` stays at the old commit
- These commits are "dangling" — no branch points to them

**Danger:**  
When you switch back to a branch (e.g. `pes checkout main`), `HEAD` gets rewritten to `ref: refs/heads/main`. The commits made in detached HEAD are now **unreachable** — no reference points to them. They will eventually be deleted by garbage collection.

**How to recover:**  
Before switching away, note the commit hash from `cat .pes/HEAD`. Then create a branch pointing to it:
```bash
# While still in detached HEAD state:
cat .pes/HEAD         # note the hash, e.g. abc123...
# Create a branch to save it:
echo "abc123..." > .pes/refs/heads/recovery-branch
# Now checkout that branch:
# HEAD → refs/heads/recovery-branch → abc123...
```
Or after switching away, if you remember the hash, you can still create the branch before GC runs since the objects remain in the store until garbage collected.

---

## Phase 6 — Garbage Collection Analysis

### Q6.1 — Algorithm to find and delete unreachable objects

**Goal:** Find all objects in `.pes/objects/` that are NOT reachable from any branch tip, and delete them.

**Algorithm (Mark and Sweep):**

```
MARK phase:
1. Start from all branch tips:
   - List all files in .pes/refs/heads/
   - Each file contains a commit hash → add to reachable set

2. For each commit hash in the reachable set:
   - Read the commit object
   - Add its TREE hash to the reachable set
   - Add its PARENT hash to the reachable set (if any)
   - Repeat for the parent (BFS/DFS traversal)

3. For each tree hash in the reachable set:
   - Read the tree object
   - Add all BLOB hashes to the reachable set
   - Add all sub-TREE hashes to the reachable set
   - Recurse for sub-trees

SWEEP phase:
4. List ALL files in .pes/objects/ (walk the shard directories)
5. For each object file found:
   - Reconstruct its hash from the path (shard dir + filename)
   - If the hash is NOT in the reachable set → DELETE the file
```

**Data structure:** A **hash set** (implemented as a hash table or sorted array of 32-byte hashes). Lookup is O(1) average for hash table. Each entry is just 32 bytes.

**Estimate for 100,000 commits, 50 branches:**
- Commits: ~100,000
- Trees: ~100,000 (one per commit, assuming no sharing)
- Blobs: assume average 20 files per commit = ~2,000,000 unique blobs (with deduplication probably ~500,000)
- **Total objects to visit: ~600,000–2,100,000**
- Memory for reachable set: 600,000 × 32 bytes = ~19 MB (very manageable)

---

### Q6.2 — Race condition between GC and concurrent commit

**The race condition:**

```
Thread A (Commit):                    Thread B (GC):
1. object_write(blob) → stores blob   
2. ...computing tree...               
                                      3. GC scans reachable objects
                                         (blob not referenced by any commit yet)
                                      4. GC sees blob as UNREACHABLE
                                      5. GC deletes the blob file
6. object_write(tree) → references 
   the now-deleted blob!
7. object_write(commit) → 
   repository is NOW CORRUPT
```

**Why this is dangerous:**  
The blob was written but not yet referenced by any commit at the time GC ran. GC correctly identifies it as unreachable and deletes it. The commit that was about to reference it now points to a non-existent object — **silent corruption**.

**How Git avoids this:**

1. **Grace period (clock-based):** Git's GC never deletes objects younger than 2 weeks (default). Since a commit operation completes in milliseconds, any object written recently is safe. This is a simple and robust solution.

2. **Quarantine directory:** In Git's `receive-pack`, newly received objects are written to a quarantine directory first. They are only moved to the real object store after the ref is atomically updated. GC never touches the quarantine area.

3. **Lock files:** Git uses `.lock` files for atomic ref updates. GC can check for in-progress operations by looking for lock files before running.

4. **Two-phase GC:** Git's `git gc` first does a dry-run to identify candidates, waits, then deletes. Objects that appear between the two scans are protected by the grace period.

**Key insight:** The safest solution is the **grace period** — objects newer than N minutes/hours/days are never deleted, even if they appear unreachable. This costs minimal extra storage but completely eliminates the race condition in practice.

---

## Implementation Notes

### Key Design Decisions

| Decision | Rationale |
|---|---|
| Heap-allocate `Index` struct | `Index` can be several MB — stack allocation causes SIGSEGV |
| `tree_load_index_inline` in `tree.c` | `test_tree` doesn't link `index.o`, so `tree.c` reads index directly |
| `mkstemp` for temp files | Safer than fixed temp names — avoids collisions in concurrent use |
| OpenSSL EVP API for SHA-256 | Non-deprecated on OpenSSL 3.0+ (replaces `SHA256_Init/Update/Final`) |
| `fsync` before `rename` | Guarantees data on disk before atomic pointer update |

### File Structure

```
PES1UG24CS367-PES-VCS/
├── object.c        ← Phase 1: Content-addressable object store
├── tree.c          ← Phase 2: Tree serialization and construction  
├── index.c         ← Phase 3: Staging area implementation
├── commit.c        ← Phase 4: Commit creation and history
├── pes.c           ← CLI entry point (provided, not modified)
├── pes.h           ← Core data structures (provided, not modified)
├── tree.h          ← Tree interface (provided, not modified)
├── index.h         ← Index interface (provided, not modified)
├── commit.h        ← Commit interface (provided, not modified)
├── test_objects.c  ← Phase 1 test (provided, not modified)
├── test_tree.c     ← Phase 2 test (provided, not modified)
├── test_sequence.sh← Integration test (provided, not modified)
├── Makefile        ← Build system (provided, not modified)
└── README.md       ← This report
```
