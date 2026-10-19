# LEC14 - File Systems

## Big Picture
- File systems give **user-friendly hierarchical names**, **sharing**, and **persistence** (files survive reboot — unlike processes).
- Interesting parts: **crash safety** (next lecture), **on-disk layout** (data structures on disk), **performance** (storage is slow → caching, concurrency).
- This lecture: what the API implies, xv6's layers, disk layout, inodes, directories, and the **buffer cache**.

---

## 1. What the API Implies
```c
fd = open("x/y", O_CREATE);
write(fd, "abc", 3);
link("x/y", "x/z");     // second name for the same file
unlink("x/y");          // remove a name while file is open
write(fd, "def", 3);    // still works!
```
- Path names are **human-readable**, chosen by users.
- `write` has **no offset argument** → the FS keeps an **implicit offset** per open file.
- A file can have **multiple names** (hard links).
- The namespace can change while a file is open → the open fd must refer to an **object independent of its name**.
- → Internal representation: the **inode**.

> 💡 A file system isn't the only way to store data persistently — databases offer a very different API (and usually sit on top of a file system).

## 2. Inode and File Descriptor
- **Inode** = the file, independent of names; identified by an **integer (inode number)**.
- Inode keeps a **link count** (names pointing to it). Also need the count of **open file descriptors**. Delete the file only when **both are 0**.
- **File descriptor** holds the **offset**.
- `link`/`unlink` operate on inodes.

## 3. Layers
```
file descriptors
path names
directories
inodes (read/write bytes)
inode cache  (synchronization; several inodes packed in one block)
logging      (crash safety — next lecture)
buffer cache (block cache: avoid going to disk)   ← memory
disk                                              ← persistence
```
Every FS is organized a bit differently, but most have these components.

## 4. Storage Devices
| Device | Access time per block |
|---|---|
| **SSD** | ~100 µs – 1 ms |
| **Magnetic disk** | ~10 ms |
- **Sector**: smallest unit the disk reads/writes (historically 512 B). **Block**: unit chosen by the FS — xv6: **1024 B = 2 sectors**. (Terms used loosely.)
- Connected via a bus (PCIe). Interface ≈ **read/write block number N**. Very different internally, but look similar to the driver.
- Driver works like the console driver: issue command → device **interrupts** when done (top/bottom halves).
- FS view: the disk is a **big array of blocks**; the FS lays out data structures so it can reconstruct everything after reboot.

## 5. xv6 Disk Layout
```
block: 0      1       2 ... 31   32 ... 44   45       46 ...
      [boot][super][   log    ][  inodes  ][bitmap][  data blocks  ]
```
| Region | Contents |
|---|---|
| Boot block | Code to start booting the OS (QEMU's `-kernel` flag skips this) |
| **Superblock** | Describes the FS: number of blocks, where log/inodes/bitmap start |
| **Log** | 30 blocks (next lecture) |
| **Inodes** | **64 bytes each**, 16 per block |
| **Bitmap** | 1 bit per data block: free or used |
| **Data** | File and directory contents |
- Everything except data = **metadata** blocks.
- `mkfs` output: 46 meta blocks (boot, super, 30 log, 13 inode, 1 bitmap), 954 data blocks → ~1000 blocks total, tiny.
- Find inode N: block = 32 + (N × 64) / 1024 → inode 0–15 in block 32, inode 17 in block 33.

## 6. On-Disk Inode (64 bytes)
```c
struct dinode {
  short type;           // free / file / directory / device
  short major, minor;
  short nlink;          // link count
  uint  size;           // bytes
  uint  addrs[12+1];    // 12 direct + 1 indirect block number
};
```
- **12 direct** block numbers → first 12 blocks of the file.
- **1 indirect** block number → a block holding **256** more block numbers (1024 / 4-byte block numbers).
- **Max file size = (12 + 256) × 1 KB = 268 KB** — a very short song.
- Bigger files: e.g. a **doubly-indirect** block (like multi-level page tables) — the FS lab adds this. Real FSes may use B-trees, and > 32-bit block numbers.

**Reading byte 8000:**
- `8000 / 1024 = 7` → direct entry `addrs[7]` (≥ 12 would mean: read the indirect block, entry `bn − 12`)
- offset within block: `8000 % 1024 = 832`
→ enough to implement `read`/`write`.

## 7. Directories
- A directory is a **file** with a structure the FS understands — an array of 16-byte entries:
```c
struct dirent {
  ushort inum;      // 2 bytes
  char name[14];    // 14 bytes
};
```
- Inode `type` says file vs directory (looking up inside a file → error).

**Path lookup `/y/x`:**
1. Root directory = **inode 1** (fixed) → in block 32, bytes 64–127
2. Scan root's data blocks (via its `addrs`) for an entry named `y` → inode number, e.g. 251
3. Read inode 251, scan its blocks for `x` → inode number → result
- Linear scan is slow; real FSes use better on-disk data structures.
- 2-byte inode numbers (max ~65K inodes) are fine for the 60s/70s, too small today. In memory xv6 uses a 4-byte int.

## 8. Demo: Disk Writes for `echo hi > x`
(xv6 modified to print every block write)

**Create `x`:**
| Write | Why |
|---|---|
| 33 | Allocate inode: mark type free → file |
| 33 | Initialize the inode (nlink = 1, ...) |
| 46 | Root directory's first data block: add entry `x` → inode number |
| 32 | Root inode (inode 1): **size** grew by 16 bytes |
| 33 | Update `x`'s inode again |

**Write `"hi"`:**
| Write | Why |
|---|---|
| 45 | **Bitmap**: allocate a free data block |
| 595 | Data block: `h` (block 595 because mkfs already filled earlier blocks) |
| 595 | Data block: `i` (echo writes char by char) |
| 33 | `x`'s inode: size, `addrs[0] = 595` |

**Write `"\n"`:** 595, then 33 again.

## 9. Code: Allocating an Inode
- `sys_open` → `create()` → look up the parent directory, check the name doesn't exist → **`ialloc()`**.
- `ialloc`: for each inode number 1..N: `bread` its block, if type is free → set type, `log_write` (→ the first write above). Simple, not efficient.
- What if several processes create files concurrently on different cores? → buffer cache locking.

## 10. Buffer Cache (`bio.c`)
```c
struct buf *bget(uint dev, uint blockno) {
  acquire(&bcache.lock);
  for (b = bcache.head.next; ...)            // already cached?
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  // not cached: recycle least recently used buffer with refcnt == 0
  ...
}
```
**Two levels of locking:**
- **`bcache.lock`** (spinlock): protects the cache's own data structures (which blocks are cached, refcounts).
- **Per-buffer sleep lock**: protects the **contents** of one buffer — only one process reads/writes it at a time.
- After `refcnt++` and releasing `bcache.lock`, the buffer **can't be evicted** (only `refcnt == 0` buffers are recycled).
- Concurrent `ialloc`s on block 33: both find it, both bump refcnt, **one gets the sleep lock**, the other waits — then sees the first one's changes (e.g. inode 18 now taken → picks 19).

**Invariant: each block is cached at most once.** Otherwise two processes could modify different copies (inode 18 vs inode 19 in block 33) and one write-back overwrites the other's change.

**Why sleep locks?**
- Spinlocks turn **interrupts off** → holding one during a disk operation, a single core would never hear the disk's completion interrupt.
- Can't **sleep** (call `swtch`) holding a spinlock.
- Sleep locks can be held **across I/O, for a long time**; waiters **sleep** instead of spinning.
```c
void acquiresleep(struct sleeplock *lk) {
  acquire(&lk->lk);           // internal spinlock
  while (lk->locked)
    sleep(lk, &lk->lk);
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}
```

**`brelse`:**
- Release the sleep lock, then (under `bcache.lock`) `refcnt--`.
- If `refcnt == 0`: move buffer to the head = **most recently used**.
- Eviction picks the **least recently used** → **LRU**, good because of **temporal locality**.
- Decrementing after releasing the sleep lock is safe: if others are waiting, refcnt > 1, never hits 0.

## 11. Summary
- A file system = **on-disk data structures** (xv6's are very simple).
- **Buffer cache** is crucial for performance (disk ops take 100s of µs to ms): one copy per block, sleep locks across I/O, LRU replacement, two-level locking.
