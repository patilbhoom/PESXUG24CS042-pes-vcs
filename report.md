# PES-VCS Analysis Lab Report

## Phase 5: Branching and Checkout

### Q5.1: Implementing pes checkout <branch>
To implement `pes checkout <branch>`, the `.pes/HEAD` file must be updated to contain `ref: refs/heads/<branch>`.
In the working directory, the files must be updated to exactly match the snapshot stored in the tree of the commit that the new branch points to. This means:
1. Deleting files that are in the current commit but not in the target commit.
2. Modifying files that differ between the current and target commit.
3. Creating files that are present in the target commit but not in the current commit.
4. The `.pes/index` must also be completely rewritten to reflect the target commit's tree.

This operation is complex because it modifies the user's actual filesystem state. It requires a recursive diff between the current index/working tree and the target commit's tree. Furthermore, we must carefully avoid overwriting unsaved user work (uncommitted changes), which leads to the dirty working directory checks.

### Q5.2: Detecting a "dirty working directory" conflict
To detect a dirty working directory conflict before a checkout:
1. Walk the target branch's commit tree and the current index side by side.
2. For each tracked file in the index, use `stat` (or metadata like `st_size` and `st_mtime`) to check if the version in the working directory differs from the index.
3. If it differs (meaning the user has unstaged changes) or if the index hash differs from the current HEAD (staged changes), and that particular file *also* has a different hash in the target branch's tree, then a conflict exists.
4. If such a conflict is found, checkout must refuse to proceed because checking out the branch would overwrite the user's uncommitted modifications to that file, causing data loss.

### Q5.3: Detached HEAD state
When committing in a "Detached HEAD" state (where `.pes/HEAD` contains a raw commit hash instead of a branch reference), the new commit object is created, and `.pes/HEAD` is updated to point to the new commit's hash. However, no branch reference file (in `refs/heads/`) is updated.
If you later checkout another branch, there will be no branch pointer referencing those commits made in the detached state, making them "unreachable".
To recover those commits, the user would need to find their hash (e.g., using `pes reflog` in real Git, or searching the object store / terminal history) and create a new branch pointing to that hash, essentially "attaching" a name to the detached timeline.

## Phase 6: Garbage Collection and Space Reclamation

### Q6.1: Garbage Collection Algorithm
**Algorithm:**
1. Collect all "roots": Read every reference in `.pes/refs/heads/` and the `.pes/HEAD` file to get the starting commit hashes.
2. Perform a Graph Traversal (BFS or DFS) starting from these roots:
   - For each commit, mark it reachable. Parse it to find its tree hash and parent commit hash(es). Enqueue the parent commits and the tree.
   - For each tree, mark it reachable. Parse it to find the hashes of its blobs and sub-trees. Enqueue them.
   - For each blob, mark it reachable.
3. Once the traversal is complete, iterate over every object file stored in `.pes/objects/`.
4. If an object file's hash is not marked as reachable, delete it.

**Data Structure:**
A hash set (or bloom filter + hash set for memory efficiency) is the best data structure to track "reachable" hashes efficiently, as it allows O(1) average time complexity for checking if a hash has already been seen.

**Estimation:**
For 100,000 commits, you need to visit 100,000 commit objects. Assuming an average of 1 tree per commit (though they share trees, let's say there are 10,000 unique trees) and maybe 50,000 unique blobs, you might visit roughly 150,000 to 200,000 unique objects. The 50 branches just serve as starting points for the traversal; the majority of commits are likely shared ancestors.

### Q6.2: Concurrent GC and Commit Operations
It is dangerous because a race condition could lead to data loss.
**Race Condition:**
1. The user stages a new file. The file is written to the object store as a blob `A`.
2. GC starts running. It scans the branch refs and index to find reachable objects. Because `A` is not yet referenced by a commit or the index (if the index wasn't scanned properly, or it was just written), GC determines `A` is unreachable.
3. GC deletes blob `A`.
4. The user runs `pes commit`, which creates a tree and commit pointing to `A`.
5. The commit is now corrupt because the blob `A` it depends on was deleted by GC.

**How Git avoids this:**
Real Git GC uses a two-week grace period by default. It only deletes unreachable objects that have a modification time (mtime) older than 2 weeks. This ensures that any newly created objects that aren't yet attached to a branch (like objects staged or currently being written) are not prematurely deleted.
