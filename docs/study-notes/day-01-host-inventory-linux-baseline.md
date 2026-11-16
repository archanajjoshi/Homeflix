# Public portfolio copy

Hostnames and private network addresses have been generalized for public sharing. The detailed local study notes remain in `HomeFlix_Study_Notes.md`.

Day-1 Homeflix-Host Inventory and Linux Baseline:

Learning goal:
Build a factual inventory of both Ubuntu hosts before planning virtual machines or changing configuration.

Notes:
- Host inventory records the real OS, CPU, memory, storage, networking, and virtualization capabilities of each machine.
- We use measured facts to check assumptions, compare hosts, and make infrastructure decisions based on evidence.
- Start with read-only inspection; record results before making changes.

Question practice:
- What resources will each host need to run its VM alongside the host operating system and other services?
- How much RAM and storage should be allocated, and what networking setup will the VMs need?
- How many VMs are needed? The project plan describes two physical Ubuntu hosts, each running one VM for the K3s cluster (two VMs total).
- What kernel version is each host running?
- Ask what evidence would answer a question. For example, `uname -r` reports the running kernel release; host and VM capacity questions require inventory and workload estimates.

Hands-on work and verified results:
- `hostnamectl`: reports host identity and system information, including the hostname and operating-system/kernel details where available.
- `cat /etc/os-release`: prints OS identification and release information.
- `uname -r`: prints the running kernel release.
- Step 1 results reviewed:
  - `host-b` (Dell Inspiron 5765): Ubuntu 26.04.1 LTS, x86-64, kernel `7.0.0-43-generic`.
  - `host-a` (HP ProBook 640 G5): Ubuntu 24.04.5 LTS, x86-64, kernel `7.0.0-42-generic`.
  - Both are Ubuntu laptops, but they run different Ubuntu releases. Record this as a compatibility/reproducibility consideration for later VM and cluster setup; decide whether to standardize after checking project requirements and host support.
  - Exact host and VM CPU, RAM, and storage sizing remain to be determined from inventory and workload needs.
- Step 2 CPU and memory results reviewed (from `lscpu` and `free -h`):
  - `host-a`: Intel Core i5-8365U, 4 cores / 8 logical CPUs, VT-x available; 15 GiB RAM total, 7.7 GiB available at capture; 4 GiB swap, 3.5 GiB used at capture.
  - `host-b`: AMD A12-9700P, 4 cores / 4 logical CPUs, AMD-V available; 6.7 GiB RAM total, 3.7 GiB available at capture; 4 GiB swap, unused at capture.
  - The HP has more reported CPU threads and memory; host-b is the tighter memory constraint for VM sizing. `available` is more useful than `free` alone as a rough estimate of RAM available for new work because Linux uses spare RAM for cache. Swap use is recorded as an observation, not a diagnosis; check live memory pressure before drawing conclusions.
- Step 3 disk layout reviewed (`lsblk`):
  - `host-a` (`disk.txt`): one 238.5 GB disk, with a 1 GB EFI partition and a 237.4 GB ext4 root partition.
  - `host-b` (`diskbb.txt`): one 931.5 GB disk, with a 1 GB EFI partition and a 930.5 GB ext4 root partition; an optical drive is also listed.
  - The many `loop` entries are mounted Snap package images, not extra physical disks.
- Step 3 mounted filesystem results reviewed (`df -hT`):
  - HP root filesystem (`/dev/sda2`, ext4): 233 GB reported, 67 GB used, 155 GB available (30% used) at capture.
  - `host-b` root filesystem (`/dev/sda2`, ext4): 915 GB reported, 15 GB used, 854 GB available (2% used) at capture.
  - `lsblk` reports block-device/partition size; `df` reports the size and free capacity of mounted filesystems. Their displayed sizes differ slightly due to filesystem overhead and unit/rounding differences.
  - `tmpfs` entries are memory-backed temporary filesystems, not additional persistent disks.
- Step 4 network interfaces and routes reviewed (`ip -br addr`, `ip route`):
  - A network interface is a host's network connection endpoint, represented in Linux as a named device. It can be physical (Wi-Fi/Ethernet) or virtual (loopback, bridges, container links, or VM networking).
  - `ip -br addr` gives a brief view of interface names, link state, and assigned IP addresses. `ip route` shows which interface/path Linux uses to reach local and remote networks.
  - HP: Wi-Fi interface `wlp0s20f3` is up at `<host-a-ip>/<subnet-prefix>`; default route goes via `<gateway-ip>`. Ethernet `enp0s31f6` is down. Virtual/container interfaces include `virbr0`, Docker bridges, and veth devices.
  - `host-b`: Wi-Fi interface `wlp2s0` is up at `<host-b-ip>/<subnet-prefix>`; default route also goes via `<gateway-ip>`. Ethernet `enp1s0` and `wlx6c5ab00ef2a9` are down.
  - Both hosts are on the same `<home-lan-subnet>` subnet and use the same default gateway, which supports direct communication on the home LAN (subject to Wi-Fi/client isolation and firewall settings).
- SSH convenience: add an alias in `~/.ssh/config` so you can connect with `ssh host-b` instead of repeatedly typing a username and IP. Example (replace the placeholders with the real login and address):
  ```sshconfig
  Host host-b
      HostName <host-ip>
      User <username>
  ```
  Use it with `ssh host-b`. Do not run `host-b` by itself or `source ~/.ssh/config`; the SSH client reads this file automatically whenever you run `ssh`.
  This is an SSH shortcut, not DNS. DNS maps a name to an address. A router DNS entry or DHCP reservation can provide a stable home-network name; later, Tailscale MagicDNS can provide names to authorized tailnet devices.
- Troubleshooting result: `ssh -v host-b` confirmed SSH read `~/.ssh/config` and applied the `host-b` entry. It stalled while connecting because the configured IP was incorrect. Correcting the IP resolved the connection. Lesson: verify the destination address before investigating authentication or server-side SSH settings.

Initial architecture notes (Day 1):
- Target from the project plan: run one Ubuntu VM on each physical host, then connect the two VMs as a K3s cluster. The VMs and cluster are planned; they are not yet confirmed as built or running.
- Candidate host A, HP ProBook: Ubuntu 24.04.5 LTS; 4 CPU cores / 8 logical CPUs; VT-x present; 15 GiB RAM with 7.7 GiB available at capture; 238.5 GB disk with 155 GB free on `/`; active Wi-Fi on `<home-lan-subnet>`.
- Candidate host B, Dell Inspiron (`host-b`): Ubuntu 26.04.1 LTS; 4 CPU cores / 4 logical CPUs; AMD-V present; 6.7 GiB RAM with 3.7 GiB available at capture; 931.5 GB disk with 854 GB free on `/`; active Wi-Fi on the same `<home-lan-subnet>` subnet.
- The hosts can address each other on the same home subnet, and SSH access from the HP to `host-b` works with an SSH config alias. Validate network reachability and stability as the design progresses.
- Capacity is asymmetric: host-b has substantially less RAM and fewer logical CPUs, while it has substantially more disk space. Size workloads for the tighter host and reserve memory/CPU for each host OS; do not assign all reported capacity to VMs.
- The two Ubuntu releases differ. Confirm project/tool compatibility and choose whether to standardize before relying on identical host configuration.
- Both CPUs report hardware virtualization support, but KVM/libvirt availability and permissions have not yet been verified; that is Day 2 work. The HP also reports `virbr0`, which is a clue to inspect, not proof that libvirt is configured and working.

Reflection / recall:
- What we learned: an inventory turns guesses about capacity and connectivity into evidence. It also shows that the two machines are not interchangeable: host-b is the storage-rich but memory/CPU-constrained host, while the HP has more memory and logical CPUs but less disk space.
- What remains unknown: sustainable VM allocations under real host load, whether wireless connectivity is stable enough for the cluster, whether any Wi-Fi client isolation or firewall rules block node traffic, and whether KVM/libvirt is ready on both hosts.
- Key distinction to recall: `lsblk` describes block devices and partitions; `df -hT` describes mounted filesystem capacity and usage. `ip -br addr` shows interfaces and addresses; `ip route` shows paths Linux uses to reach networks.
- Day 1 is complete. Next: Day 2, inspect KVM, QEMU, and libvirt on both hosts before creating VMs.

---------------------------

Day2- KVM, QEMU and libvirt:

Learning goal:
Understand the hypervisor stack and inspect KVM/libvirt on both hosts.

Notes:
- To be completed during Day 2.

Hands-on work and verified results:
- Not started.

Reflection / recall:
- Not started.
