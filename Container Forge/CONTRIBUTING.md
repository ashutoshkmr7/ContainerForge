# Contributing

Thanks for your interest in Container Forge.

## Development Setup

Use a Linux environment with:

```bash
sudo apt update
sudo apt install -y g++ make busybox-static
```

Build the project:

```bash
make
```

Generate the test root filesystem:

```bash
make rootfs
```

## Before Opening a Pull Request

Please:

1. Build successfully with `make`.
2. Run `./container-forge help`.
3. Test changes affecting namespaces, mounts, or cgroups on a disposable Linux environment.
4. Keep generated files such as `rootfs/` and the compiled binary out of commits.
5. Update the README when changing commands or user-facing behavior.
6. Keep the implementation dependency-light and compatible with C++17.

## Commit Messages

Prefer short, descriptive commit messages, for example:

```text
feat: add memory statistics
fix: clean stale container state
docs: clarify cgroup requirements
```

## Scope

Container Forge is primarily a systems-programming and educational project. Changes should favor clarity and correctness over unnecessary abstraction.
