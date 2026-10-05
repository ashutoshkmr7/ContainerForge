# Container Forge

**Container Forge** is a lightweight Linux container runtime written in **C++17**. It builds a small, Docker-style container environment directly on top of Linux kernel primitives—without Docker, libcontainer, or an external container runtime library.

> **Learning / systems project:** Container Forge is designed to demonstrate how Linux namespaces, mount operations, `pivot_root`, cgroup v2, and a tiny PID 1 can be combined to create an isolated process environment. It is **not a production-grade security boundary**.

## Features

- **PID namespace isolation** — containers get their own process tree.
- **UTS namespace isolation** — configurable container hostname.
- **Mount namespace isolation** — private filesystem mounts.
- **IPC namespace isolation**.
- **Network namespace isolation** by default, with `--host-net` as an opt-in.
- **Private root filesystem** using `pivot_root`, with a `chroot` fallback.
- **Minimal `/dev`** backed by tmpfs and common device nodes.
- **Private `/proc`** showing processes from the container's PID namespace.
- **cgroup v2 resource controls**:
  - CPU quota with `--cpu`
  - Memory limit with `--mem`
  - Process-count limit with `--pids`
- **Container lifecycle commands**:
  - `run`
  - `ps`
  - `stats`
  - `stop`
- **Tiny PID 1** that forwards termination signals and reaps child processes.
- **BusyBox root filesystem generator** for creating a small runnable userspace.
- No external C++ runtime dependencies beyond the standard Linux/POSIX APIs.

## Architecture

```text
                         Container Forge
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
             Namespaces                   cgroup v2
                 │                           │
       ┌─────────┼─────────┐         ┌───────┼────────┐
       │         │         │         │       │        │
      PID       UTS      Mount      CPU    Memory    PIDs
       │         │         │
       │         │         └── private root filesystem
       │         └────────────── container hostname
       └──────────────────────── isolated process tree
```

At a high level:

1. `clone()` creates a child in the requested Linux namespaces.
2. The child prepares the root filesystem and mounts `/proc`, `/dev`, `/tmp`, and `/sys`.
3. `pivot_root` changes the process into the container filesystem.
4. The parent creates a cgroup v2 and applies resource limits.
5. The child becomes PID 1 inside the PID namespace.
6. Runtime state is recorded under `/run/container-forge` so `ps`, `stats`, and `stop` can manage running containers.

## Project Structure

```text
container-forge/
├── container_forge.cpp       # Container runtime implementation
├── make_rootfs.sh            # Builds a minimal BusyBox root filesystem
├── Makefile                  # Build and rootfs targets
├── README.md                 # Documentation
├── LICENSE                  # MIT license
├── .gitignore               # Generated/runtime files
├── .gitattributes           # Git text handling
├── .github/
│   └── workflows/
│       └── build.yml        # Ubuntu CI build
├── CONTRIBUTING.md          # Contribution guidelines
└── SECURITY.md              # Security limitations and reporting guidance
```

## Requirements

Container Forge targets a **Linux environment** with:

- `g++` with C++17 support
- `make`
- Linux namespaces and mount support
- **cgroup v2** for resource limits
- root privileges for the container setup
- `busybox-static` recommended for the smallest root filesystem

The project is intended to work well on modern Ubuntu systems, including Ubuntu 24.04.

### WSL2

WSL2 can be useful for development, but kernel configuration and cgroup availability can vary.

Check the cgroup filesystem with:

```bash
stat -fc %T /sys/fs/cgroup
```

Expected output:

```text
cgroup2fs
```

If you are using WSL, run Windows-side commands such as `wsl -l -v` from **PowerShell**, not from the Linux shell.

## Build

Install the dependencies:

```bash
sudo apt update
sudo apt install -y g++ make busybox-static
```

Build the runtime:

```bash
make
```

Verify:

```bash
./container-forge help
```

## Create the Root Filesystem

Generate a minimal BusyBox-based root filesystem:

```bash
chmod +x make_rootfs.sh
sudo ./make_rootfs.sh
```

Verify the generated BusyBox binary:

```bash
ls -lh rootfs/bin/busybox
```

The result should be a regular executable.

## Quick Start

Start an isolated shell:

```bash
sudo ./container-forge run --name box1 --hostname mybox /bin/sh
```

Inside the container:

```sh
hostname
ps
ls /
exit
```

The hostname should be `mybox`, and the process view should be isolated from the host.

## Resource Limit Examples

### CPU

Start a CPU-bound workload with a 25% CPU limit:

```bash
sudo ./container-forge run --name worker --cpu 25 /bin/yes > /dev/null
```

In another terminal:

```bash
sudo ./container-forge ps
sudo ./container-forge stats -w worker
```

Stop the workload:

```bash
sudo ./container-forge stop worker
```

### Memory

Run a workload with a 50 MiB memory limit:

```bash
sudo ./container-forge run --mem 50M /bin/tail /dev/zero
```

When the workload exceeds its cgroup memory limit, cgroup v2 can terminate the process. A SIGKILL-based termination commonly results in exit code `137`.

### Process Count

Limit the container to ten processes:

```bash
sudo ./container-forge run --pids 10 /bin/sh
```

The cgroup `pids.max` controller prevents the container from creating more processes than the configured limit.

## Command Reference

```text
container-forge run [options] [--] [COMMAND [ARGS...]]
container-forge ps
container-forge stats [-w] NAME|ID
container-forge stop [-t SEC] NAME|ID
```

### `run` Options

| Option | Description |
|---|---|
| `--rootfs DIR` | Root filesystem; defaults to `./rootfs` |
| `--name NAME` | Human-readable container name |
| `--hostname NAME` | Hostname inside the container |
| `--mem SIZE` | Memory limit, e.g. `64M` or `1G` |
| `--cpu PERCENT` | CPU quota; `100` represents one CPU core |
| `--pids N` | Maximum number of processes |
| `--host-net` | Share the host network namespace |

## How It Works

### 1. Linux Namespaces

The runtime uses:

- `CLONE_NEWPID`
- `CLONE_NEWUTS`
- `CLONE_NEWNS`
- `CLONE_NEWIPC`
- `CLONE_NEWNET` unless `--host-net` is selected

These provide isolation for processes, hostnames, mounts, IPC, and networking.

### 2. Root Filesystem

The container setup:

1. makes mount propagation private;
2. bind-mounts the selected root filesystem;
3. mounts a private `/proc`;
4. mounts a small tmpfs-backed `/dev`;
5. mounts a writable `/tmp`;
6. switches into the root filesystem with `pivot_root`;
7. falls back to `chroot` if `pivot_root` cannot be used.

### 3. cgroup v2

Container resource limits are applied using cgroup v2 control files:

```text
memory.max
cpu.max
pids.max
```

The child process is added to its cgroup before it is released to execute the requested command.

### 4. PID 1

Inside the PID namespace, Container Forge acts as a tiny init process. It:

- forwards termination signals;
- waits for child processes;
- reaps orphaned processes;
- returns the workload's exit status.

### 5. Runtime State

Running containers are tracked using small state files under:

```text
/run/container-forge
```

These records are used by `ps`, `stats`, and `stop`.

## Security Limitations

Container Forge intentionally does **not** implement several hardening features found in production runtimes:

- no user namespaces;
- no capability dropping;
- no seccomp profiles;
- no read-only root filesystem;
- no virtual Ethernet or port mapping;
- no image/layer system;
- no volume management;
- no daemon;
- foreground execution only.

**Do not run untrusted workloads with Container Forge.** The runtime currently operates with real root privileges and is intended for learning, experimentation, and systems programming practice.

See [`SECURITY.md`](SECURITY.md) for more details.

## Troubleshooting

### cgroup v2 is not available

Check:

```bash
stat -fc %T /sys/fs/cgroup
```

Resource limits require cgroup v2.

### Root filesystem is missing

Run:

```bash
sudo ./make_rootfs.sh
```

### BusyBox is missing

Install:

```bash
sudo apt install -y busybox-static
```

### Permission errors

Namespace and cgroup setup normally requires root:

```bash
sudo ./container-forge run ...
```

### Build fails

Make sure the compiler supports C++17:

```bash
g++ --version
```

Then rebuild from a clean state:

```bash
make clean
make
```

## Development

Build:

```bash
make
```

Create the root filesystem:

```bash
make rootfs
```

Remove the compiled binary:

```bash
make clean
```

Remove the binary and generated root filesystem:

```bash
make clean-all
```

GitHub Actions also builds the project automatically on pushes and pull requests using Ubuntu 24.04.

## Roadmap

Potential future improvements:

- detached/background containers with log handling;
- `exec` support using `setns()`;
- user namespace support;
- Linux capability dropping;
- seccomp filtering;
- veth networking and port mapping;
- bind-mounted volumes;
- read-only root filesystem support;
- richer container metadata and logs;
- image/layer support.

## License

Container Forge is released under the MIT License. See [`LICENSE`](LICENSE).
