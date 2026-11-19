# Day 3: Terraform Fundamentals

Study journal for Terraform concepts, workflow, and libvirt provider/connectivity assessment. Hostnames and private network addresses are intentionally omitted.

## Learning goal

## Command preference

For package discovery and package-status checks in this study, prefer APT commands such as `apt search`, `apt list --installed`, and `apt-cache policy`. Avoid `dpkg-query` when an APT command provides the needed information.

Learn Terraform's role, providers, resources, variables, outputs, state, and the `init`, `plan`, `apply`, and `destroy` workflow. Install Terraform after read-only checks. Assess the libvirt provider and connectivity; validate provider compatibility and Host B package requirements before recommending host changes.

## Course bookmark

Recommended course: [Terraform 101 for Beginners](https://www.youtube.com/watch?v=eIncaxqQ7AY). Use the chapter list shown in the course:

- `00:00` Hook — Why you need Terraform (optional)
- `01:20` What is Terraform? (watch)
- `02:10` Core Concepts: Provider, Resource, State, Module (watch Provider, Resource, and State; Modules are optional for Day 3)
- `03:48` HCL Syntax (watch)
- `05:15` The 4-Command Workflow (watch)
- `06:40` Demo — Your First Config (no cloud needed) (watch as a demo; do not run infrastructure-changing commands against HomeFlix hosts)
- `09:20` `terraform init` explained (watch)
- `11:23` `terraform plan` explained (watch)
- `12:50` `terraform apply` — deploy it! (watch to understand; do not apply to the HomeFlix hosts during the conceptual lesson)
- `14:30` State File — Terraform's memory (watch)
- `16:20` Input Variables & Output Values (watch)
- `19:00` Terraform Modules (skip or defer; not a Day 3 core objective)
- `24:00` Best Practices (watch if time permits)
- `25:40` Summary & What's Next (optional)

This is a reusable reference for later HomeFlix sessions. Return to chapters as needed, especially workflow, state, variables/outputs, and later modules.

## Progress

Initial read-only availability check: on both hosts, the user sourced a local `terraform.sh` check script and it reported “Terraform is not installed or is not in PATH.” Terraform v1.17.2 was subsequently installed and verified on Host A; Host B remains without Terraform based on the initial script-reported check.

## Provider and host checks

Read-only results supplied by the user:

- Host A is Ubuntu 24.04.5 LTS (`noble`). Installed package versions are `libvirt-clients` and `libvirt-daemon-system` 10.0.0-2ubuntu8.19, and `qemu-system-x86` 1:8.2.2+ds-0ubuntu1.18.
- Host B is Ubuntu 26.04.1 LTS (`resolute`). The queried packages are not installed. Its local apt indexes show candidates `libvirt-clients` and `libvirt-daemon-system` 12.0.0-1ubuntu5.6 and `qemu-system-x86` 1:10.2.1+ds-1ubuntu3.3.
- These package records show a two-release Ubuntu and libvirt/QEMU version difference. The candidate list establishes package availability in the local apt index, not that these are the complete or provider-compatible install requirements. No package index refresh or installation was performed during this check.

### What the package-check commands mean

- `dpkg-query -W -f='${Package} ${Version}\n' package...` queries the local dpkg database for the named packages. `-W` requests package records; `-f` controls the printed fields (package name and installed version). On Host A it confirmed the three queried packages are installed at the reported versions. `apt search package` can also show versions and an `[installed]` marker, as a later Host A check demonstrated. `dpkg-query` is a focused query for exact known package names; `apt search` is useful for package discovery and can include similarly named results. Separate search arguments are combined as an AND, so `apt search libvirt-clients libvirt-daemon-system qemu-system-x86` returned no match because no single package matched all three patterns. A later exact-name alternation regex returned all three requested packages, each marked `[installed]`, with versions matching the `dpkg-query` output. Both are read-only and show package records, not whether services run or whether a provider can connect.
- `apt-cache policy package...` reads APT's locally cached package metadata. `Installed: (none)` means dpkg has no installed version recorded. `Candidate:` is the version APT would select from the currently cached configured sources. The version table shows which source/version entries APT knows about and their priorities. This command is read-only; it neither refreshes the package index nor installs a package. Candidate availability does not prove the package set is complete or compatible with the selected Terraform provider.
- `cat /etc/os-release` reports the distribution/release identity (for example, codename `noble` or `resolute`); it does not report package versions.
- Initial Terraform availability was checked by sourcing a local script on both hosts; it reported Terraform absent from `PATH`. The script contents and direct `command -v terraform` result have not yet been reviewed.
- Current upstream libvirt provider documentation shows release 0.9.9 and documents SSH transports, while the Registry overview still contains a “remote connections (not yet implemented)” note. Treat the documentation mismatch as unresolved; choose and pin a specific release, then verify its actual transport and API compatibility before recommending Host B changes.
- Existing inventory records SSH from Host A to Host B as working. This establishes SSH reachability for that configured path, but not that the Terraform provider can connect to the libvirt API over that path. Host B still has no queried libvirt stack; its APT simulation and dependency set are recorded below.

Terraform CLI installation is complete on Host A; no VM or libvirt infrastructure changes have been made as part of Day 3 so far.

## Terraform installation preparation: package signing key

On Host A, the user completed `sudo apt update` and `sudo apt install gnupg software-properties-common`, then imported and inspected the HashiCorp signing key. The printed fingerprint matched HashiCorp's currently published value `D55C 0D1A C78A 8D81 26CB 631C FC9C A96A CA02 6560`; the key showed creation date 2026-09-09 and expiry 2031-09-08. The user added `/etc/apt/sources.list.d/hashicorp.list` on Host A with the verified key scoped through `signed-by` and the Ubuntu `noble` suite; `tee` echoed the saved line. APT indexes were refreshed; `apt-cache policy terraform` showed candidate `1.17.2-1` from the verified HashiCorp `noble/main amd64` source and `Installed: (none)`. The user installed Terraform and verified `Terraform v1.17.2` on `linux_amd64`. Provider initialization and a read-only query against Host A are complete. No VM or Host B package changes have been made. Use `apt` for interactive package-manager steps, per the user's preference.

- `sudo apt update` downloaded fresh package indexes from already configured repositories. It updates APT's local metadata cache, not installed packages; it does not itself install or upgrade software.
- `sudo apt install gnupg software-properties-common` completed according to the user. Without `-y`, APT shows its proposed changes and asks before proceeding. `gnupg` provides GPG tools to inspect/convert the key; `software-properties-common` is included in HashiCorp's documented Ubuntu setup prerequisites.
- `wget -O- https://apt.releases.hashicorp.com/gpg` downloads the repository's public signing key and writes it to standard output (`-O-`), rather than to a named file.
- `gpg --dearmor` removes the ASCII armor encoding (the readable-text wrapper around the key) and outputs the binary OpenPGP keyring format APT expects. It does not decrypt the public key and does not verify that the key is authentic; the fingerprint comparison is the identity check.
- `sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg` writes the converted key to a system keyring file that APT can read. `sudo` is needed because `/usr/share/keyrings` is system-owned. `> /dev/null` discards `tee`'s duplicate terminal output; it does not suppress the file write.
- `gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint` asks GPG to inspect this specific keyring, not the user's usual keyring, and print its fingerprint. In the multiline form, the backslash at the end of each line tells Bash the command continues on the next line; the shell passes the same options as one command. `gpg` starts the program; `--no-default-keyring` prevents it from also consulting the user's ordinary keyring; `--keyring PATH` selects the exact downloaded keyring file; `--fingerprint` prints the key's full fingerprint. The command reads the keyring and displays identifying information; it does not edit the key. Compare the result to HashiCorp's current official Linux package key fingerprint: `D55C 0D1A C78A 8D81 26CB 631C FC9C A96A CA02 6560`. HashiCorp rotated the Linux package key on 2026-09-10; the previously used fingerprint `798A EC65 4E5C 1542 8C8E 42EE AA16 FCBC A621 E701` is no longer the current fingerprint. A fingerprint mismatch means stop before adding the repository.

The key-fetch and conversion pipeline alone does not prove the downloaded key is HashiCorp's. The independent fingerprint comparison against HashiCorp's published security page is the identity check. The official sources are [HashiCorp Linux package indexes](https://apt.releases.hashicorp.com/index.html) and [HashiCorp security](https://www.hashicorp.com/en/trust/security).


### Terraform repository entry

The user added the repository entry on Host A:

`deb [arch=amd64 signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com noble main`

`tee` echoed the exact line it wrote to `/etc/apt/sources.list.d/hashicorp.list`, confirming the write command completed. The `signed-by` scope uses the verified HashiCorp key; `noble` matches Host A's Ubuntu 24.04 codename; `amd64` matches package architecture shown in the host's APT output. The user reports running `sudo apt update` and `apt-cache policy terraform`. The first excerpt showed older versions; a shortened follow-up confirmed `Installed: (none)` and candidate `1.17.2-1` from `https://apt.releases.hashicorp.com noble/main amd64`. Terraform v1.17.2 was then installed on Host A and verified with `terraform version`.


### Filtering the package-policy output

The suggested command `apt-cache policy terraform | sed -n '1,8p'` keeps the package query read-only and limits what is displayed. `|` sends `apt-cache` output into `sed`; `sed` is a stream editor that reads text line by line. `-n` disables its normal automatic printing, and `'1,8p'` tells it to print lines 1 through 8. It does not edit a file or change APT state. The aim is to show `Installed`, `Candidate`, and the first version/source entries without pasting the entire version table.


### Provider compatibility evidence to date

The latest upstream release found during the Day 3 check is `dmacvicar/libvirt` v0.9.9. The upstream README and transport guide document `qemu+ssh` and `qemu+sshcmd`; the transport guide says native `qemu+sshcmd` needs `ssh` locally and `nc` (netcat) or `virt-ssh-helper` remotely. The Registry overview for the latest release still includes a stale-looking “remote connections (not yet implemented)” comment, so the documentation is inconsistent. No formal minimum libvirt/QEMU version compatibility matrix was found in the reviewed upstream material. The provider was initialized under Terraform v1.17.2 and successfully performed a read-only query against Host A libvirt 10.0. The upstream libvirt documentation states its remote protocol is backward-compatible for older clients talking to newer servers, which supports (but does not directly prove) using this provider with Host B libvirt 12.0. A direct provider-to-Host-B query remains untested because Host B has no libvirt daemon installed. Treat API compatibility as likely at protocol level, with end-to-end Host B verification still required after its stack is provisioned. Sources reviewed: [upstream README](https://github.com/dmacvicar/terraform-provider-libvirt), [transport guide](https://github.com/dmacvicar/terraform-provider-libvirt/blob/main/docs/transports.md), [release list](https://github.com/dmacvicar/terraform-provider-libvirt/releases), [Registry overview](https://registry.terraform.io/providers/dmacvicar/libvirt/latest/docs).


### Remote transport helper check

The user ran `command -v nc` and `command -v virt-ssh-helper` in a Host A terminal; results were `/usr/bin/nc` and `/usr/bin/virt-ssh-helper`. This confirms the two helper executables are available on the Terraform runner, but does not answer whether Host B, the remote libvirt target, has the required helper. Host B was subsequently checked: `/usr/bin/nc` is available and `virt-ssh-helper` is not found. Because the upstream `qemu+sshcmd` transport documentation says the remote side needs `nc` or `virt-ssh-helper`, the documented helper prerequisite is met by `nc`. This alone does not prove libvirt is installed or reachable there; the queried libvirt/QEMU packages remain absent.


### Shell redirection in the temporary provider configuration

The suggested shell line `cat > main.tf <<'EOF'` uses two different operators:

- `>` redirects standard output to `main.tf`. If the file already exists, `>` replaces its contents; in the fresh `mktemp` directory it creates the file.
- `<<'EOF'` starts a here-document: the following lines are supplied as standard input to `cat` until a line containing only `EOF`. Quoting the delimiter (`'EOF'`) stops the shell from expanding `$variables`, backticks, or command substitutions inside the body.

Together, `cat` copies the literal Terraform block from the here-document into `main.tf`. This affects only the temporary scratch folder, not HomeFlix infrastructure. The provider-init task has been proposed but not yet reported as run.


### Provider initialization result

The user ran `terraform init` in a unique temporary directory with the required provider pinned to `dmacvicar/libvirt` `= 0.9.9`. It completed successfully under Terraform v1.17.2 on `linux_amd64`, downloaded v0.9.9, and wrote `.terraform.lock.hcl` in that temp directory. No project files or libvirt resources were changed. The output marked the provider signature `self-signed` (key ID `0833E38C51E74D26`): Terraform verifies a registry provider signature, but HashiCorp does not provide a chain of trust for a self-signed community provider, so the publisher identity is not verified by HashiCorp. Provider source and behavior still require review. Successful init verifies Terraform can install and initialize this provider build; it does not verify a libvirt API connection or host-version compatibility.

The user then added a provider block using `qemu:///system?socket=/var/run/libvirt/libvirt-sock-ro`, a read-only `libvirt_node_info` data source, and an output exposing the CPU model. `terraform plan` successfully read Host A node info and planned only output `provider_connection_check = "x86_64"`; no `terraform apply` was run. This confirms provider v0.9.9 can connect to Host A libvirt 10 using the read-only socket and read this data source. It does not confirm remote SSH connectivity or Host B compatibility. The connection may activate a systemd socket/service; subsequent observed service state was not reported.


### APT package simulation on Host A

The user ran `apt install --simulate libvirt-clients libvirt-daemon-system qemu-system-x86` in a Host A terminal. APT reported all three packages already at the installed versions, `0 upgraded, 0 newly installed, 0 to remove, 58 not upgraded`. It also noted `libfwupd2` as automatically installed and no longer required. Because this was a simulation, no package changes occurred; no `apt autoremove` was run. This result describes Host A only and does not establish Host B's installation set. The Host B simulation is recorded below.


### APT package simulation on Host B

The user shared the complete output of `apt install --simulate libvirt-clients libvirt-daemon-system qemu-system-x86` from Host B. APT's simulation summary was `Upgrading: 0, Installing: 83, Removing: 0, Not Upgrading: 36`. The explicit package roots were the three queried packages, with candidate versions `libvirt-clients` and `libvirt-daemon-system` 12.0.0-1ubuntu5.6 and `qemu-system-x86` 1:10.2.1+ds-1ubuntu3.3. The simulated dependency closure includes `libvirt-daemon-driver-qemu`, network and storage drivers, `qemu-utils`, firmware/OVMF, TPM tools, and supporting packages. Suggested packages are shown separately and are not part of the simulated install list. Simulation did not install packages or change Host B. The output warns that simulation disables locking and may not reflect the exact state at a later install; refresh indexes and review a fresh simulation immediately before any future package installation.

### Day 3 practical assessment status

Terraform v1.17.2 is installed on Host A. Provider v0.9.9 initialized in a scratch directory and its read-only `libvirt_node_info` data source successfully queried Host A using the read-only socket; no `apply` or VM launch occurred. Host B has `nc`, satisfying one documented `qemu+sshcmd` remote helper option, and the existing inventory records SSH access from Host A. A provider-to-Host-B API query is not possible yet because the required libvirt stack is absent. The simulated Host B package roots are `libvirt-clients libvirt-daemon-system qemu-system-x86`; the full simulation reports 83 new packages, no upgrades/removals, and 36 held back. No Host B package installation is recommended or performed in Day 3; defer actual package installation to the host-provisioning step, then make a read-only Terraform plan before creating any VM.

## Terraform theory notes (learner summary and clarifications)

The learner's summary covered IaC, providers, resources, modules, state, variables, outputs, workspaces, and the core CLI workflow. Refine the key ideas as follows:

- **Infrastructure as code (IaC):** Terraform uses declarative configuration to describe desired infrastructure, which helps make changes reviewable and repeatable. Terraform manages infrastructure through providers; keep ownership boundaries clear with Ansible (guest OS configuration), Kubernetes (workloads), and other controllers.
- **Provider:** A plugin that exposes a system's API and resource/data-source schemas to Terraform. It connects Terraform configuration to an API; it does not itself define all the infrastructure.
- **Resource:** A declared object Terraform manages through a provider, such as a libvirt VM domain, volume, or network. A resource's name and type identify it in configuration/state.
- **Module:** A reusable package of Terraform configuration (resources, variables, outputs, and possibly child modules). It is more than a named logical block; it packages an interface and implementation for reuse.
- **State:** A mapping between Terraform resource instances and real objects, with metadata and cached attributes. State is not what makes Terraform idempotent. Repeated plans generally converge to no changes when configuration, state, and real infrastructure agree; drift, provider behavior, or changing inputs can still cause changes. Local `.tfstate` files can contain secrets, so do not commit state to Git. Protect state with access controls, encryption, backups, and locking where supported.
- **Variables and `.tfvars`:** `variable` blocks declare inputs and their type/description/default; `terraform.tfvars` and `*.auto.tfvars` assign values. A tfvars file can override a variable's default, but CLI/HCP values and auto tfvars can take precedence over `terraform.tfvars`. Do not hardcode values that should vary by environment, but also do not turn every constant into a variable. Never commit secret values; `.tfvars` files are safe to commit only when their contents are non-sensitive.
- **Outputs:** `output` blocks expose selected values for CLI use, parent modules, automation, or other configurations. Outputs are shown after apply and may also appear in plans; applied output values are stored in state. Mark sensitive outputs to redact them in normal CLI output, but remember the values remain in state and can be exposed by raw/JSON output commands.
- **Workspaces:** CLI workspaces provide separate state instances for the same configuration and backend. They are not, by themselves, strong security or environment isolation; different environments may need separate configuration/backend credentials and access controls. Workspaces are not mandatory for every Terraform project.
- **Workflow:** `init` prepares the directory, configures the backend, and installs providers/modules. `plan` refreshes/compares the known infrastructure and previews actions. `apply` normally generates a plan and asks for approval before executing it; a saved plan can be applied directly. `destroy` plans and removes resources managed by the current configuration/workspace; review destruction plans carefully.
- **Files and sources:** `terraform.tfstate` and plan files can hold sensitive data; `.terraform.lock.hcl` records provider selections/checksums and is normally committed. Official docs are at [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform); provider and module listings/documentation are at the [Terraform Registry](https://registry.terraform.io/).

### Additional concepts to include in the theory review

- **Data source:** A provider-backed read query that fetches information about an existing object or system without asking Terraform to create or manage that object. Use one to reference existing infrastructure or inspect facts needed by a configuration. Data sources still require provider connectivity and can reveal sensitive details.
- **Backend:** The configuration that determines where Terraform stores state and how it accesses that state. The default is local state; a remote backend can centralize state for collaboration and may support locking. Choose and protect the backend before multiple people or automation run Terraform against the same infrastructure.
- **State locking:** A backend feature that prevents concurrent Terraform operations from writing to the same state at once. It reduces the risk of conflicting updates; not every backend supports locking.
- **Dependency graph:** Terraform derives ordering relationships between resources from references and explicit dependencies. It uses the graph to decide what can run in parallel and what must wait. A reference such as a VM using a volume's ID creates an implicit dependency.
- **Drift:** A difference between the configuration/state Terraform expects and the real infrastructure, often caused by manual changes or external systems. Planning refreshes observed information and can propose changes to reconcile drift. Review the plan to decide whether to restore the declared configuration or update the configuration to match an intentional change.
- **`terraform fmt`:** Formats Terraform configuration files into a consistent canonical style. Use it to make configuration easier to review; it does not validate provider behavior or change infrastructure.
- **`terraform validate`:** Checks configuration syntax and internal consistency against the installed provider schemas. It does not contact the infrastructure to prove connectivity or guarantee that an apply will succeed.
- **`.terraform.lock.hcl`:** A dependency lock file that records the selected provider versions and checksums so future `terraform init` runs can select and verify the same provider packages. Normally commit this file. It does not contain Terraform state.
- **Version pinning:** Version constraints in the Terraform and provider blocks limit which tool/provider versions are accepted. Pinning a tested provider release and committing the lock file makes installations more repeatable; periodically review and update pins deliberately.

## Day 3 theory review status

The learner reports studying the Terraform fundamentals and the additional definitions for data sources, backends, state locking, dependency graphs, drift, `fmt`, `validate`, `.terraform.lock.hcl`, and version pinning on 2026-11-19. Day 3 practical checks and theory review are complete. Reflection prompt: Which Terraform assumption or failure case do you most want to verify before provisioning the first HomeFlix VM?

### Learner reflection and terminology clarification

The learner's reflection before the first HomeFlix VM: “Provider is correct and is configured properly in backend. Ensure state lock to make sure it is not fickle.” The underlying concerns are provider correctness and avoiding unreliable/concurrent state changes. Keep the concepts distinct:

- Provider configuration tells Terraform which provider plugin to use and how to connect to the target API (for this project, libvirt and its connection URI/socket/transport). It is not configured in the backend.
- Backend configuration tells Terraform where and how to store state. A remote backend may offer state locking; the current scratch-directory experiment used Terraform's default local backend and did not configure shared remote state or multi-user locking.
- State locking prevents concurrent operations on a supported backend from writing to the same state at once. It reduces race/conflicting-write risk, but does not verify provider correctness, prevent all errors, or by itself make state reliable. State reliability also depends on the backend's durability, access controls, backups, and correct configuration.

Day 3 is complete. No infrastructure was created and Host B's virtualization stack remains uninstalled pending its planned host-provisioning work.
