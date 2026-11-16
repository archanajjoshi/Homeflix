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

The Host A inventory also showed a `virbr0` interface and Docker bridge/veth interfaces. These are clues for later inspection, not proof that libvirt or VM networking is configured and working.

## Planning implications and open checks

- Capacity is asymmetric: Host B has less RAM and fewer logical CPUs; Host A has more RAM and logical CPUs. Host B has substantially more disk space. Size the cluster for the tighter compute host and reserve resources for each host OS.
- The hosts run different Ubuntu releases. Confirm compatibility with the selected virtualization and cluster tools; decide whether to standardize before automating both hosts.
- Both CPUs report hardware virtualization support. Verify KVM, QEMU, libvirt services, permissions, and networks during Day 2.
- Validate actual host load, usable memory, network stability, and VM resource requirements before choosing CPU, RAM, and disk allocations.
- A two-node K3s design is the target; this inventory alone does not establish high availability or storage resilience.
