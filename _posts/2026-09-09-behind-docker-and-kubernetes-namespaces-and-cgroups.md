---
layout: post
title: "Behind Docker and Kubernetes: It's Actually Namespaces and Cgroups"
date: 2026-09-09
excerpt: "Docker and Kubernetes feel like magic, but there's no container-specific code in the Linux kernel. It's two much older mechanisms — namespaces and cgroups — doing all the work."
---

I used to think Docker had some special kernel-level "container mode" tucked away somewhere. It
doesn't. Open the Linux kernel source and search for `container` — you won't find a subsystem, a
system call, nothing. Docker and Kubernetes didn't invent containers, and they don't get any help
from the kernel that you couldn't get yourself. What they actually do is drive two much older,
much less glamorous features that have been sitting in Linux for over a decade: **namespaces** and
**cgroups**. Everything else — images, registries, YAML manifests, schedulers — is just tooling
wrapped around those two things.

## So what's actually being isolated?

Here's the problem a container has to solve. It's supposed to feel like its own little machine —
its own filesystem, its own process list, its own hostname, its own slice of CPU and memory so it
can't starve whatever else is running on the box. But strip away the marketing and a container is
just a regular process, running on the exact same shared kernel as everything else on that host.

So how do you get one process to *believe* it has the machine to itself, while also making sure it
*behaves* and only takes its fair share? Turns out the kernel already splits that job into two
neat halves. Namespaces decide what a process is allowed to **see**. Cgroups decide what it's
allowed to **use**. That's really the whole trick.

## Namespaces: giving a process its own little bubble

A namespace takes some global resource the kernel manages — process IDs, network interfaces,
mount points — and wraps it so that a process inside the namespace only sees its own private copy,
not the host's real one. Linux ships with a handful of these: `pid` hides the rest of the system's
processes and lets your container think its app is PID 1; `net` gives it its own network stack;
`mnt` gives it its own view of the filesystem; `uts` lets it set its own hostname without touching
the host's; `ipc` keeps shared memory and message queues private; and `user` lets "root" inside the
container be a completely unprivileged nobody on the host.

When Docker starts a container, under the hood it's just calling `clone()` with a pile of
`CLONE_NEW*` flags, and the resulting process gets brand-new, empty namespaces instead of joining
the host's. That's the entire reason `ps aux` inside a container only shows a handful of processes
even though the host might be running thousands — the container literally can't see the rest.

You don't need Docker to try this yourself. This works on a plain Linux box:

```bash
# Run a shell in new UTS, PID, and mount namespaces
sudo unshare --uts --pid --mount --fork /bin/bash

# Inside, this shell genuinely believes it's PID 1
ps aux
```

That's it. That's "container isolation." No hypervisor, no virtual hardware, no magic — just the
kernel handing a process a private view of state it would normally share with everyone else.

## Cgroups: making sure nobody hogs the machine

Namespaces alone won't stop a container from eating every last bit of CPU or memory on the host —
they only control visibility, not consumption. That's where **cgroups** (control groups) come in.
A cgroup groups a set of processes together and lets the kernel cap, meter, and prioritize what
that group is allowed to consume as a whole.

This is what's actually enforcing limits like these:

```bash
docker run --cpus="1.5" --memory="512m" my-image
```

Docker doesn't police that limit itself in userspace somewhere. It writes it into the cgroup
filesystem, and from that point on it's the kernel's scheduler and memory manager doing the
enforcing, the same way they would for any other group of processes. On a cgroup v2 system you can
go look at the actual files yourself:

```bash
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/cpu.max
```

Those numbers *are* the limit. There's no separate "container runtime police" watching over the
process — it's plain cgroup accounting, doing what it's always done.

## If the kernel does all that, what's Docker even for?

Fair question. If namespaces handle isolation and cgroups handle limits, Docker's job is really
everything *around* those two primitives, not a replacement for them. It gives you a portable image
format — layers on `overlayfs` — so there's something consistent to mount into that fresh `mnt`
namespace. It runs a daemon that talks to a low-level runtime called `runc` (which implements the
[OCI runtime spec](https://github.com/opencontainers/runtime-spec)) so you never have to call
`clone()` and set up cgroups by hand. It wires up virtual network interfaces and NAT rules so
containers sitting in separate `net` namespaces can still reach each other and the outside world.
And it gives you registries — a place to push and pull those images from.

Kubernetes sits a layer above even that. It never touches namespaces or cgroups directly — it talks
to a container runtime through the CRI (`containerd`, `CRI-O`, and so on), which calls `runc`,
which finally talks to the kernel. What Kubernetes actually decides is *which node* a pod lands on,
and *how much* cgroup budget it gets (that's your `resources.requests` and `resources.limits` in
the pod spec). It's also the reason containers in the same pod share `localhost` — they're placed
in the same `net` namespace on purpose. Kubernetes is orchestration sitting on top of primitives
the kernel has had since long before "cloud native" was a phrase anyone used.

## Why any of this is worth knowing

Once "a container is just a process with different namespaces and a cgroup" clicks, a bunch of
things that used to feel like arbitrary Docker trivia suddenly make sense on their own. `docker run
--pid=host` lets a container see every process on the host because it simply skips creating a new
PID namespace — there was never a special "host visibility mode," just one less wall being built.
Containers used to read `/proc/meminfo` and see the *host's* full memory, even with a `--memory`
limit set, because cgroups restrict usage, they don't rewrite what `/proc` reports — a mismatch
that broke plenty of JVMs before tooling caught up. "Container escape" vulnerabilities are really
just namespace or capability misconfigurations, not some exotic container-specific attack surface.
And containers can start in milliseconds and run without virtualization extensions because there's
genuinely no hypervisor anywhere in the picture.

None of this makes Docker or Kubernetes less useful — packaging, scheduling, self-healing, and
networking at scale are still hard, worthwhile problems to have solved for you. But the isolation
part, the bit that feels like magic, isn't really theirs. It's Linux, doing what it's quietly done
since kernel 2.6.24 finished off the last of the core namespaces. Containers aren't a new kind of
thing running on your machine. They're just a new, friendlier way of asking the same old kernel to
hide and budget things on your behalf.
