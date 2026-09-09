---
layout: post
title: "Behind Docker and Kubernetes: It's Actually Namespaces and Cgroups"
date: 2026-09-09
excerpt: "Docker and Kubernetes feel like magic, but there's no container-specific code in the Linux kernel. It's two much older mechanisms — namespaces and cgroups — doing all the work."
---

Docker and Kubernetes get talked about like they invented containers. They didn't. There is no
`container` system call, no `container` subsystem in the Linux kernel. What Docker and Kubernetes
actually do is orchestrate two much older, much more boring kernel features: **namespaces** and
**cgroups**. Everything else — images, registries, YAML, schedulers — is tooling built on top of
those two primitives.

## The problem: one kernel, many "isolated" processes

A container is supposed to look like its own little machine: its own filesystem, its own process
tree, its own network interface, its own hostname, and a capped amount of CPU/memory so it can't
starve its neighbors. But under the hood, a container is just a regular Linux process running on
the same shared kernel as everything else on the host.

So the question is: how do you make one process on a shared kernel *believe* it has the machine to
itself, and *behave* like it only gets a fair slice of resources? That's exactly the two-part job
namespaces and cgroups split between them:

- **Namespaces** control what a process can *see*.
- **Cgroups** control what a process can *use*.

## Namespaces: what a process can see

A namespace wraps a global kernel resource so that a process inside the namespace sees its own
isolated instance of it, instead of the real, host-wide one. Linux has several kinds:

| Namespace | Isolates |
|---|---|
| `pid`     | Process IDs — container's PID 1 is not the host's PID 1 |
| `net`     | Network interfaces, routing tables, ports |
| `mnt`     | Filesystem mount points |
| `uts`     | Hostname and domain name |
| `ipc`     | Inter-process communication (message queues, shared memory) |
| `user`    | User and group IDs (root in the container ≠ root on the host) |
| `cgroup`  | The process's view of its own cgroup hierarchy |

When Docker starts a container, it calls `clone()` (or `unshare()`) with a combination of these
`CLONE_NEW*` flags. The new process gets fresh namespace instances instead of joining the host's.
That's why `ps aux` inside a container shows only a handful of processes with your app as PID 1,
even though the host is running thousands.

You can see this without Docker at all:

```bash
# Run a shell in new UTS, PID, and mount namespaces
sudo unshare --uts --pid --mount --fork /bin/bash

# Inside, this shell thinks it's PID 1
ps aux
```

That's the whole trick behind "container isolation." No hypervisor, no virtual hardware — just the
kernel handing out a private *view* of shared state.

## Cgroups: what a process can use

Namespaces alone don't stop one container from hogging all the CPU or memory on the host. That's
what **control groups (cgroups)** are for. A cgroup is a kernel mechanism for grouping processes
and applying resource limits, accounting, and prioritization to the group as a whole.

With cgroups you can cap how much CPU time a group of processes gets, how much memory it can
allocate before being OOM-killed, how much I/O bandwidth it can consume, and more. Docker sets
these up for you every time you pass flags like:

```bash
docker run --cpus="1.5" --memory="512m" my-image
```

Under the hood, that turns into cgroup filesystem entries. On a cgroup v2 system, you'd see
something like:

```bash
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/cpu.max
```

Those files *are* the limit. There's no separate "container runtime" enforcing it — the kernel
scheduler and memory manager read those cgroup settings directly, the same way they would for any
other process group.

## So what does Docker actually add?

If the kernel already does isolation (namespaces) and resource limiting (cgroups), what's left for
Docker to do?

- **Image format & layering** — a portable way to package a root filesystem (`overlayfs` layers)
  so `mnt` namespace has something consistent to mount.
- **A daemon and CLI** — `dockerd` calls the low-level runtime (`runc`, which implements the
  [OCI runtime spec](https://github.com/opencontainers/runtime-spec)) to actually wire up the
  namespaces and cgroups for you, instead of you calling `clone()`/`unshare()` by hand.
- **Networking glue** — virtual ethernet pairs, bridges, and NAT rules so containers in separate
  `net` namespaces can still talk to each other and the outside world.
- **Distribution** — registries, tags, and push/pull.

Kubernetes goes a layer higher still. It doesn't touch namespaces or cgroups directly — it talks to
a container runtime (via the CRI: `containerd`, `CRI-O`, etc.), which in turn calls `runc`, which
calls the kernel. Kubernetes' job is scheduling *which node* a pod's containers land on, and
declaring *how much* cgroup budget (via `resources.requests` / `resources.limits` in a pod spec)
and *which* namespace isolation (via the pod's shared network namespace, for instance — that's why
containers in the same pod share `localhost`) that container should get. It's an orchestration
layer sitting on top of primitives the kernel has had for well over a decade.

## Why this matters

Once you know a container is "just a process with different namespaces and a cgroup," a lot of
confusing behavior stops being confusing:

- Why `docker run --pid=host` lets a container see every process on the host (it skips creating a
  new PID namespace).
- Why containers historically could see the *host's* total memory/CPU count from `/proc/meminfo`
  and `/proc/cpuinfo`, even with a `--memory` limit set (cgroups constrain usage, they don't fake
  those procfs files — this needed dedicated fixes and awareness in apps).
- Why "container escape" vulnerabilities are really about namespace or capability misconfiguration,
  not some separate container security model.
- Why running containers doesn't require virtualization extensions the way VMs do — there's no
  hypervisor in the picture at all.

Docker and Kubernetes are genuinely useful — packaging, scheduling, networking, and self-healing at
scale are hard problems worth solving. But the isolation "magic" people attribute to them is really
just Linux doing what it's done since kernel 2.6.24 introduced the last of the core namespaces.
Containers aren't a new kind of thing. They're a new way of asking the same old kernel to hide and
budget resources for you.
