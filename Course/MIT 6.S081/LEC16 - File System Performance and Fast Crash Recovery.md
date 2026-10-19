# LEC16 - File System Performance and Fast Crash Recovery

## Big Picture
- Case study: **ext3** = Linux's ext2 file system + a **journal** (journal = log).
- Logging is a near-"magic" way to add crash recovery to almost any storage system (databases, file systems, distributed systems).
- ext3 fixes xv6's logging performance with **asynchronous syscalls**, **batching**, and **concurrency** — at the cost of more complexity and changed crash semantics.

---

## 1. Review: xv6 Logging
- Writes go to the **buffer cache**; at `end_op`: copy modified blocks to the log → write **header block** (= **commit point**) → install to home locations → erase header.
- **Write-ahead rule**: all of an atomic group's updates must be in the log before **any** is applied to home locations.
- **Freeing rule**: a transaction's log space can't be **reused** until all its blocks are written to their home locations (and the header must be erased first, so stale block numbers can't be replayed onto new data).

**Why xv6's scheme is slow:**
- **Synchronous**: a syscall like `write`/`create` can't return until all of that work finishes (log writes, header, installs, erase). On a mechanical disk (~10 ms per write) → only a few modifying syscalls per second.
- While committing, **nothing else** can happen in the FS → syscalls effectively one at a time.
- Every block written **twice** (log + home).

## 2. ext3 Data Structures
**In memory:**
- **Write-back buffer cache**: blocks clean / dirty / **pinned** (can't be written home yet).
- Information about **multiple transactions** in different stages; each transaction has a **sequence number**, the set of **block numbers** it modified, and **handles** for its in-progress syscalls.

**On disk log (circular):**
```
[log superblock: offset + seq# of oldest valid transaction]
[descriptor block | data blocks ... | commit block] [descriptor | data ... | commit] ...
```
- **Descriptor block**: magic number, sequence number, home block numbers (like xv6's header).
- **Commit block**: separate block marking the transaction complete.
- Descriptor/commit blocks start with a **magic number** to distinguish them from data blocks.
- Only **one open transaction** (in memory) accepts syscalls; older closed transactions are being committed or installed; log space is freed **oldest first**, then reused (wraps around).

## 3. Performance Idea 1: Asynchronous System Calls
- Syscalls just modify **cached** blocks and **return** — never wait for disk writes (may wait for reads).
- Benefits:
  - Syscalls return quickly
  - **I/O concurrency**: app computes while the disk writes
  - Makes large **batching** easier
- **Cost**: syscall returning ≠ data on disk. Program creates file, writes, closes, prints "done", you pull the plug → data may be **gone**. In xv6, a returned `write` was durable; in ext3, unknown.
- Careful apps (databases, text editors — want old file or new file, never garbage) use **`fsync(fd)`**: "**flush** my writes to disk; don't return until they're durable". Most programs (e.g. compilers) don't bother.

## 4. Performance Idea 2: Batching
- One open transaction collects **all syscalls for ~5 seconds**, then commits as one big transaction (possibly hundreds of blocks).
- Wins:
  - **Amortize fixed costs** (descriptor + commit blocks, seek/rotate to the log) over many syscalls
  - **Write absorption**: many syscalls modify the **same blocks** (many 64-byte inodes per block; consecutive bitmap bits in one block) → modify in cache repeatedly, write once
  - **Disk scheduling**: writing many blocks **sequentially** (to the log) is much faster than one at a time; even home-location writes, handed over in a big batch, can be **sorted** (by track on a mechanical disk; smaller wins on SSDs)
- (xv6 has a little batching: concurrent syscalls in one group commit.)

## 5. Performance Idea 3: Concurrency
1. **Many syscalls in parallel** contribute to the current open transaction (important on multicore — no waiting for a committing transaction).
2. **Several transactions in different stages** at once:
   - one **open** (accepting syscalls)
   - some **committing** (writing to the log)
   - some **writing to home locations**
   - oldest being **freed**
- New syscalls don't wait for older transactions to commit or install (xv6 must).
- Problem: a newer transaction modifies a cached block while an older one is writing it to the log. → When closing a transaction, ext3 **(notionally) copies** its blocks; the commit writes **that snapshot**; newer transactions modify the real cached block. Done efficiently with **copy-on-write**.
- Result: multicore parallelism + overlap of CPU and disk I/O.

## 6. The API Inside ext3
```c
h = start();          // begin syscall's atomic group → handle
b = get(h, blockno);  // tell log: this block belongs to h's transaction
... modify b in the cache ...
stop(h);              // this syscall is done (does NOT commit)
```
- Handles let ext3 know which transaction each syscall belongs to; a transaction commits only after **all its syscalls have called `stop`**.
- Syscalls must **pre-declare** how many blocks they may need (so a transaction never starts that couldn't fit in the log → avoids deadlocks).

## 7. Committing a Transaction (every ~5 s, by a background kernel thread)
1. **Block new syscalls**
2. **Wait for outstanding syscalls** in this transaction to finish
3. **Open a new transaction**; unblock syscalls
4. Write **descriptor block** + **data blocks** (the saved snapshot) to the log
5. **Wait** for those writes
6. Write the **commit block**; **wait** → **commit point** (now survives a crash)
7. Write blocks to their **home locations**
8. When all are home: **free** that log space (update the log superblock to point to the new oldest transaction)
- If the log wraps around and catches the oldest transaction, new transactions **wait** for it to be installed and freed. Logs are sized so this is rare.
- Step 1 is a performance defect — system calls wait during that period.

## 8. Recovery
- Assumption: power failure; the **disk itself is intact** (only RAM is lost, incl. all bookkeeping).
1. Read **log superblock** → oldest valid transaction.
2. **Scan forward**: descriptor → expected number of data blocks → commit block → next descriptor ...
3. Stop at the **last complete transaction**: next block isn't a valid descriptor, or a descriptor has no valid commit block. Ignore a partial transaction at the end.
4. **Replay** all blocks of valid transactions to home locations, then start the OS.

**How to tell a stale block from a real descriptor/commit?**
- **Magic number** at the start.
- A **data** block that happens to start with the magic number: ext3 writes 0 there and sets a **flag** in the descriptor; recovery restores the magic number before replaying → only descriptor/commit blocks can start with the magic number.
- **Sequence numbers** in descriptor/commit blocks: e.g. T8 overwrote part of freed T5; T8 crashed before its commit, and T5's old commit block sits exactly where T8's would be → seq 5 ≠ 8 → not valid.
- Recovery replays only a few hundred blocks → **well under a second**, even for a huge FS.

## 9. Why Block New Syscalls While Closing a Transaction?
Example if ext3 didn't:
```
T1: create("x")  ......................... allocates inode 17 ... (T1 commits)
T2:      unlink("y") frees inode 17 ........ (not finished, crash!)
```
- `create` in T1 sees the free made by T2's `unlink` → reuses inode 17. T1 commits, crash before T2 commits.
- Recovery: T1 replayed, T2 ignored → `y` still uses inode 17, and `x` uses inode 17 too → **two files share an inode**.
- A **later** transaction's update leaked into an **earlier** one → atomicity broken.
- ext3's simple fix: no syscall from the new transaction starts until all syscalls of the previous one have finished.
- One of about a dozen similar subtle ordering cases ext3 must get right because of concurrency.

## 10. File Data: Journaling Modes
| Mode | What's logged | Trade-off |
|---|---|---|
| **Journaled data** | Metadata **and** file content | Simple, but data written twice → slower |
| **Ordered data** (most popular) | **Metadata only**; file content written directly to home location | Fast, but needs ordering |

- **Ordered mode danger**: allocate a new block for a file; inode update committed to log; crash before data written → after recovery the file contains **another user's old deleted data**.
- **Fix**: write the **new file content first**, and only after it's on disk **commit** the transaction that updates the inode.
- Crash after data, before inode commit: harmless — the block is still marked **free** in the bitmap (bitmap update wasn't committed either), so nothing is lost or exposed.

## 11. ext4: Faster Commits
- Combining descriptor + commit into one block saves a block, not time (still must wait for data before marking committed, and still need magic + sequence numbers).
- **ext4** writes data blocks **and** the commit block **together** without waiting; the commit block holds a **checksum** of the data blocks. If the disk reordered writes and crashed before the data landed, the checksum fails at recovery → transaction ignored. Saves a full disk rotation.

## 12. Takeaways
- Logging makes **multi-step disk updates atomic** with respect to crashes.
- Correctness depends on the **write-ahead rule**.
- Logging gives **fast recovery** (replay a short log).
- ext3 uses **batching + concurrency** (and async syscalls) for much better performance than xv6, with much more complexity.
- xv6's limitation in contrast: only **one transaction in the log** → must fully commit + install + erase before the next can proceed.
