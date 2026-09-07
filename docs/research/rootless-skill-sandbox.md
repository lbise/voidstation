# Rootless Skill sandbox and HTTPS enforcement

Research for [issue 15](https://github.com/lbise/voidstation/issues/15). This note does not make the application or storage decision. It recommends the smallest design I would trust for the settled five-minute Python 3.12 Skill run on one Linux host.

## Recommendation

Use **rootless Docker**, owned by a dedicated `voidstation-sandbox` Linux account, and a separate narrow supervisor. The application talks to the supervisor over its own authenticated Unix socket. Only the supervisor can reach the rootless Docker socket. The application and every Skill container receive no engine socket.

Start every script container with `--network none`. Docker documents that this leaves only loopback in the container, so it prevents direct DNS, TCP, UDP, and metadata requests rather than merely suggesting a proxy. [Docker none network driver](https://docs.docker.com/engine/network/drivers/none/)

A network grant must therefore mean access to a small supervisor-owned HTTPS request broker over a unique per-run Unix socket. This is a deliberate API change: a networked Skill can make bounded HTTPS requests through the broker, but cannot open arbitrary Internet sockets. It is the smallest credible way to enforce an exact-origin list against malicious Python. An `HTTPS_PROXY` value is not an enforcement boundary. Python's `urllib.request` installs proxy handling from `*_proxy` environment variables, and a script can instead use `socket` or `http.client` directly. [Python urllib proxy handling](https://docs.python.org/3.12/library/urllib.request.html#urllib.request.getproxies)

Rootless Docker is the better fit here because Docker Compose is already accepted for the application and its rootless documentation states the resource requirement and the required systemd delegation precisely. Podman can also run the container shape, but adds no security property to this design. Podman's own rootless notes say that cgroup v1 has no resource-limit support and that its default rootless networking is `pasta`. We do not need that networking when the container has none. [Podman rootless limitations](https://github.com/containers/podman/blob/main/rootless.md)

## What the sources establish

### Rootless engine and host requirements

Docker rootless mode runs both daemon and containers in a user namespace, unlike `userns-remap`, where the daemon remains privileged. It requires `newuidmap` and `newgidmap`, plus at least 65,536 subordinate UIDs and GIDs for the account in `/etc/subuid` and `/etc/subgid`. [Docker rootless mode](https://docs.docker.com/engine/security/rootless/)

For rootless Docker, `--cpus`, `--memory`, and `--pids-limit` work only with cgroup v2 and systemd. If `docker info` reports Cgroup Driver `none`, Docker ignores those flags. With the systemd driver, Docker says the usual user delegation contains only `memory` and `pids`; its documented `user@.service` drop-in delegates `cpu cpuset io memory pids`. `cpuset` needs systemd 244 or later. [Docker rootless resource limits](https://docs.docker.com/engine/security/rootless/tips/#limiting-resources)

This makes the following host requirements mandatory, not tuning advice:

- a unified cgroup v2 host, systemd user manager, rootless Docker with Cgroup Driver `systemd`;
- a system administrator-installed `user@.service` delegation for at least `cpu memory pids`, with the Docker-documented controller list preferred;
- the subordinate-ID setup above and a persistent user manager if the supervisor must survive logout. Docker documents `loginctl enable-linger` for this, and says a system-wide rootless Docker service is unsupported. [Docker rootless tips](https://docs.docker.com/engine/security/rootless/tips/)

Systemd's cgroup documentation agrees on the model: a container manager must own a delegated subtree, not create cgroups below systemd-owned arbitrary paths. `Delegate=` gives the service authority to create and manage sub-cgroups there. [systemd cgroup delegation](https://systemd.io/CGROUP_DELEGATION/)

A Docker socket is privileged control access. Docker warns that access to its daemon can give root access to the host that runs the daemon. Rootless reduces that blast radius, but the socket still lets its holder create containers and bind mount data readable by the sandbox account. Keep it out of the application and containers. [Docker socket protection](https://docs.docker.com/engine/security/protect-access/)

### Container controls available from Docker

Docker documents hard memory limits, CPU quota through `--cpus`, process limits through `--pids-limit`, read-only root filesystems, bind mounts, tmpfs mounts, an init process that forwards signals and reaps children, and explicit stop, kill, and remove operations. A memory limit alone permits equal swap by default. Set `--memory=512m --memory-swap=512m` if the 512 MiB limit includes swap. [Docker resource constraints](https://docs.docker.com/engine/containers/resource_constraints/) [Docker run reference](https://docs.docker.com/reference/cli/docker/container/run/)

`docker stop` sends the configured signal, `SIGTERM` by default, then sends `SIGKILL` after its timeout. Forced removal also sends `SIGKILL`. [Docker stop](https://docs.docker.com/reference/cli/docker/container/stop/) [Docker remove](https://docs.docker.com/reference/cli/docker/container/rm/)

Docker's default seccomp profile blocks, among other things, `mount`, `unshare`, and new-namespace `clone`. It is useful defense in depth, not a reason to treat a container as a VM. [Docker seccomp](https://docs.docker.com/engine/security/seccomp/)

### HTTPS policy facts

Python 3.12 has ordinary direct HTTPS clients and HTTP CONNECT support. `urllib.request` also follows redirects by default through `HTTPRedirectHandler`. Those facts are why a proxy environment variable, a DNS allowlist at container start, or checking only the first URL cannot enforce the grant. [Python HTTP client](https://docs.python.org/3.12/library/http.client.html) [Python redirects](https://docs.python.org/3.12/library/urllib.request.html#urllib.request.HTTPRedirectHandler)

The broker can make normal certificate and hostname verification mandatory. Python documents that `PROTOCOL_TLS_CLIENT` enables certificate verification and hostname checking by default, and that the TLS handshake aborts when OpenSSL rejects the hostname. [Python SSL contexts](https://docs.python.org/3.12/library/ssl.html#ssl.SSLContext.check_hostname)

Do not use `ipaddress.is_private` as the policy. Python documents that the shared `100.64.0.0/10` range is neither private nor global. Instead, vendor a reviewed snapshot of the IANA IPv4 and IPv6 special-purpose registries and permit only addresses marked globally reachable. This blocks loopback, private, link-local, and the link-local metadata addresses, including `169.254.169.254`. [IANA IPv4 registry](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml) [IANA IPv6 registry](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml)

## Proposed run shape

This section is an engineering proposal, not a Docker feature claim.

The supervisor runs as `voidstation-sandbox`, a Linux account separate from the web application account. Its rootless Docker daemon uses that account's private runtime directory. The supervisor accepts only a typed request such as `start(run-id, immutable-package-id, entrypoint, validated-input, grants)` and `cancel(run-id)`. It does not accept an image name, a mount path, Docker arguments, or an arbitrary command from the application.

For a run it creates a fresh supervisor-owned directory and mounts only these items:

- the selected package copy and entry script, read-only;
- a fresh writable workspace, capped at 256 MiB, with no path to installation, database, application, account home, engine runtime directory, or host socket;
- individual granted Secret files, read-only, at fixed names;
- when the network capability is granted, one per-run broker socket. It is an authority to make broker requests, not a host filesystem grant.

The fixed image is built and pinned by Voidstation. It contains Python 3.12, its standard library, CA certificates, and no package manager, shell, compiler, or user-selected executable. The supervisor invokes a fixed Python command against the selected entry script, rather than invoking a package-provided command line. Run it as a non-root container user, with a read-only root filesystem, no added capabilities, `no-new-privileges`, Docker's default seccomp profile, and no devices or published ports. Reject native extensions and executable package payloads at installation. `--init` is appropriate because scripts may fork.

The intended fixed limits are `--cpus=1`, `--memory=512m`, `--memory-swap=512m`, `--pids-limit=64`, a 256 MiB workspace tmpfs or equivalently enforced per-run filesystem quota, and an external five-minute monotonic-clock watchdog. Set output byte limits in the supervisor while it reads stdout and stderr. Once the 1 MiB capture limit is reached, mark the protocol failure and stop the container. Never rely on a script's own limits.

The supervisor validates the engine's effective settings after creation. In particular, it must confirm the systemd cgroup driver and inspect the run cgroup for the expected `cpu.max`, `memory.max`, `memory.swap.max`, and `pids.max` values. A successful `docker run` is not proof of a limit because rootless Docker can ignore cgroup flags on an unsuitable host.

## Exact-origin broker policy

The broker accepts a structured request with method, URL, bounded headers, and bounded body. It must reject arbitrary socket, CONNECT, proxy, DNS, and TLS-control operations. It has a separate output and body-size limit and a deadline no later than the run deadline.

For each request, the broker should:

1. Parse the URL itself. Accept only `https`, no userinfo, no IP literals, and an authority that exactly equals one granted origin after ASCII IDNA and lowercase-host canonicalization. The grant contains scheme, host, and explicit port. It should normally allow only port 443.
2. Resolve the allowed hostname itself. Reject the request if any returned A or AAAA address is absent from the broker's IANA globally-reachable allow set. Connect only to a retained, validated numeric address from that resolution. This binds the validation to the connection and closes the usual DNS-rebinding gap.
3. Send SNI and HTTP `Host` for the allowed hostname, validate the normal CA chain and hostname, and reject a TLS failure. Do not disable verification or add a Skill-provided CA.
4. Disable automatic redirects. If the response has `Location`, either return it as a response or treat the next hop as a new request that repeats every check above. Do not carry credentials or request headers to a different origin by default.
5. Refuse proxy settings and only expose the broker socket to the run. The broker itself must not proxy to an arbitrary authority after the check.

This is stronger than a transparent CONNECT proxy. A CONNECT proxy can check its `host:port`, but it still needs a network arrangement that proves the script cannot bypass the proxy. With `--network none`, there is no bypass route. The tradeoff is real: Skills must use a Voidstation fetch client over the broker socket instead of an unrestricted Python networking API.

## Cancellation and cleanup proposal

The supervisor owns the state machine. On user cancellation, timeout, oversized output, or malformed output, it stops accepting broker requests, closes active broker connections, sends `SIGTERM`, waits at most ten seconds, then sends `SIGKILL` or force-removes the container. It then verifies that the engine reports no remaining container processes, removes the container, removes the per-run directory, and records `cancelled` rather than a terminal script result. A script result that races with cancellation must not change that state. The workspace is temporary. The supervisor keeps only the already-specified conversation event and non-sensitive diagnostic metadata.

No cleanup sequence can roll back a request already sent to a remote service. Cancellation can stop later requests and delete local state. It cannot undo an external side effect.

## Required runtime proof before accepting this design

No target-host runtime test was run for this research note. Treat every item below as an acceptance gate, not as completed work.

| Gate | Pass condition |
| --- | --- |
| Host | `stat -fc %T /sys/fs/cgroup` is `cgroup2fs`; `docker info` says `rootless` and Cgroup Driver `systemd`; the sandbox user's `cgroup.controllers` includes `cpu memory pids`. |
| Hard limits | A CPU spinner is throttled to one CPU; an allocator is OOM-killed at 512 MiB; a forker cannot exceed 64 processes; swap does not exceed the chosen policy. Inspect the cgroup files as well as process behavior. |
| Files | A hostile script cannot read the application data, another run, the sandbox account home, the engine socket, or an ungranted Secret. It can write only the workspace and cannot exceed 256 MiB. |
| Network | Direct IPv4, IPv6, DNS, loopback, RFC 1918, link-local, and metadata connections fail. The broker rejects an IP literal, mixed public and private DNS answer, DNS rebinding attempt, wrong TLS name, invalid chain, disallowed port, and redirect to another origin. It permits a valid granted origin. |
| Lifecycle | A process tree, stalled stdout, and pending broker request all terminate at cancellation and at five minutes. The run directory and container disappear, while the final event has the correct state. |
| Regressions | Repeat the checks after Docker, kernel, systemd, rootlesskit, image, CA bundle, and broker changes. |

## Limits and choices left to the owner

Rootless containers reduce the consequence of a container-runtime compromise. They do not remove kernel, OCI runtime, Docker daemon, or supervisor vulnerabilities. This design needs normal host patching and a dedicated account, and it should not be described as a security boundary equal to a VM.

The owner still needs to choose whether the network capability may use the constrained fetch API. If unrestricted Python sockets are a product requirement, this recommendation no longer proves exact-origin enforcement. It needs a more complex privileged network namespace and firewall or transparent-proxy design, with a fresh threat-model and runtime verification.

The owner also needs to choose the allowed HTTP methods, request and response sizes, supported authentication patterns, non-443 origin policy, redirect behavior, and whether a Secret may be granted together with a network origin. Those are product policy choices. The broker can enforce whichever narrow form is selected.
