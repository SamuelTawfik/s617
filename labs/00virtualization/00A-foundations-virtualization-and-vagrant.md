# Lab 00 - Foundations: Virtualization, Vagrant, and Running Any S617 Lab

**Cybersecurity Lab Room S617 - Virtualization & Containerization**

> **Why this lab exists.** Every future lab in this course ships as a folder you will bring up *yourself*, on *any* machine, with a single command - no matter what virtualization software or container tool it uses. This lab teaches you the mental model and the exact workflow to do that, to **read and understand** what a lab installs before you run it, and to reason about how virtualized systems are **attacked** and **defended**. Master this and you can install every lab that follows - and you'll have the security instincts to go with it.
>
> **Three things you'll be able to do by the end:**
> 1. Explain what virtualization is, the kinds of it, and when to use each.
> 2. Use Vagrant + VirtualBox (or any provider) + Docker to stand up a lab,
>    read its recipe, and tear it down cleanly.
> 3. Describe how an attacker escapes or pivots between containers, VMs, and the
>    host - and the concrete controls that stop them.

---

## Part 1 - What is virtualization, really?

**Virtualization is running a computer inside a computer.** Instead of one operating system owning the physical hardware, a layer of software hands out *virtual* hardware (CPU, memory, disk, network) to one or more guest systems that each believe they're running on a real machine.

Why we do it - and why it's the backbone of a cyber lab:

- **Isolation** - a guest can be broken, infected, or blown up without harming
  the host or other guests. You *want* a disposable place to detonate malware.
- **Reproducibility** - the same virtual machine can be rebuilt identically,
  anywhere. A lab that works for the instructor works for every student.
- **Snapshots** - freeze a known-good state, make a mess, roll back in seconds.
- **Density** - many isolated environments on one physical box (the whole lab
  room, or a cloud server).

Two broad kinds, and the one idea that separates them:

### Virtual Machines (hardware virtualization)

A **hypervisor** creates virtual hardware and each VM boots its **own full operating system, including its own kernel.** The guest is strongly isolated because the boundary is enforced close to the hardware.

- **Type 1 ("bare-metal") hypervisor** runs *directly on the hardware*:
  VMware ESXi, Microsoft Hyper-V, Proxmox/KVM, Xen. Used for servers and data
  centers - fast and efficient.
- **Type 2 ("hosted") hypervisor** runs *as an app on top of a normal OS*:
  VirtualBox, VMware Workstation. This is what you'll use on a laptop.

> **CPU support matters.** Hardware virtualization uses CPU features - **Intel
> VT-x** (CPU flag `vmx`) or **AMD-V** (flag `svm`). "Nested virtualization" is
> running a hypervisor *inside* a VM, which needs those features passed through.
> It's a nice-to-have for some demos but not required for most of this course.

### Containers (OS-level virtualization)

A container does **not** boot its own kernel. It's an ordinary **process** that the host kernel has given a restricted view and a resource budget. Lighter and faster than a VM, but it **shares the host kernel** - which is the key security difference we'll keep returning to. (Lab 01 takes containers apart by hand.)

### The spectrum - this is the single most useful slide to remember

```
  weaker isolation / faster                      stronger isolation / heavier
  --------------------------------------------------------------------------->>
  plain process -> container -> hardened container -> microVM -> full VM
                  (shares the host kernel)        (own kernel, hardware boundary)
```

You choose a point on this line based on **how much you trust the workload.** Running your own code? A container is fine. Detonating hostile malware? Use a VM - ideally one you can snapshot and revert.

> **How you'll experience this in the course:** your workshop machine is a **VM running in VirtualBox** - a whole graphical desktop in its own window. The **containers** you'll build run *inside* that VM and appear as **shells** with a different prompt. So, concretely: **a VirtualBox window = a VM; a shell with a different prompt inside it = a container.** When a lab needs to show booting a brand-new VM, you'll create one in VirtualBox from a small ISO - no nested virtualization required.

---

## Part 2 - What is Vagrant, and why the whole course uses it

Clicking through a hypervisor's GUI to build a VM by hand is slow, error-prone, and impossible to reproduce. **Vagrant** fixes that: it lets you describe a virtual machine in a small text file - the **`Vagrantfile`** - and build it with one command. This is **Infrastructure as Code (IaC):** the machine is defined by
a recipe you can read, version-control, and share, not by a pile of manual clicks.

The four ideas that make Vagrant click:

| Term | What it is |
|---|---|
| **Provider** | The virtualization software Vagrant drives (VirtualBox here; could be Hyper-V, VMware, libvirt, Docker). |
| **Box** | The base image Vagrant starts from (e.g. a minimal Ubuntu). Small, reusable. |
| **Vagrantfile** | The recipe: which box, how much RAM/CPU, networking, and how to provision it. |
| **Provisioner** | The script that configures the box after boot - a shell script, Ansible, etc. This is where a lab installs its tools. |

**Why this is perfect for a lab you have to distribute:** instead of shipping a giant pre-built machine, a lab ships as a *tiny folder* - a `Vagrantfile` plus a provisioning script. You run one command; Vagrant downloads a small base image and builds the exact environment the lab needs, identically, on your machine.

---

## Part 3 - The skill you'll reuse all semester: bring up any S617 lab

This is the core operational goal. **Every future lab follows the same steps.** 
Learn them once here.

### One-time setup (install the tools)

1. Install **VirtualBox** (the provider) - the free hypervisor from Oracle.
2. Install **Vagrant** - get the current **64-bit** build for your OS.
   - *Windows/Intel Mac:* both run fine. *Apple Silicon Mac:* VirtualBox can't
     run x86 VMs - use the remote lab path your instructor provides instead.
3. Verify:
   ```bash
   vagrant --version
   VBoxManage --version
   ```

### Running a lab (every time)

A lab folder contains at least a `Vagrantfile` and a provisioning script. From inside that folder:

```bash
vagrant up        # build + boot the VM, then run the provisioner. First run
                  # downloads the base box and installs the lab's tools.
vagrant ssh       # log into the running lab machine
# ... do the lab ...
vagrant halt      # shut the VM down (keeps it; boots faster next time)
vagrant reload    # restart, re-reading the Vagrantfile (after config changes)
vagrant destroy   # delete the VM completely; `vagrant up` rebuilds it clean
vagrant status    # what state is the VM in?
```

> 📸 **Screenshot:** _a `vagrant up` run finishing and `vagrant status` showing the VM running._

> **The disposability is a feature.** Broke the lab? `vagrant destroy` then
> `vagrant up` gives you a brand-new, clean environment in minutes. You can't permanently wreck anything - experiment freely.

### READ before you run - the habit that defines this course

Before `vagrant up` on anything, **open the `Vagrantfile` and the provisioning
script and read them.** You should be able to answer:

- What base box is it pulling, and from where?
- What is the provisioner going to install or change?
- Does anything run with elevated privileges, download from the internet, or
  open ports?

This isn't busywork - *"never run what you haven't read"* is a core security discipline, and a provisioning script is just someone else's code running as root on your machine. Throughout this course, lab scripts are written to be **readable**, with comments explaining each step, precisely so you can practice this.

### Where Docker fits

Some labs don't need a whole VM - they run **containers** inside the lab VM (or directly), defined in a `Dockerfile` (how to build an image) and a `docker-compose.yml` (how to run one or more containers together). You'll bring those up with:

```bash
docker compose up -d      # start the lab's containers in the background
docker compose ps         # see what's running
docker compose logs       # read their output
docker compose down       # stop and remove them
```

Same philosophy: the lab ships a readable recipe, you bring it up with one command, and you read it first. (The next lab builds a Docker image and `entrypoint.sh` from scratch.)

---

## Part 4 - The attacker's view: escaping and pivoting

Virtualization is a security control, but it is **not** a magic wall. Attackers specifically target the boundaries. The threat model has three moves:

### Move 1 - Container -> Host (the "escape")

Because a container **shares the host kernel** and is just a restricted process, anything that removes the restrictions puts the attacker on the host as root. The classic paths (all of which are *misconfigurations or kernel bugs*, not magic):

- **A kernel exploit from inside the container** - one shared kernel, so a
  successful exploit owns the host and every other container on it.
- **`--privileged` or excess capabilities** - hands the container most of root's powers; `CAP_SYS_ADMIN` is nearly root and often enough to break out.
- **The Docker socket mounted into a container** - the container can command the host's Docker daemon to launch a new container mounting the host's `/`, i.e. instant host root. (This is also why *being in the `docker` group = root* on the host.)
- **Runtime bugs** - e.g. **runC CVE-2019-5736**, where a malicious container overwrote the host's container-runtime binary and ran as host root on the next `exec`.

### Move 2 - VM -> Host, and VM -> VM (the rarer, harder escape)

A VM has its own kernel and a hardware-enforced boundary, so escaping it means breaking the **hypervisor** itself - much rarer and higher-skill. Historic examples like **"VENOM"** (a virtual floppy-controller bug) or various VirtualBox/VMware guest-to-host CVEs show it's possible but uncommon. Their rarity is exactly *why* a VM is the boundary you choose for truly hostile code.

### Move 3 - Lateral movement (pivoting without an escape)

Often an attacker doesn't need to break the hypervisor at all - they just use the **network**. If containers or VMs share a network with no segmentation, a foothold in one lets them scan and attack the others, or reach the host's services. `--network host` (a container using the host's network namespace with no isolation) and flat, unsegmented lab networks are common enablers.

> **The attacker's one-sentence model:** *find the resource that should have been isolated but wasn't* - a capability left on, a socket mounted, an unpatched shared kernel, a flat network - and use it to reach the host or the neighbors.

---

## Part 5 - The defender's view: protecting virtualized systems

Every attack above maps to a concrete control. This is your hardening checklist and the mindset for every lab you'll build and run.

**Protect the host (it's the crown jewel):**
- **Patch the host kernel and hypervisor** promptly - the shared kernel is the biggest container risk, and hypervisor CVEs are how VM escapes happen.
- **Treat host access as the highest privilege.** Limit who's in the `docker` group; it's root-equivalent.

**Harden containers (least privilege):**
- **Never run `--privileged`** unless you truly must; **drop capabilities** you don't need (`--cap-drop=ALL`, add back only what's required).
- **Never mount the Docker socket** into a container you don't fully trust.
- **Run as a non-root user** inside the container where possible.
- **Apply seccomp / AppArmor / SELinux profiles** to restrict syscalls example sudo apparmor_parser -r /etc/apparmor.d/<profile-name>.
- **Use user namespaces** so container-root maps to an unprivileged host user.

**Choose the right boundary:**
- **Run genuinely untrusted or hostile code in a VM, not a container** - or in a microVM (Firecracker/Kata) when you need both speed and a real boundary.

**Segment the network:**
- **Don't use `--network host`** for untrusted workloads; give each its own network namespace.
- **Segment lab networks** so a compromised guest can't freely reach the host or its neighbors (you'll see this again in the range/pfSense labs).

**Recover fast:**
- **Snapshot known-good states** and revert after exposure. Disposable, reproducible environments (the Vagrant model) *are* a defense: rebuild clean instead of cleaning up.

> **The defender's one-sentence model:** *isolate every resource, grant the least privilege that works, keep the shared kernel patched, segment the network, and be able to rebuild clean.*

---

## Portfolio deliverable

Add a short write-up to your S617 portfolio that proves you can both *operate* and *reason about* virtualized systems:

1. **Operate:** bring up the S617 workshop VM with `vagrant up`, run
   `vagrant ssh`, and paste the output of `vagrant status` and `uname -a` from inside. Then `vagrant halt`. (Show you can drive the workflow.)
2. **Read:** open this lab's `Vagrantfile` and provisioning script and write 3-4 sentences describing, in your own words, what `vagrant up` does to your machine - what it downloads, installs, and changes.
3. **Reason (attack):** in your own words, explain *one* way an attacker escapes a container to the host, and *why* it works.
4. **Reason (defend):** list three concrete controls that would stop or contain that escape.

Commit it to your fork under `submissions/<your-username>/lab00/` and open a pull request following the repo's contribution steps.

---

## Where this leads

- **Lab 01 - Namespaces & cgroups:** build a container *by hand* with no Docker, see the isolation (and its leaks) directly, and connect it to the escape techniques above.
- Later labs ship as Vagrant + Docker folders you now know how to bring up, read, attack, and defend.

### One-line summary to remember

> Virtualization runs a computer inside a computer for **isolation,  reproducibility, and recovery**. **Vagrant** turns a machine into a readable recipe you can build with one command. The boundary between guest and host is a security control an attacker will probe so **isolate everything, grant least privilege, patch the shared kernel, segment the network, and rebuild clean.**
