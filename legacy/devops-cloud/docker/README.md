# Containerization with Docker

Comprehensive interview study guide covering Docker, containers vs. VMs, image layers, multi-stage builds, and Docker volumes/networking.

---

## 1. Containers vs. Virtual Machines

| Attribute | Virtual Machines (VMs) | Docker Containers |
| :--- | :--- | :--- |
| **Guest OS** | Includes a full, separate Guest OS | **Shares host OS kernel** (No Guest OS) |
| **Virtualization Layer**| Hypervisor (Hardware-level) | Container Engine (OS-level via namespaces/cgroups) |
| **Resource Usage** | High (GBs of memory/disk per VM) | **Extremely Low** (MBs, shares host libraries) |
| **Startup Time** | Slow (minutes to boot OS) | **Instant** (seconds or milliseconds) |
| **Isolation** | Strong (complete hardware hypervisor partition) | Sandboxed (namespace boundaries, shared kernel) |

---

## 2. Linux Kernel Features Powering Docker

Docker is not a magic compiler; it is a user-friendly wrapper over two core **Linux Kernel features**:

1. **Namespaces (Isolation):** Isolates the view of resources per container. Types of namespaces:
   * `pid` (processes)
   * `net` (network interfaces)
   * `mnt` (mount points/filesystem)
   * `uts` (hostname)
   * `ipc` (interprocess communication)
2. **Control Groups (cgroups - Resource Limiting):** Restricts and monitors resource usage (CPU, memory, disk IO, network bandwith) for a group of processes, preventing a single container from crashing the entire host.

---

## 3. Deep-Dive: Linux Namespace, Cgroups v2, & OverlayFS Mechanics

For staff-level systems engineering, you must understand exactly how the Linux kernel isolates processes, bounds memory/CPU, and layers filesystems under the hood.

### 1. Namespaces and Filesystem Isolation (`clone` & `pivot_root`)
When a container engine starts a container, it invokes the Linux **`clone()` syscall** rather than standard `fork()`. The `clone()` syscall allows child processes to execute in completely new, isolated kernel namespaces by passing specific execution flags:

```c
// Lower-level C representation of container creation syscalls
int container_pid = clone(
    container_main_function, 
    stack_pointer_end, 
    CLONE_NEWPID  | // Isolates Process IDs (Container PID 1 maps to Host PID 10425)
    CLONE_NEWNET  | // Isolates network stack (creates unique loopback & eth0 routing)
    CLONE_NEWNS   | // Isolates mount points (MNT namespace)
    CLONE_NEWIPC  | // Isolates Interprocess Communication (System V IPC, POSIX message queues)
    CLONE_NEWUTS  | // Isolates hostname and NIS domain name
    CLONE_NEWUSER | // Isolates UID/GID mappings (Container root maps to unprivileged host UID)
    SIGCHLD, 
    arg_payload
);
```

#### Filesystem Isolation: `chroot` vs. `pivot_root`
* **`chroot` (Change Root):** Changes the active root directory directory path of the calling process to a specific sub-folder. **Security Gotcha:** `chroot` is insecure. Root processes can easily escape the jail using standard path traversals or nesting directory file descriptors.
* **`pivot_root` (Pivot Mount):** Moves the root file system of the current process's mount namespace to a target directory, and moves the old root directory to a sub-folder. This makes the old filesystem completely unreachable, ensuring secure, absolute container filesystem sandboxing.

---

### 2. Cgroups v2 (Control Groups) Unified Architecture
While Cgroups v1 used discrete, split directory hierarchies for each resource controller (CPU, Memory, IO), **Cgroups v2** implements a **single, unified controller hierarchy**, eliminating race conditions and simplifying resource configurations.

```
Unified cgroup v2 Directory Root: /sys/fs/cgroup/
├── cgroup.procs              (Lists all host-wide process IDs)
├── docker/                   (Docker container resource slice)
│   ├── container_id_123/     (Individual container leaf cgroup)
│   │   ├── cgroup.procs      (Stores process IDs running inside this container)
│   │   ├── cpu.max           (Defines period-quota CPU limit, e.g. "50000 100000" = 0.5 CPU)
│   │   ├── memory.max        (Defines maximum hard memory limit, e.g. "536870912" = 512MB)
│   │   └── io.max            (Defines maximum read/write IO operations per second)
```

If a container exceeds the hard threshold defined in its local `memory.max` control file, the Linux kernel's **OOM (Out Of Memory) Killer** intercepts and terminates process `PID 1` inside the cgroup instantly, resulting in a container exit status code `137`.

---

### 3. OverlayFS (Layered Storage Engine) Mechanics
To keep image storage small and builds fast, Docker uses **OverlayFS** to overlay multiple read-only filesystem layers into a single cohesive directory tree.

```
       Merged View Directory (What the container process actually sees)
                     /sys/fs/overlay/merged/
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
       Upperdir (Writable)             Lowerdir (Read-Only)
    /var/lib/docker/overlay2/       /var/lib/docker/overlay2/
         [container-layer]               [immutable-image-layers]
               │
               ▼
       Workdir (Scratch)
    /var/lib/docker/overlay2/
         [scratch-space]
```

#### The Four OverlayFS Directories
* **Lowerdir:** Immutable, read-only layers representing base OS and packages.
* **Upperdir:** The read-write container runtime layer. Any changes, additions, or deletions go here.
* **Workdir:** Internal scratch space used by OverlayFS to execute atomic file operations before committing changes to Upperdir.
* **Mergeddir:** The unified virtual view. Linux merges Upperdir and Lowerdir into this directory, presenting a standard file structure to the container's processes.

#### Copy-on-Write (CoW) Resolution Mechanics
1. **File Read:**
   * If a file exists in Upperdir, read it directly.
   * If not in Upperdir, read it from Lowerdir with zero performance penalty.
2. **File Modification (Copy-on-Write):**
   * If a container process attempts to write to a file residing in the read-only Lowerdir, OverlayFS intercepts the write operation.
   * It copies the target file from Lowerdir up to the writable Upperdir.
   * The process then performs its write modifications directly on the Upperdir clone, hiding the original lower file.
3. **File Deletion:**
   * You cannot delete files from the read-only Lowerdir.
   * To delete a file, OverlayFS writes a special **whiteout character device file** (major `0`, minor `0`) in the corresponding Upperdir path. The kernel detects this whiteout file and completely hides it from the Mergeddir view.

---

---

## 3. Image Layers & Multi-Stage Builds

### Docker Image Layers (Copy-on-Write)
Every instruction in a `Dockerfile` (e.g., `FROM`, `RUN`, `COPY`) creates a **read-only, immutable filesystem layer**.
* When you run a container, Docker adds a thin, writeable **Container Layer** on top.
* **Cache Optimization:** Docker caches layers. If you change a line at the bottom of your Dockerfile (e.g., copying source files), Docker reuses cached layers from prior steps (like installing node modules), speeding up builds. Put stable commands (package installs) *above* volatile commands (source copy).

### Multi-Stage Builds
A technique used to minimize image size and exclude build tools (compilers, npm, git) from production images:

```dockerfile
# Stage 1: Build the binary
FROM golang:1.20-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

# Stage 2: Production runtime image
FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```
* **Result:** The final image contains strictly the thin `alpine` OS and the pre-compiled compiled executable, dropping the massive Go SDK and intermediate files (reducing image size from 800MB to ~15MB).

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between a Docker Image and a Docker Container?
* **Answer:** A **Docker Image** is an inactive, read-only, immutable blueprint template containing the application code, libraries, dependencies, and environment configurations needed to run. A **Docker Container** is a running, active **live instance** of an image. It is instantiated by adding a read-write filesystem layer on top of the image and starting isolated processes inside OS-allocated namespaces.

### Q2: Why are Docker volumes necessary, and what is the difference between a Bind Mount and a Named Volume?
* **Answer:** By default, containers are **stateless and ephemeral**: any data written inside a container is stored in its thin, writeable layer and is permanently lost when the container is deleted. To persist data (like database files), we use **Docker Volumes**:
  * **Bind Mount:** Maps an exact physical path on the host machine (e.g., `/home/user/db`) to a path inside the container. This is highly useful for local development (changes on the host update inside the container instantly).
  * **Named Volume:** Managed directly by Docker inside its internal storage directory (e.g., `/var/lib/docker/volumes/`). Named volumes are safer, portable across containers, and have better performance on non-Linux hosts (Mac/Windows).

### Q3: What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile?
* **Answer:**
  * **`ENTRYPOINT`** sets the **main executable** that will always run when the container starts. It defines the container's primary command.
  * **`CMD`** sets the **default arguments** that will be passed to the `ENTRYPOINT` command. If the user runs `docker run image args`, the user-provided arguments will completely override `CMD`, but `ENTRYPOINT` will still execute.
