# LEC15 - Crash Recovery

## Big Picture
- Problem: a **crash** (power failure, kernel panic) in the middle of a **multi-step file system operation** can leave the **on-disk** FS **inconsistent** (a block in two files, an inode used twice...).
- (Not about disks dying/catching fire — that needs backups.)
- Solution: **logging** (from the database world) → FS operations become **atomic**, recovery is **fast**.
- xv6's log is minimal and slow; ext3 (next lecture) shows a fast one. Last xv6 lecture — papers from now on.

---

## 1. What Can Go Wrong
After a crash, **only what's on disk survives** — memory (processes, fds, caches) is gone.

**Creating a file** (`echo hi > x`): write 33 (allocate inode) → 33 (init inode) → 46 (directory entry) → 32 (root inode size) → 33.
- Crash after allocating the inode, before the directory entry → inode allocated but in no directory → **lost inode** (can't even delete it).
- Reorder (directory entry first, then mark inode allocated)? Crash in between → directory points to an inode marked **free** → it gets **allocated to another file** → two files share one inode → user 2 may read user 1's data. **Worse.**

**Writing to a file**: 45 (bitmap: allocate block 595) → 595 (data) → 33 (inode points to 595).
- Crash after bitmap, before inode → **lost data block**.
- Reorder (inode first, then bitmap)? Crash in between → inode uses block 595 but bitmap says **free** → another file gets it too → **block shared by two files** → security problem.

→ Reordering doesn't help. The real issue: **several writes must appear on disk atomically**.

Consequences of broken invariants: crash again after reboot, or silently **lose/mix data**.

## 2. Logging: Goals
1. **Atomic** FS system calls: all of a call's writes happen, or none
2. **Fast recovery**: no need to scan the whole disk (unlike `fsck`-style checkers)
3. Can be **high performance** (not in xv6)

## 3. The Basic Scheme
Disk = **log** region + **file system** region.
1. **Log writes**: write updated blocks into the **log**, remembering each one's **home location** (e.g. "this goes to 45")
2. **Commit**: write a **commit record** (log header) at a fixed location with the **count** of logged blocks
3. **Install**: copy blocks from the log to their **home locations**
4. **Clean**: set the count back to **0**

**Recovery on reboot**: read the header. `n == 0` → nothing. `n > 0` → re-install all `n` blocks, then clean.

**Crash cases:**
| Crash point | After recovery |
|---|---|
| Before commit | Nothing installed → as if the syscall never ran |
| After commit, before/during install | Recovery re-installs everything → as if the syscall completed |
| During install / before clean | Installs again — safe because install is **idempotent** (writing the same value to the same block again does no harm) |

- **Commit point** = writing the header block. Assumption: a **single sector write is atomic** (all or nothing).
- **Write-ahead rule**: never install (write to home location) until **all** of the operation's writes are in the log and committed.
- Many logging variants exist (postpone install, postpone cleaning...), but all follow the write-ahead rule.
- (An "append" isn't idempotent, but the log operates on block writes; append is a higher-level FS operation made atomic by the log.)

## 4. xv6 Log Structure
```
on disk:   [ header: n, block#[0..n-1] ][ data 0 ][ data 1 ] ... [ data n-1 ]
           block 2                       blocks 3 ... 31
in memory: struct logheader { n; block[] }   + the actual data lives in the buffer cache
```

## 5. Code
### Transactions: begin_op / end_op
```c
// every FS system call:
begin_op();
  ... FS code ...   // writes go to buffer cache + in-memory log header
end_op();           // commit happens here
```
All block writes between them form one **atomic** transaction.

### log_write (replaces bwrite)
```c
void log_write(struct buf *b) {
  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++)
    if (log.lh.block[i] == b->blockno) break;   // already logged → "log absorption"
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {
    bpin(b);          // keep it in the buffer cache
    log.lh.n++;
  }
  release(&log.lock);
}
```
- The block is already updated in the buffer cache; `log_write` just records "this block must go into the log at commit".
- Every `bwrite` in the FS is replaced by `log_write`.

### commit
```c
static void commit() {
  if (log.lh.n > 0) {
    write_log();      // copy modified blocks from cache → log region
    write_head();     // write header with n  ← THE COMMIT POINT
    install_trans();  // copy log blocks → home locations
    log.lh.n = 0;
    write_head();     // clean the log
  }
}
```
- `write_head`: one `bwrite` of the header block. Crash before → transaction didn't happen; crash after → recovery will finish it.
- (Code could be optimized — e.g. unnecessary reads — kept simple.)

### Recovery
- At boot: `initlog` → `recover_from_log`: `read_head` → `install_trans` → clear log. Same `install_trans` as normal. If it crashes during recovery, it simply runs again next boot.

## 6. Demo: Real Disk Writes for `echo hi > x`
Printing in `bwrite` instead of `log_write` shows many more writes. For the create:
```
3, 4, 5     write_log: logged blocks into log region
2           write_head: COMMIT
33, 46, 32  install_trans: blocks to home locations
2           clean the log (n = 0)
```
(`h`+`i` were one write; the newline another transaction.)
- Every block is written **twice** → ~**2× slower**. Read the ext3 paper asking: how do they avoid this?

## 7. Complications
### (1) Eviction
- If the buffer cache **evicts** a logged-but-uncommitted block, it writes it to its **home location** → violates write-ahead → atomicity broken by a crash.
- Fix: **`bpin`** (refcnt++) in `log_write` → never evicted; **unpinned** after install.
- If the cache is full of pinned blocks and needs a buffer → xv6 **panics**. Can't just return an error: earlier writes of the same syscall would have to be **undone**.

### (2) Operations must fit in the log
- Log = **30 blocks**. An operation writing more would have to spill to home locations → not allowed.
- 30 was chosen by checking the max blocks any FS operation writes (most write only a handful).
- **Big `write`** (e.g. 1 MB = 1000 blocks): `filewrite` **splits** it into many small transactions. The whole `write` isn't atomic — Unix doesn't require that — but each piece is, so the **FS invariants** always hold (e.g. allocating a block + updating the inode is atomic).
- Pinned blocks → buffer cache must be bigger than the log.

### (3) Concurrent FS calls
- Several transactions write into the same log. If the log fills up before any finishes, **none can commit** (partial transaction = write-ahead violation).
- → All **concurrent** operations together must fit in the log.
- `begin_op` admission control:
```c
void begin_op(void) {
  acquire(&log.lock);
  while (1) {
    if (log.committing)
      sleep(&log, &log.lock);          // can't add while installing
    else if (log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE)
      sleep(&log, &log.lock);          // might not fit → wait
    else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```
- `end_op`: `outstanding--`; when it reaches **0**, **commit all of them together** = **group commit**. Then wake up waiters.
- Why not commit a finished transaction early? Transactions must be committed **in the order they happened** — reordering writes could produce behavior user programs can observe as wrong (e.g. a read saw write A, then did write B; B must not land without A).

## 8. Summary
- **Logging** makes multi-step FS operations crash-safe: write-ahead rule, single-sector commit, idempotent replay.
- Works great for **crash safety**; **performance** is the weak point → ext3 next lecture.
- Debugging concurrency (from the lock lab Q&A): **assertions for invariants**, print statements; deadlocks are easier (panic + gdb backtrace); race detectors exist for shared variables accessed without locks.
