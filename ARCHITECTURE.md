# System Architecture & Laboratory Design

This document describes the architectural foundations, Linux kernel subsystems, process management lifecycles, and memory hierarchy demonstrated across the practical experiments in this repository.

---

## 1. High-Level Operating System Architecture

The Linux operating system follows a monolithic kernel architecture with modular subsystem separation. Applications run in unprivileged User Mode (Ring 3) and interact with the kernel through the System Call Interface (SCI) to execute privileged operations in Kernel Mode (Ring 0).

```mermaid
flowchart TD
    subgraph UserSpace["User Space (Ring 3 - Unprivileged)"]
        Apps["User Applications & C Programs (exp7.c, exp8.c, cpu.c, exp10.c)"]
        Shell["GNU Bash Shell & POSIX Utilities (ls, pwd, sed, awk, tr, top)"]
        LibC["GNU C Standard Library (glibc - printf, malloc, free, signal)"]
    end

    subgraph SCI["System Call Interface (SCI)"]
        SysCalls["fork(), wait(), exit(), getpid(), brk(), sbrk(), mmap(), write(), read()"]
    end

    subgraph KernelSpace["Kernel Space (Ring 0 - Privileged)"]
        ProcessMgr["Process & Thread Scheduler (CFS)"]
        MemMgr["Virtual Memory Manager (Paging, MMU, Page Tables)"]
        VFS["Virtual File System (VFS & Ext4)"]
        NetStack["Network Stack (TCP/IP, Raw Sockets, Netfilter)"]
        DeviceDrivers["Device Drivers (Disk, CPU, TTY, Network NIC)"]
    end

    subgraph Hardware["Hardware Layer"]
        CPU["CPU (x86_64 Cores, TLB, Cache)"]
        RAM["Physical RAM"]
        Storage["Storage / Virtual Disk"]
        NIC["Network Card / Interfaces"]
    end

    Apps --> LibC
    Shell --> LibC
    LibC --> SysCalls
    Apps -.-> SysCalls
    SysCalls --> ProcessMgr
    SysCalls --> MemMgr
    SysCalls --> VFS
    SysCalls --> NetStack
    ProcessMgr --> CPU
    MemMgr --> RAM
    VFS --> Storage
    NetStack --> NIC
```

---

## 2. Process Lifecycle & State Transition Architecture

In Linux, process execution follows a defined lifecycle managed by the Completely Fair Scheduler (CFS). Experiments 4, 9, and 10 explicitly exercise and diagnose these state transitions.

```mermaid
stateDiagram-v2
    [*] --> New: fork() system call (Experiment 10)
    New --> Ready: Admitted to Scheduler Run-Queue
    Ready --> Running: Scheduler Dispatch (CPU allocated)
    Running --> Ready: Time slice expired / Preempted
    Running --> Waiting: I/O or sleep(1) requested (Experiment 9 io.c / wait)
    Waiting --> Ready: I/O complete / Timer elapsed
    Running --> Terminated: exit(0) / SIGINT Ctrl+C / kill PID
    Terminated --> [*]: wait(NULL) reaps child PCB (Experiment 10)
```

### Parent-Child Process Synchronization Flow (`exp10.c`)

```mermaid
sequenceDiagram
    autonumber
    actor Terminal as User / Terminal
    participant Parent as Parent Process (PID: 7428)
    participant Kernel as Linux Kernel (Scheduler & PCB)
    participant Child as Child Process (PID: 7429)

    Terminal->>Parent: ./exp10 execution starts
    Parent->>Kernel: fork() system call
    Kernel-->>Parent: Returns Child PID (7429)
    Kernel-->>Child: Returns 0 (Process cloned)
    
    par Concurrent Execution
        Parent->>Terminal: Prints "Parent PID: 7428"
        Parent->>Kernel: wait(NULL) [Blocks Parent]
    and
        Child->>Terminal: Prints "Child PID: 7429"
        Child->>Terminal: Prints "Child process is running"
        Child->>Kernel: sleep(2)
        Kernel-->>Child: Timer wake up
        Child->>Terminal: Prints "Child process terminated"
        Child->>Kernel: exit(0)
    end

    Kernel-->>Parent: Unblocks wait() and delivers termination status
    Parent->>Terminal: Process exits cleanly (No zombies)
```

---

## 3. Virtual Memory Architecture & Address Space Layout

Linux allocates each 64-bit user-space process a private 48-bit canonical virtual address space (0x000000000000 to 0x7FFFFFFFFFFF). This layout was analyzed in **Experiment 8** (`malloc`, `free`) and **Experiment 9** using `pmap -x`.

```mermaid
flowchart TD
    subgraph VirtualAddressSpace["Process Virtual Address Space (64-bit Linux)"]
        KernelSeg["Kernel Space (Reserved - 0xFFFF800000000000+)"]
        StackSeg["Stack Segment (Local variables, function frames) - Grows DOWN ⬇️"]
        Gap1["Unallocated Virtual Address Space"]
        MmapSeg["Memory Mapping Segment (Shared Libraries: libc.so, ld.so)"]
        Gap2["Unallocated Virtual Address Space"]
        HeapSeg["Heap Segment (Dynamic allocation via malloc/brk/sbrk) - Grows UP ⬆️"]
        BSSSeg["BSS Segment (Uninitialized global and static data)"]
        DataSeg["Data Segment (Initialized global and static variables)"]
        TextSeg["Text Segment (Binary machine instructions - Read-Only & Executable)"]
    end

    TextSeg --> DataSeg --> BSSSeg --> HeapSeg -.-> Gap2 -.-> MmapSeg -.-> Gap1 -.-> StackSeg --> KernelSeg
```

### Memory Allocation & Inspection Workflow (`exp8.c`)

```mermaid
sequenceDiagram
    participant User as CLI Invocation (./exp8 100)
    participant Program as exp8.c User Space
    participant LibC as glibc (malloc/free)
    participant Kernel as Linux Virtual Memory Manager
    participant Tools as Diagnostics (free -h / ps aux)

    User->>Program: Run ./exp8 100
    Program->>LibC: malloc(100 * 1024 * 1024)
    LibC->>Kernel: mmap() / brk() syscall
    Kernel-->>LibC: Allocated Virtual Pages (VIRT increases)
    Program->>Program: Touch pages (Write bytes to memory)
    Note over Program,Kernel: Triggers Page Faults -> Physical frames mapped into RSS
    Tools->>Kernel: Query free -h & ps aux
    Kernel-->>Tools: Shows 100 MB active RSS consumption
    User->>Program: Press Ctrl+C (Sends SIGINT)
    Program->>Program: handle_sigint() invoked
    Program->>LibC: free(ptr)
    Program->>User: "Memory released" and exits cleanly
```

---

## 4. Laboratory Experiments Architecture Mapping

| Layer | Experiments | Core Operating System Mechanism |
| :--- | :--- | :--- |
| **System Virtualization** | **Experiment 1** | Type-2 Hypervisor VirtualBox, Hardware Virtualization, ISO Bootloader, Root FS Installation. |
| **POSIX Shell & Filesystem** | **Experiment 2(a), 2(b)** | VFS Inodes, Directory Trees, Path Resolution (`pwd -P/-L`), POSIX file manipulation (`mkdir`, `rm`, `touch`). |
| **I/O Redirection & Text Streams** | **Experiment 3(a), 3(b)** | Standard Streams (`stdin=0, stdout=1, stderr=2`), Pipes (`pipe()`), Regex pattern capture groups with `sed`. |
| **Process Control & Monitoring** | **Experiment 4** | Process Control Block (`/proc`), Signals (`SIGKILL`, `SIGTERM`), Background jobs (`&`), `top`, `ps aux`. |
| **Shell Programming & Automation** | **Experiment 5(a), 5(b)** | Shell script parameter expansion (`$1`), file system test operators (`-f`, `-d`), character translation (`tr`). |
| **User Accounting & Advanced Sed**| **Experiment 6(a), 6(b), 6(c)** | User login accounting (`/var/run/utmp`, `who`, `awk`), parameterized stream editing, multi-file iteration. |
| **Benchmarking & Performance** | **Experiment 7** | Compiled native C execution vs Interpreted shell overhead, POSIX `time` metrics (User vs Sys vs Real). |
| **Dynamic Memory & Signals** | **Experiment 8** | Dynamic heap allocation (`malloc`), Resident Set Size (RSS), Signal Interception (`signal(SIGINT)`). |
| **Scheduler & Resource Analysis** | **Experiment 9** | CPU-bound vs I/O-bound process scheduling, CPU burst behavior, Virtual Memory Mapping examination (`pmap -x`). |
| **Process Hierarchy & System Calls** | **Experiment 10** | Process creation (`fork()`), Process Identification (`getpid()`), Parent-Child synchronization (`wait()`, `exit()`). |

---

## 5. Cross-Disciplinary Architecture: Operating Systems & Cyber Security Synergy

The mechanisms provided by the operating system serve as the direct foundation for foundational cyber security concepts:

```mermaid
flowchart LR
    subgraph OS["Operating System Foundations"]
        Mem[Memory Architecture & Stack/Heap]
        Proc[Process Isolation & Credentials]
        Pkt[Raw Sockets & Network Drivers]
        Sys[Kernel System Calls & Logs]
    end

    subgraph Cyber["Foundational Cyber Security (FOCS)"]
        BOF["Buffer Overflow & Stack Smashing (FOCS Exp 2)"]
        Sniff["Packet Capture & Protocol Inspection with Wireshark (FOCS Exp 4)"]
        Scan["Host Discovery & Port Scanning with Nmap (FOCS Exp 3)"]
        IDS["Intrusion Detection & Traffic Analysis with Snort 3 (FOCS Exp 8)"]
        Auth["Brute-Force & Credential Auditing with Hydra (FOCS Exp 5)"]
    end

    Mem --> BOF
    Pkt --> Sniff
    Pkt --> Scan
    Pkt --> IDS
    Proc --> Auth
    Sys --> IDS
```
