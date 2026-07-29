# Ansible Ad-Hoc Commands Assignment

**Student:** Egwu Chidiebere Agha
**Course:** DevOps / Configuration Management
**Date:** July 2026

## Table of Contents

1. [Overview](#1-overview)
1. [Infrastructure Setup — Part A](#2-infrastructure-setup--part-a)
1. [Ad-Hoc Commands — Part B](#3-ad-hoc-commands--part-b)
1. [Ansible Concepts Reference](#4-ansible-concepts-reference)
1. [Files Submitted](#5-files-submitted)
1. [Troubleshooting Notes](#6-troubleshooting-notes)
1. [How to Verify](#7-how-to-verify)
1. [Cleanup](#8-cleanup)

## 1. Overview

This project demonstrates the use of **Ansible** for configuration management using **ad-hoc commands only** — no playbooks were written or used. Ad-hoc commands are single, one-off Ansible tasks run directly from the command line with `ansible`, as opposed to `ansible-playbook`, which runs a sequence of tasks defined in a YAML file.

### 1.1 Objectives

1. Provision an Ansible **Controller** and **2 managed nodes** on AWS EC2.
1. Configure passwordless SSH access from the controller to each managed node using a key pair.
1. Build an **inventory file** that organizes hosts into logical groups.
1. Execute **12 ad-hoc Ansible commands** covering connectivity checks, command execution, fact-gathering, package management, service management, file management, user management, and configuration file editing.
1. Demonstrate core Ansible principles: **idempotency**, **parallelism (forks)**, **host patterns**, **privilege escalation**, and **modules**.


## 2. Infrastructure Setup — Part A

### 2.1 Architecture

```
                    [Local Laptop / WSL]
                            |
                            |  SSH with .pem key
                            v
        +---------------------------------------+
        |  Controller: node1 (3.91.55.23)        |
        |  - Ansible installed here              |
        |  - Holds inventory + commands.txt      |
        +---------------------------------------+
             |                              |
             | SSH (web group)               | SSH (db group)
             v                              v
   [node1 - 3.91.55.23]           [node2 - 3.95.198.221]
   Managed node (web group)        Managed node (db group)
             |
             | ansible_connection=local
             v
        [localnode]
   (the controller managing itself,
    no SSH hop required)
```

### 2.2 AWS Prerequisites

1. **Launch 2 EC2 instances** (Ubuntu 22.04/24.04 LTS recommended), e.g. `t2.micro`/`t3.micro` for free-tier eligibility.
1. **Key Pair**: Create or reuse a single `.pem` key pair (`ansible-assignment.pem`) and assign it to both instances at launch. Using one key for both nodes simplifies the inventory.
1. **Security Group** (applied to both instances):
   
   |Type|Protocol|Port|Source                                             |
   |----|--------|----|---------------------------------------------------|
   |SSH |TCP     |22  |My IP (or the controller’s SG, for tighter scoping)|
1. Note the **public IPv4 addresses** assigned to each instance — these go into the inventory as `ansible_host`. Elastic IPs are recommended if instances will be stopped/started, since public IPs otherwise change on restart.

### 2.3 Initial Setup Steps

1. **Set key permissions locally** (required — SSH refuses keys that are group/world-readable):
   
   ```bash
   chmod 400 ~/Downloads/ansible-assignment.pem
   ```
1. **SSH into the controller** to confirm access before doing anything else:
   
   ```bash
   ssh -i ~/Downloads/ansible-assignment.pem ubuntu@3.91.55.23
   ```
1. **Copy the key from the controller to itself** so it can reach `node2` without depending on the laptop:
   
   ```bash
   scp -i ~/Downloads/ansible-assignment.pem \
       ~/Downloads/ansible-assignment.pem \
       ubuntu@3.91.55.23:/home/ubuntu/.ssh/ansible-assignment.pem
   ```
   
   Then, **on the controller**, lock down permissions again (SCP does not always preserve the mode bit):
   
   ```bash
   chmod 400 /home/ubuntu/.ssh/ansible-assignment.pem
   ```
1. **Install Ansible on the controller**:
   
   ```bash
   sudo apt update
   sudo apt install ansible -y
   ansible --version
   ```
1. **Create the working directory** on the controller:
   
   ```bash
   mkdir -p ~/ansible-lab
   cd ~/ansible-lab
   ```
1. **Create the inventory file** (see §2.4) at `~/ansible-lab/inventory`.
1. **Test connectivity** from the controller to both nodes before running anything else:
   
   ```bash
   ansible -i inventory datacenter -m ping
   ```
   
   A successful result looks like:
   
   ```
   node1 | SUCCESS => {
       "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python3" },
       "changed": false,
       "ping": "pong"
   }
   node2 | SUCCESS => {
       "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python3" },
       "changed": false,
       "ping": "pong"
   }
   ```

### 2.4 Inventory File: `inventory`

Groups organize hosts so commands can target specific sets rather than naming hosts individually. `[datacenter:children]` is a **group of groups** — it lets a single command target both `web` and `db` together.

```ini

[web]
node1 ansible_host=3.91.55.23 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/ansible-assignment.pem

[db]
node2 ansible_host=3.95.198.221 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/ansible-assignment.pem

[local]
localnode ansible_connection=local

[datacenter:children]
web
db
```

**Line-by-line explanation:**

|Line / Variable                                        |Purpose                                                                                                                                                        |
|-------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`[web]`                                                |Group name; here it contains one host, `node1`                                                                                                                 |
|`ansible_host`                                         |The actual IP/hostname Ansible connects to (the inventory *name*, e.g. `node1`, can differ from the connection address)                                        |
|`ansible_user`                                         |The SSH login user — `ubuntu` is the default for Ubuntu AMIs                                                                                                   |
|`ansible_ssh_private_key_file`                         |Path to the private key **as it exists on the controller**, not the laptop                                                                                     |
|`[db]`                                                 |Group containing `node2`                                                                                                                                       |
|`[local]`                                              |Group containing a pseudo-host, `localnode`                                                                                                                    |
|`ansible_connection=local`                             |Tells Ansible to execute modules directly on the controller with no SSH transport at all                                                                       |
|`[datacenter:children]`                                |Declares a parent group whose members are the groups `web` and `db` — targeting `datacenter` reaches both                                                      |

### 2.5 Validating the Inventory

Before running any commands, it’s good practice to confirm Ansible parses the inventory as expected:

```bash
# List all hosts and the groups they belong to
ansible-inventory -i inventory --list

# List hosts matched by a specific pattern/group
ansible datacenter -i inventory --list-hosts
ansible node* -i inventory --list-hosts
```

## 3. Ad-Hoc Commands — Part B

All commands below were run from `~/ansible-lab` on the controller using the syntax:

```bash
ansible -i inventory <target> -m <module> -a "<module arguments>" [flags]
```

Where `<target>` is a group name (`web`, `db`, `datacenter`, `local`) or a pattern (`node*`, `all`).

### 3.1 Command 1 — Connectivity Check (`ping` module)

```bash
ansible -i inventory datacenter -m ping
```

- **Purpose**: Confirms SSH connectivity and that Python is available on each managed node — the standard first health check before running anything else.
- **Module**: `ping` (not ICMP — it connects over SSH, executes a small Python payload, and returns `pong`).
- **Expected result**: `SUCCESS` with `"ping": "pong"` for both `node1` and `node2`.

### 3.2 Command 2 — Print Date/Time (`command` module)

```bash
ansible -i inventory web -m command -a "date"
```

- **Purpose**: Runs a simple command to confirm remote execution works and to check clock/timezone sync across nodes.
- **Module**: `command` — the safest execution module. It does **not** invoke a shell, so it cannot process pipes (`|`), redirects (`>`), or environment variable expansion (`$VAR`). This avoids shell-injection risk for simple commands.
- **Target**: `web` group only (`node1`).
- **Expected output**: A single line like `Tue Jul 29 14:02:11 UTC 2026`.

### 3.3 Command 3 — Free Disk Space (`shell` module)

```bash
ansible -i inventory datacenter -m shell -a "df -h / | grep -v Filesystem"
```

- **Purpose**: Reports free disk space on the root partition for every host in `datacenter`.
- **Module**: `shell` — required here specifically **because** of the pipe (`|`). The `command` module would pass `df -h / | grep -v Filesystem` literally to `df` as arguments and fail, since it has no shell to interpret the pipe.
- **Expected output** (per host): a single filtered line, e.g. `/dev/root  8.0G  2.1G  5.6G  28% /`.
- **Security note**: `shell` should generally be avoided in favor of `command` unless shell features (pipes, redirects, globbing) are actually needed — it’s more powerful but also more exposed to injection if arguments come from untrusted input.

### 3.4 Command 4 — Gather Facts, Filtered (`setup` module)

```bash
ansible -i inventory datacenter -m setup -a "filter=ansible_memtotal_mb"
```

- **Purpose**: Demonstrates Ansible’s **fact-gathering** system, filtered to return only total memory instead of the full (very large) facts dictionary.
- **Module**: `setup` — runs automatically at the start of every playbook (but not for ad-hoc commands unless called explicitly) and gathers dozens of facts: OS, architecture, network interfaces, memory, CPU, mounted filesystems, etc.
- **Expected output**:
  
  ```json
  node1 | SUCCESS => {
      "ansible_facts": {
          "ansible_memtotal_mb": 957
      },
      "changed": false
  }
  ```
- **Tip**: Run without the `filter=` argument once to see the full fact list — useful for understanding what data is available for use in real playbooks (`ansible datacenter -i inventory -m setup`).

### 3.5 Command 5 — Install a Package, Twice (`package` module — idempotency)

```bash
# 5a. First run — installs htop
ansible -i inventory datacenter -b -m package -a "name=htop state=present"

# 5b. Second run — identical command, demonstrates idempotency
ansible -i inventory datacenter -b -m package -a "name=htop state=present"
```

- **Purpose**: Demonstrates **idempotency** — Ansible’s core principle that running the same task repeatedly produces the same end state without redundant side effects.
- **Module**: `package` — a generic wrapper that delegates to the OS-appropriate backend (`apt` on Ubuntu, `yum`/`dnf` on RHEL, etc.), so the same command works across distributions.
- **`-b` flag**: Short for `--become`; escalates privileges (sudo) since installing packages requires root.
- **Expected results**:
  - **5a**: `"changed": true` — `htop` was not present, so Ansible installed it.
  - **5b**: `"changed": false` — Ansible checked current state, found `htop` already installed, and took no action.
- **Why this matters**: In configuration management, idempotency means playbooks are safe to re-run repeatedly (e.g. on a schedule, or after a failure) without causing errors or duplicate installations.

### 3.6 Command 6 — Manage a Service (`service` module)

```bash
ansible -i inventory datacenter -b -m service -a "name=ssh state=started enabled=yes"
```

- **Purpose**: Ensures the SSH daemon is both **currently running** (`state=started`) and **will start automatically on boot** (`enabled=yes`) — two independent properties.
- **Module**: `service` — a generic wrapper over `systemd`/`init.d`/`upstart` depending on the OS’s init system.
- **`-b` flag**: Required — modifying service state needs root.
- **Expected output**: `"changed": false` in most cases here, since SSH is already running (this is exactly how the ad-hoc connection itself is working) and already enabled on Ubuntu AMIs by default.

### 3.7 Command 7 — Push a File with Content (`copy` module)

```bash
ansible -i inventory web -b -m copy -a 'content="Hello from Ansible" dest=/tmp/hello.txt'
```

- **Purpose**: Creates `/tmp/hello.txt` on `web` group hosts with specific inline content, without needing a source file on the controller.
- **Module**: `copy` — normally copies a local file to the remote host (`src=` / `dest=`), but the `content=` argument lets it write a literal string directly, convenient for small config snippets or ad-hoc file drops.
- **`-b` flag**: Not strictly required for `/tmp` (world-writable), but included for consistency with tasks that do need it.
- **Expected output**: `"changed": true` on first run (file didn’t exist); `"changed": false` on subsequent identical runs (content already matches — `copy` checksums the destination).
- **Verification**: `ssh` to `node1` and run `cat /tmp/hello.txt`.

### 3.8 Command 8 — Create a Directory with Permissions (`file` module)

```bash
ansible -i inventory datacenter -b -m file -a "path=/tmp/ansible_test state=directory mode=0755"
```

- **Purpose**: Creates `/tmp/ansible_test` with `rwxr-xr-x` permissions across every host in `datacenter`.
- **Module**: `file` — manages files, directories, symlinks, and permissions/ownership. `state=directory` specifically tells it to create a directory (and any missing parent directories) rather than a file.
- **Expected output**: `"changed": true` on first run; `"changed": false` if re-run and the directory + mode already match.
- **Verification**: `ls -ld /tmp/ansible_test` should show `drwxr-xr-x`.

### 3.9 Command 9 — Create a System User (`user` module)

```bash
ansible -i inventory db -b -m user -a "name=deployer state=present shell=/bin/false create_home=yes"
```

- **Purpose**: Provisions a `deployer` account on the `db` group (`node2`) intended for automated/service use rather than interactive login.
- **Module**: `user` — manages user accounts, including UID/GID, groups, password, SSH keys, and shell.
- **Key arguments**:
  - `state=present` — create the user if it doesn’t exist (idempotent; leaves it alone if it does).
  - `shell=/bin/false` — assigns a non-interactive shell, so `deployer` cannot get an interactive login session even with valid credentials.
  - `create_home=yes` — creates `/home/deployer`.
- **Expected output**: `"changed": true` on first run, including the new UID assigned; `"changed": false` on re-run.
- **Verification**: `id deployer` and `getent passwd deployer` on `node2`.

### 3.10 Command 10 — Edit a Config File (`lineinfile` module)

```bash
ansible -i inventory datacenter -b -m lineinfile -a "path=/etc/motd line='# Managed by Ansible' state=present create=yes"
```

- **Purpose**: Ensures a specific line exists in `/etc/motd` (the message-of-the-day shown at login), creating the file if it doesn’t already exist.
- **Module**: `lineinfile` — surgically manages a single line within a file without touching the rest of its content; safer than overwriting the whole file with `copy` when only one line needs to be guaranteed present.
- **Key arguments**:
  - `state=present` — ensure the line exists (as opposed to `state=absent`, which would remove it).
  - `create=yes` — create the file if missing (by default `lineinfile` fails on a non-existent file).
- **Expected output**: `"changed": true` on first run; `"changed": false` if the line is already present.
- **Verification**: `cat /etc/motd` on each host.

### 3.11 Command 11 — Ad-Hoc Pattern Matching

```bash
ansible -i inventory node* -m ping
```

- **Purpose**: Demonstrates targeting hosts by **wildcard pattern** against inventory hostnames, rather than by declared group name.
- **How it works**: `node*` matches every inventory hostname beginning with `node` — here, `node1` and `node2` — without needing them to share a group. This is equivalent in effect to `datacenter` in this particular inventory, but patterns work even when no matching group has been defined.
- **Other useful patterns** (for reference, not required above): `all` (every host), `web:db` (union of two groups), `web:&db` (intersection), `web:!db` (in `web` but not `db`), `node1,node2` (explicit comma-separated list).
- **Shell escaping note**: on some shells `*` may need quoting (`"node*"`) to avoid being expanded as a filename glob by the local shell before Ansible sees it.

### 3.12 Command 12 — Parallelism with Forks

```bash
ansible -i inventory datacenter -f 10 -m ping
```

- **Purpose**: Demonstrates controlling **parallel execution** across hosts.
- **How it works**: Ansible’s default fork count is 5, meaning at most 5 hosts are processed simultaneously (irrelevant here with only 2 hosts, but critical at scale). `-f 10` raises that ceiling to 10 concurrent connections/executions.
- **Why it matters**: With large inventories (hundreds of hosts), the default of 5 becomes a bottleneck; increasing forks (bounded by controller CPU/memory and target-side load tolerance) reduces total run time significantly.
- **Expected output**: Identical `SUCCESS`/`pong` results to Command 1 — the fork count changes *execution speed and concurrency*, not the outcome.

## 4. Ansible Concepts Reference

|Concept                                       |Where Demonstrated                       |Explanation                                                                                                                                                                                                                                                       |
|----------------------------------------------|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**Modules**                                   |Commands 1–10                            |The unit of work in Ansible. Each module (`ping`, `command`, `shell`, `setup`, `package`, `service`, `copy`, `file`, `user`, `lineinfile`) is a small, purpose-built program; Ansible ships hundreds of them covering cloud, networking, packages, files, and more|
|**Ad-Hoc vs. Playbook**                       |Entire assignment                        |Ad-hoc = one command, one action, not saved. Playbook = ordered YAML tasks, reusable and version-controlled                                                                                                                                                       |
|**Inventory**                                 |§2.4                                     |The file (or dynamic source) listing managed hosts and the groups/variables that describe how to reach and treat them                                                                                                                                             |
|**Groups**                                    |`web`, `db`, `datacenter`                |Logical sets of hosts so commands can target subsets instead of naming hosts individually                                                                                                                                                                         |
|**Group of Groups**                           |`[datacenter:children]`                  |A group whose members are other groups, letting one command reach several groups at once                                                                                                                                                                          |
|**Patterns**                                  |Command 11 (`node*`)                     |Wildcard/set-based host targeting independent of declared groups                                                                                                                                                                                                  |
|**Connection Types**                          |`ssh` (default) vs. `local` (`localnode`)|How Ansible reaches a host — over SSH to a remote machine, or directly on the controller with no network hop                                                                                                                                                      |
|**Privilege Escalation**                      |`-b` flag, Commands 5–6, 8–10            |“Become” — runs the task as another user (root by default) via `sudo`; required for system-level changes                                                                                                                                                          |
|**Idempotency**                               |Commands 5a/5b                           |Re-running the same task converges to the same state; a no-op second run reports `changed: false`                                                                                                                                                                 |
|**Forks / Parallelism**                       |Command 12 (`-f 10`)                     |Number of hosts processed concurrently; default is 5, tunable via `-f` or `ansible.cfg`                                                                                                                                                                           |
|**Facts**                                     |Command 4 (`setup` module)               |System information (memory, CPU, OS, network) auto-discovered from each managed host, usable as variables in playbooks                                                                                                                                            |
|**`command` vs `shell`**                      |Commands 2 vs 3                          |`command` runs a binary directly (no shell features, safer); `shell` invokes `/bin/sh -c` (supports pipes/redirects, more powerful but riskier)                                                                                                                   |
|**Check Mode** (reference only, not run above)|—                                        |`--check` (dry-run) previews changes without applying them; `--diff` shows the before/after of file changes                                                                                                                                                       |

## 5. Files Submitted

|File          |Description                                                                                                               |
|--------------|--------------------------------------------------------------------------------------------------------------------------|
|`inventory`   |INI-format inventory file defining `web`, `db`, `local` groups, the `datacenter` group-of-groups, and connection variables|
|`commands.txt`|All 12 ad-hoc commands, each preceded by an explanatory comment                                                           |
|`README.md`   |This documentation file                                                                                                   |
|`screenshots/`|Folder containing a terminal screenshot of the output for each of the 12 commands, named `01-ping.png` … `12-forks.png`   |

## 6. Troubleshooting Notes

|Symptom                                                                       |Cause                                                                                                                              |Fix                                                                                                                                                                  |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`UNREACHABLE` / `Permission denied (publickey)`                               |Wrong key path, wrong permissions, or SG blocking port 22                                                                          |Verify the key path in the inventory actually exists on the machine running Ansible; run `chmod 400 <key>`; confirm SG allows TCP 22 from your IP                    |
|`no such identity: ... No such file or directory`                             |`ansible_ssh_private_key_file` points to a path that doesn’t exist on the controller (e.g. a laptop path was left in the inventory)|`find / -iname "*.pem" 2>/dev/null`, then correct the inventory path or move the key to match it                                                                     |
|`sudo: a password is required`                                                |Running a `-b` (become) task against `localnode` from a machine where the current user needs a sudo password                       |Prefer targeting `datacenter` (SSH users configured for passwordless sudo on EC2) instead of `local`, or configure `NOPASSWD` in `/etc/sudoers` for the local user   |
|`/etc/motd: No such file or directory`                                        |`lineinfile` doesn’t create files by default                                                                                       |Add `create=yes` to the module arguments                                                                                                                             |
|`chsh: invalid shell '/usr/sbin/nologin'` or similar `user` module errors     |`/usr/sbin/nologin` may not exist/behave identically across distros                                                                |On Ubuntu, prefer `/bin/false` (always present) for a no-login shell                                                                                                 |
|`Host key verification failed`                                                |New EC2 instance’s SSH fingerprint isn’t in `known_hosts` yet                                                                      |Add `ansible_ssh_common_args='-o StrictHostKeyChecking=no'` under `[all:vars]` (already included in this inventory), or manually `ssh` once to accept the fingerprint|
|Ansible hangs with no output                                                  |Security Group not allowing the controller’s current IP (e.g. IP changed since SG rule was set)                                    |Re-check “My IP” in the SG rule — home/office IPs can change                                                                                                         |
|`WARNING: Platform linux on host X is using the discovered Python interpreter`|Cosmetic warning only, not an error                                                                                                |Safe to ignore for this assignment; can be silenced by explicitly setting `ansible_python_interpreter` in the inventory                                              |
|`apt` module fails with “Could not get lock”                                  |Another process (e.g. unattended-upgrades) holds the `apt` lock on the target                                                      |Wait a few seconds and re-run, or use `package` module which retries more gracefully in some versions                                                                |

## 7. How to Verify

1. SSH into the controller:
   
   ```bash
   ssh -i ansible-assignment.pem ubuntu@3.91.55.23
   ```
1. Move into the working directory:
   
   ```bash
   cd ~/ansible-lab
   ```
1. Confirm connectivity:
   
   ```bash
   ansible -i inventory datacenter -m ping
   ```
1. Run each command from `commands.txt` in order, capturing a screenshot of the output for each into `screenshots/`.
1. Spot-check results directly on the managed nodes where relevant, e.g.:
   
   ```bash
   # On node1
   cat /tmp/hello.txt
   ls -ld /tmp/ansible_test
   cat /etc/motd
   
   # On node2
   id deployer
   ```

## 8. Cleanup

To avoid ongoing AWS charges after grading/submission:

1. Terminate both EC2 instances (`node1`, `node2`) from the AWS Console or CLI.
1. Delete the associated Security Group if it isn’t reused elsewhere.
1. Release any Elastic IPs allocated (these incur charges when unattached).
1. Optionally, remove the `.pem` key pair from the AWS Console if no longer needed for other work.
