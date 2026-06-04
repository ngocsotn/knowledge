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
