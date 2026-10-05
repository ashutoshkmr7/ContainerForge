# Security

## Important Warning

**Container Forge is not a production container runtime and should not be treated as a security boundary.**

The runtime performs privileged Linux operations and currently runs containers with host root privileges. Namespace isolation and cgroup limits reduce process and resource visibility, but they do not provide the defense-in-depth expected from mature container runtimes.

## Known Limitations

The current implementation does not provide:

- user namespace isolation;
- Linux capability dropping;
- seccomp filtering;
- a read-only root filesystem;
- a production-grade device policy;
- virtual networking and port isolation;
- image verification or layered filesystems;
- a hardened daemon architecture.

## Safe Usage

Use Container Forge with:

- disposable test environments;
- trusted workloads;
- non-sensitive development systems;
- virtual machines when experimenting with namespace and mount behavior.

Do **not** use it to execute arbitrary code from untrusted users, downloaded images, or unknown repositories.

## Reporting a Vulnerability

If you discover a security issue, please do not publish exploit details in a public issue before the maintainer has had an opportunity to assess it.

For this educational project, open a GitHub issue with a concise description and reproduction steps when public discussion is appropriate.
