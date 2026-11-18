# Homeflix host inventory

Public project snapshot from the Day 1 baseline. Host labels and private network addresses are generalized. Memory and filesystem figures are observations from the time of capture, not fixed capacity guarantees.

## Target architecture

The project plan calls for two physical Ubuntu hosts, each running one VM, with the VMs forming a K3s cluster. The VMs and cluster are planned and have not yet been confirmed as deployed.

## Physical hosts

| Host | Hardware and OS | CPU and virtualization | Memory at capture | Storage at capture | Network |
|---|---|---|---|---|---|
| Host A | HP ProBook 640 G5; Ubuntu 24.04.5 LTS; x86-64; kernel `7.0.0-42-generic` | Intel Core i5-8365U; 4 cores / 8 logical CPUs; VT-x reported | 15 GiB total; 7.7 GiB available; 4 GiB swap, 3.5 GiB used | 238.5 GB disk; 155 GB available on ext4 root filesystem | Wi-Fi active; same home LAN subnet as Host B |
| Host B | Dell Inspiron 5765; Ubuntu 26.04.1 LTS; x86-64; kernel `7.0.0-43-generic` | AMD A12-9700P; 4 cores / 4 logical CPUs; AMD-V reported | 6.7 GiB total; 3.7 GiB available; 4 GiB swap, unused | 931.5 GB disk; 854 GB available on ext4 root filesystem | Wi-Fi active; same home LAN subnet as Host A |

## Network observations

Both hosts use Wi-Fi on the same home subnet and report the same default gateway. This supports direct LAN communication, subject to Wi-Fi client-isolation and firewall settings. SSH from Host A to Host B works using an SSH config alias.

The Host A inventory also showed a `virbr0` interface and Docker bridge/veth interfaces. Day 2 confirmed that `virbr0` is used by the active default libvirt NAT network; VM-to-VM reachability has not been tested. The Docker interfaces remain separate clues for later inspection.

## Planning implications and open checks

- Capacity is asymmetric: Host B has less RAM and fewer logical CPUs; Host A has more RAM and logical CPUs. Host B has substantially more disk space. Size the cluster for the tighter compute host and reserve resources for each host OS.
- The hosts run different Ubuntu releases. Confirm compatibility with the selected virtualization and cluster tools; decide whether to standardize before automating both hosts.
- Both CPUs report hardware virtualization support. `/dev/kvm` exists on both, and each user's reported read/write permission check succeeded. These facts establish CPU capability and current-user device access, but do not prove that a QEMU guest can launch.
- Day 2 package/service checks show different user-space readiness: Host A has QEMU and libvirt packages installed (except the `qemu-kvm` metapackage); its `libvirtd` service was inactive when checked after the live queries, while the relevant libvirt sockets had been listening and the read-only queries succeeded. Host B has none of the queried QEMU/libvirt packages, no `virsh`, and no matching services or sockets. Host B is not ready to provision a VM; validate package selection against its Ubuntu release and the chosen provider before installing.
- Host A's `default` NAT network is active, persistent, and set to autostart. Its private subnet details are intentionally omitted.
- Host A has active directory storage pools named `boot`, `default`, and `Downloads`, set to autostart and pointing to standard libvirt boot and image directories plus a user Downloads directory. Pool persistence and usable capacity are not confirmed by the checks so far.
- Validate actual host load, usable memory, network stability, and VM resource requirements before choosing CPU, RAM, and disk allocations.
- A two-node K3s design is the target; this inventory alone does not establish high availability or storage resilience.
