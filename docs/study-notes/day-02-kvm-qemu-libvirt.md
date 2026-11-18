# Day 2: KVM, QEMU, and libvirt

Study journal for inspecting virtualization readiness on both Homeflix physical hosts. Hostnames and private network addresses are intentionally omitted.

## Learning goal

Understand the roles of KVM, QEMU, and libvirt; verify virtualization and KVM/libvirt readiness on both hosts; inspect relevant services, networks, and storage pools. Begin with read-only checks.

## Project-plan sequence

The project plan calls for one KVM/QEMU/libvirt VM on each physical Ubuntu host. Therefore, Host B will need a working QEMU/libvirt host stack before its VM is provisioned. Day 2 records the current baseline without installing packages. Package selection and compatibility should be validated against Host B's Ubuntu release and the selected Terraform libvirt provider before installation; the plan assigns provider and connectivity assessment to Day 3 and the second VM to Day 6. Keep this exercise on Day 2 until its checks and reflection are complete.

## Concepts and useful questions

- CPU virtualization support is a processor capability. `lscpu` reports `VT-x` on the Intel host and `AMD-V` on the AMD host. These results do not establish that Linux currently exposes KVM or that a VM can be managed.
- `/dev/kvm` is the Linux interface used for KVM acceleration. Its presence is evidence of KVM device availability, but does not by itself prove that QEMU can launch a VM or that libvirt is configured and working.
- KVM provides Linux kernel support for hardware-accelerated virtualization; QEMU runs the VM and emulates its devices; libvirt manages VM definitions and related virtual networks and storage pools.
- Windows analogy: Hyper-V is the closest overall comparison to the KVM/QEMU/libvirt stack, but the mapping is not one-to-one. KVM is closest to the Hyper-V hypervisor; QEMU provides the VM process and virtual hardware, with no single exact Windows equivalent; libvirt is closest to Hyper-V management tools and APIs. KVM alone is not the entire VM host.
- `virsh` is libvirt's command-line client. It can query and manage the hypervisor, VM definitions, virtual networks, and storage pools. In this exercise we will use only query commands. A connection may activate a systemd socket or service, so inspect socket units before trying a connection when preserving the observed service state matters.
- For each check, ask what it proves, what it does not prove, what assumptions it relies on, and what failure cases could produce the result.

## Hands-on work and evidence

### Step 1: CPU virtualization capability

Read-only `lscpu` checks were run on both hosts. Both report architecture `x86_64`; the Intel host reports `VT-x` and the AMD host reports `AMD-V`.

**Interpretation:** CPU virtualization support is reported on both hosts. This is not evidence yet that KVM is available to Linux or that QEMU/libvirt can operate.

The follow-up checks found `vmx` on the Intel host and `svm` on the AMD host. `/dev/kvm` exists on both, with mode `crw-rw----+`, owner `root`, and group `kvm`. On the AMD host, `stat` reported mode `660`. The `+` in the device listing indicates an ACL is present. Later checks confirmed current-user read/write access on both hosts. The AMD `grep` printed `svm` twice; because `-o` prints each occurrence on the first matching line, this still establishes that the flag is reported.

**Interpretation:** Both CPUs report hardware virtualization flags, and both Linux hosts expose the KVM device node. Later permission checks confirmed current-user access on both. This still does not prove that QEMU can launch an accelerated guest or that libvirt is installed and working.

### Step 2: QEMU/libvirt packages and services

The planned read-only package check uses `dpkg-query -W` to query the local package database for QEMU, libvirt, and VM installation tools; its format prints status, package name, and version. `2>&1` includes error messages such as an unrecognized or absent package in the captured output. The planned `systemctl list-units --type=service --all --no-pager` check lists service units including inactive ones; `grep -Ei` filters for libvirt/QEMU service names, and `|| true` makes no matches a normal outcome. Neither command installs or changes anything.

Installation with `sudo apt install` would change the hosts and alter the baseline, so the exercise first records what is present.

Read-only results on Host A: `libvirt-clients`, `libvirt-daemon-system`, `qemu-system-x86`, and `virtinst` are installed. `qemu-kvm` is not installed (`un` status). The service listing showed `libvirt-guests` active/exited, `libvirtd` loaded but inactive/dead, and `virtlockd` and `virtlogd` running. The inactive legacy monolithic `libvirtd` unit did not alone establish that libvirt was unusable; later read-only `virsh` queries succeeded.

Read-only results on Host B: `dpkg-query` found none of the five queried packages, and the filtered service listing showed no matching services. This does not negate the observed `/dev/kvm` device node; it means the queried user-space QEMU/libvirt stack was not found in these package and service checks.

Package presence alone does not prove a service is running or a VM can start. An inactive service can be installed but stopped; socket activation may also allow a client connection when a service unit appears inactive.

### Step 3: Check libvirt command and socket availability

`command -v virsh` checks whether the `virsh` executable is available in the current shell's `PATH`. `systemctl list-units --type=socket --all --no-pager` lists socket units, including inactive ones; the `grep` filter keeps likely libvirt-related entries. Checking sockets before connecting helps identify socket activation, where a client connection can cause systemd to start a service. These checks only inspect state and do not install packages or connect to libvirt.

Results: Host A has `/usr/bin/virsh`; its legacy `libvirtd` admin, read-only, and standard sockets are active/listening. The `virtlockd` and `virtlogd` sockets are active as well. `virtqemud.socket` is not found. Host B has no `virsh` found in `PATH` and no matching libvirt/QEMU sockets in the listing. Host A's `libvirtd.socket` was listening while `libvirtd.service` was inactive, so a client connection could activate the daemon.

### Step 4: Inspect saved network and storage-pool definitions

Libvirt can store persistent network and storage-pool definitions as files. Listing those directories is a read-only way to see saved definitions; it does not establish which definitions are active at runtime. A live `virsh` query can show runtime state, but on Host A may activate the listening libvirt socket and start a daemon.

Results: Host A has `/etc/libvirt/qemu/networks/default.xml` and three storage-pool XML files (`boot.xml`, `default.xml`, `Downloads.xml`), with `autostart` subdirectories in both configuration locations. This establishes saved definition files are present, not that the definitions are active or usable. Host B has neither queried directory, consistent with the absence of the queried libvirt packages; no saved definitions were found at those locations.

On Host B, a `chmod 666` attempt against the missing directories returned “cannot access”; it made no permission change. `chmod` changes permissions and is not needed for this read-only exercise. Keep inspecting with listing or read commands only.

For the saved XML inspection, `sudo` allows reading root-owned mode-`600` files; `grep` reads and filters without changing them. `-H` prints the source filename with each matching line. `-E` enables extended regular expressions so `|` means “or” among the requested XML tags. The trailing `*` expands to the storage XML files; the backslash continues the command onto another shell line.

Host A's network XML shows a saved network named `default`, configured for NAT and using bridge `virbr0`, with a private subnet and DHCP range (exact addresses omitted from this public note). This is saved configuration evidence; the later live query confirmed that the network is active. The first storage-file attempt did not run because the typed path ended in `*.xml.` with an extra period, so no file matched.

Host A's storage XML inspection found saved pools named `boot`, `default`, and `Downloads`. All three have `type='dir'`, so they use directory-backed storage. Their paths are the standard libvirt boot directory, VM images directory, and a user Downloads directory; the personal home path is omitted from this public note. Saved definitions do not establish that a pool is active or has usable capacity.

Read-only `virsh --readonly -c qemu:///system` queries succeeded on Host A. The `default` network is active, persistent, and set to autostart. The `boot`, `default`, and `Downloads` directory pools are all active and set to autostart. `pool-list --all` output did not include a persistence column, so persistence for those pools is not directly confirmed by that output. A subsequent `systemctl is-active libvirtd` returned `inactive`. The service state was inactive at that observation; the earlier listening socket and successful read-only queries remain evidence that the system libvirt connection worked at query time. Socket activation and idle shutdown may account for service state changing over time; we have not established which occurred.

Host A's `kvm_device.sh` output says `/dev/kvm` is readable and writable by the current user; its displayed script contents confirm it uses `test -r` and `test -w`. Host B's `kvm_devicebb.sh` output also says the device is readable and writable by the current user; that script's contents were not displayed. In Bash, `source file` runs the file in the current shell. These permission checks show current-user access but do not prove a QEMU guest can launch.

## Day 2 status (checks complete; reflection pending)

CPU virtualization flags, KVM device nodes, and reported current-user read/write access to `/dev/kvm` are present on both hosts. Host A has QEMU/libvirt user-space packages, a listening libvirt socket, successful read-only system libvirt queries, an active persistent NAT network set to autostart, and three active directory pools set to autostart. `libvirtd` was inactive when checked after the queries; the reason for the service state at that later observation is unknown. Host B has none of the queried QEMU/libvirt packages, no `virsh`, no matching service/socket units, and no queried libvirt configuration directories; it is not ready to provision a VM. No packages have been installed during Day 2. Neither host's VM guest launch was tested.

## Reflection

Reflection prompt: Which evidence best helped distinguish CPU virtualization capability from a usable KVM/libvirt setup, and what would you verify before deciding on Host B's package set?
