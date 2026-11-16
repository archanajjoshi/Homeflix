# Day 1: Host Inventory and Linux Baseline

Study journal for the Homeflix infrastructure project. Hostnames and private network addresses are generalized in the public repository.

## Learning goal

Build an evidence-based baseline of both physical Ubuntu hosts before planning VMs or changing configuration.

## Concepts and useful questions

- A host inventory records OS, CPU, memory, storage, network, and virtualization capabilities so infrastructure choices use measured facts.
- Ask what evidence would answer a question. For example: “What OS and kernel is each host running, and which command confirms it?”
- `hostnamectl` reports host and operating-system information; `cat /etc/os-release` prints OS release details; `uname -r` prints the running kernel release.
- CPU model, core count, and logical CPU count help estimate processing capacity. `lscpu` reports these details and virtualization flags.
- RAM is shared by the host OS and workloads. `free -h` reports total, used, free, available memory, and swap. The `available` value is a better rough estimate of memory available for new work than `free` alone.
- `lsblk` shows block devices and partitions. `df -hT` shows capacity and usage for mounted filesystems. `tmpfs` is memory-backed temporary storage, not a persistent disk.
- A network interface is a physical or virtual connection endpoint used to send and receive traffic. `ip -br addr` summarizes interfaces, state, and addresses; `ip route` shows where Linux sends traffic.
- An SSH config entry is a connection shortcut, not DNS. `Host` sets the alias, `HostName` the destination address, and `User` the account on the remote host. Use it with `ssh <alias>`; SSH reads `~/.ssh/config` automatically.

## Question practice

- What resources does each host need for its VM while still running its own OS and services?
- How much CPU, RAM, and storage can be allocated safely, based on measured capacity and workload needs?
- How many VMs does the target design call for?
- Which interface and route carry traffic between the hosts?
- What evidence distinguishes total disk capacity from free space on a mounted filesystem?

## Hands-on work

Completed read-only checks on both hosts:

```bash
hostnamectl
cat /etc/os-release
uname -r
lscpu
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
df -hT
ip -br addr
ip route
```

Verified inventory and architecture implications are recorded separately in [the host inventory](../inventory/hosts.md).

SSH practice: added an alias in `~/.ssh/config` and connected using `ssh <alias>`. Verbose SSH output showed the config entry was loaded; the initial hang was caused by an incorrect destination IP. Correcting the address resolved it. Lesson: confirm the destination before investigating authentication or server configuration.

## Reflection

The inventory replaced assumptions with measurements and showed that the two machines are not interchangeable: one has more memory and CPU threads, while the other has substantially more disk capacity. VM sizes must leave resources for each host OS and be based on workload needs.

Still to verify: practical VM allocations under real host load, network stability and reachability, and KVM/libvirt readiness on both hosts. CPU virtualization flags are encouraging evidence, but do not prove that KVM/libvirt is installed and working.

Key recall: `lsblk` describes block devices; `df` describes mounted filesystems. `ip addr` describes interfaces and addresses; `ip route` describes network paths.

**Day 1 complete. Next: Day 2, inspect KVM, QEMU, and libvirt on both hosts.**
