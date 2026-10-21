
## 1. Purposes of an OS
| Purpose                   | Meaning                                                                                      |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| Abstract the hardware     | Give apps processes and file systems instead of raw CPU/RAM/disk (convenience + portability) |
| Multiplex                 | Run many programs (editor, compiler, DB) at the same time                                    |
| Isolation                 | Programs must not interfere unintentionally, even if buggy                                   |
| Sharing                   | ...but allow interaction when wanted (editor writes a file, compiler reads it)               |
| Security / access control | Prevent sharing when *not* wanted (other users can't read your files)                        |
| Performance               | Don't get in the way of apps using the hardware fully                                        |
| Range of uses             | Same OS (e.g. Linux) runs laptops, games, DB servers, cloud                                  |

## 2. Classic OS Organization
```
User: vi(text editor) cc(C compiler) sh(Shell) ... (processes)
----------------- system calls -----------------
Kernel: file system, processes, memory allocation,
              access control, TCP/IP, network stack, drivers...
-------------------------------------------------
Hardware: CPU   RAM   disk   Network
```
- **Kernel** = special program that **boots first**, is always running, and is the "guardian" of hardware resources.
- Kernel services: file system (file contents + name space/directories), **process** management (each running program has its own memory + share of CPU), memory allocation, access control.
- A full-blown kernel also has IPC, TCP/IP, drivers for hundreds of devices → **millions of lines** of code.
- This course focuses on **what happens inside the kernel** and the **interface between user programs and the kernel**.

## 3. System Calls
- Look like function calls, but **jump into the kernel**:
```c
fd = open("out", 1);          // returns a file descriptor
write(fd, "hello\n", 6);      // fd, address of buffer, byte count
pid = fork();                 // create a new process
```
- Difference from a normal function call: the kernel code runs **with special privileges** (direct access to hardware like the disk); a normal function gets no extra privilege.
- `open`, `write` in the C library are really **assembly stubs** that execute the RISC-V **`ecall`** instruction → transfers control to the kernel, which reads the arguments from the process's registers/memory.

> 💡 High-level languages (Python) wrap system calls for portability, but in the end a Python `open()` still turns into these system calls.

## 4. Why OS Design Is Hard and Interesting
- **Unforgiving environment**: you build the infrastructure everyone else assumes; underneath you is only hardware (QEMU helps).
- **Design tensions**:
	- Efficient (low-level, close to hardware) **vs** abstract, portable, easy to use
	- Powerful services **vs** simple interfaces
	- Flexible for programs **vs** secure (programs can't touch hardware or interfere)
- **Features interact**: e.g. `open` + `fork` — should the child see fds opened before `fork`? (Yes.)
- **Hardware and uses keep changing**: SSDs, multicore (pervasive for ~15 years), much faster networks → designs need rethinking.

## 5. xv6 Setup
- **xv6**: simplified Unix-like teaching OS, same overall structure as Unix but small enough to read all the source.
- Runs on a **RISC-V** CPU — in the course, emulated by **QEMU** (on Linux), with emulated memory, disk, console.
- `make qemu` → builds kernel + user programs → boots → `$` = the **shell**.
- `ls` shows ~two dozen files, including `ls` itself (a file of machine instructions).

## 6. read / write — copy.c
```c
char buf[64];
while (1) {
    int n = read(0, buf, sizeof(buf));
    if (n <= 0) break;
    write(1, buf, n);
}
exit(0);
```
- **File descriptor** 0 = console input, 1 = console output (the **shell sets these up** before the program starts — a pervasive Unix convention).
- `read(fd, buf, max)` returns:
	- number of bytes read
	- **0** at end of file
	- **-1** on error
- `read`/`write` don't care about data format — just a **stream of 8-bit bytes**; interpretation is up to the app.

> ⚠️ If you pass `max` larger than the buffer, the kernel happily writes past it (e.g. over the return address on the stack). C won't stop you — be careful, and **check syscall return values** (errors are usually -1).

## 7. open — open.c
```c
int fd = open("output.txt", O_WRONLY | O_CREATE);
write(fd, "ooo\n", 4);
```
- `open` creates/opens a file, returns a **new file descriptor** (small integer, e.g. 3).
- The fd **indexes a per-process table inside the kernel** that says what each fd refers to.
- Each process has **its own fd space** → the same number can mean different files in different processes.
- O_WRONLY: Write-only ; O_CREATE: Creates the file if it does not exist.

## 8. The Shell
- Command-line interface, used for system management, file manipulation, development, scripting.
- Typing `ls` = ask the shell to run the program in the **file** named `ls`.
- **I/O redirection**:
	- `ls > out` — output to file `out`
	- `grep x < out` — input from file `out`

## 9. fork — fork.c
```c
int pid = fork();
printf("fork() returned %d\n", pid);
if (pid == 0) printf("child\n");
else          printf("parent\n");
```
- `fork` creates a **copy of the calling process's memory** (instructions + data + stack).
- **Returns in both processes**:
  - Parent: child's **PID** (> 0)
  - Child: **0**
- Demo output was **interleaved character by character** — both processes run truly in parallel (QEMU emulates multiple cores).
- In xv6 parent and child are identical except fork's return value; each has its **own separate memory** (both think memory starts at 0).
- The **fd table is copied too** → child sees the same open files.

## 10. exec — exec.c
```c
char *argv[] = { "echo", "this", "is", "echo", 0 };
exec("echo", argv);
printf("exec failed!\n");
```
- Loads instructions from a file and **replaces the current process's memory**, then starts executing it.
- `argv` is an array of string pointers ending with **0 (null pointer)** — C arrays don't carry their length, so the kernel scans until the null.
- **Does not return** on success (nothing left to return to); returns **-1** only on error (e.g. file not found).
- **Preserves the fd table**.

## 11. fork + exec + wait — forkexec.c
```c
int pid = fork();             // From here on, there are two processes
if (pid == 0) {               // The child process goes here
    exec("echo", argv);       // It becomes echo, printing "this is echo"
    printf("exec failed!\n");
    exit(1);                  // If it fails, it ends with status 1
} else {
    int status;
    wait(&status);         // Stop and wait for the child process to finish
    printf("the child exited with status %d\n", status);
}
```
- The standard Unix idiom to run a program **and regain control**: fork, child execs.
- The shell does exactly this for every command; `wait` makes it wait before printing the next `$`.
- `exit(n)` → `n` delivered to parent's `wait(&status)`. Convention: **0 = success**, 1 = error.
- `wait` waits for **any** child, returns its PID; no children → returns **-1** immediately. Two children → call `wait` twice.
- A child **cannot wait for its parent**.
- Output order between parent and child is **not guaranteed** (exec is slow: reads file from disk, allocates memory).

> 💡 fork copies all memory only for exec to throw it away — wasteful for big programs. Later labs implement **copy-on-write fork** to avoid the copy.

## 12. I/O Redirection — redirect.c
```c
if (fork() == 0) {
    close(1);                                  // close fd 1
    open("output.txt", O_WRONLY | O_CREATE);   // guaranteed to return 1
    exec("echo", argv);                // echo this is echo > output.txt
}
```
- **`open` returns the lowest unused fd** → after `close(1)`, the file becomes fd 1.
- `echo` just writes to fd 1 — it has **no idea** about redirection; only the shell knows.
- Done in the **child**, so the shell's own fd 1 is unchanged.
- This is why **fork and exec are separate**: between them, the child still runs the shell's code and can rearrange fds.

## 13. Takeaways
- Interfaces are simple: pass integers (fds, PIDs) back and forth.
- Functionality behind them is sophisticated (creating/copying processes).
- Simple abstractions **combine** in useful ways (redirection).
