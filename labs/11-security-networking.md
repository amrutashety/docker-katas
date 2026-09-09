# Security & networking best practices

## Learning Goals

- Minimize the attack surface of your images
- Run containers as a non-root user, and understand why that matters
- Drop unnecessary Linux capabilities and lock down the container filesystem
- Isolate services on the network so only what needs to be reachable, is reachable

## Introduction

By default, Docker containers run as `root`, can use the full set of Linux capabilities, and are free to write anywhere in their own filesystem. None of that is usually necessary for your application to work, and all of it makes a compromised container more dangerous. Hardening a container is mostly about **removing privileges your app doesn't need**.

## Exercise

### Overview

- Minimize the attack surface by choosing a small base image
- Run the application as a non-root user
- Drop Linux capabilities and use a read-only filesystem
- Isolate a backend service on its own internal network

### Step by step instructions

<details>
<summary>Minimize the attack surface</summary>

Every package installed in your image is code that could contain a vulnerability, and every tool available inside a container (a shell, `curl`, a package manager) is a tool an attacker can use if they get in.

- Revisit the images you built in [07-building-an-image](07-building-an-image.md) and [08-multi-stage-builds](08-multi-stage-builds.md). Which base image did you use? `ubuntu`, `alpine`, or `scratch`?
- Compare the size and the number of installed packages between a full distro and a minimal one:

  ```bash
  docker run --rm ubuntu:22.04 sh -c "dpkg -l | wc -l"

  docker run --rm alpine sh -c "apk list --installed | wc -l"
  ```

- See [image-best-practices](image-best-practices.md) for `.dockerignore`, linting and image signing, all of which also reduce what unintentionally ends up in your image.

> :bulb: Multi-stage builds (exercise 08) are one of the best tools you have here: your final image only needs the runtime artifact, not the compilers and build tools used to create it.

</details>

<details>
<summary>Run as a non-root user</summary>

If an attacker manages to break out of your application into the container's OS, running as `root` gives them root inside the container - and if they can also find a container-escape vulnerability, potentially root on the host too.

- Open the `Dockerfile` in [building-an-image](building-an-image/) that you completed in exercise 07.
- Add a non-root user and switch to it before the `CMD`:

  ```dockerfile
  RUN useradd --create-home appuser
  USER appuser
  ```

- Rebuild the image and confirm which user the process runs as:

  ```bash
  docker build -t myfirstapp:nonroot .

  docker run --rm myfirstapp:nonroot id
  ```

  You should see `uid=...(appuser)` instead of `uid=0(root)`.

> :bulb: If your app needs to bind to a port, remember that ports below 1024 require root privileges on Linux - that's one reason many application images default to a high port like `5000` or `8080`.

</details>

<details>
<summary>Drop capabilities and use a read-only filesystem</summary>

Even as a non-root user, a container by default still has a broad set of Linux capabilities and a writable root filesystem. You can restrict both at `docker run` / compose time, without changing the image at all.

- Run a container with all capabilities dropped, and only add back what's strictly required:

  ```bash
  docker run --rm --cap-drop=ALL myfirstapp:nonroot
  ```

  If the app still works with all capabilities dropped, it didn't need any of them.

- Run the container with a read-only root filesystem:

  ```bash
  docker run --rm --read-only myfirstapp:nonroot
  ```

  If your app needs to write somewhere (logs, temp files), give it just that one writable location with a `tmpfs` or volume mount, e.g. `--read-only --tmpfs /tmp`.

- Compare this to the [Running as Root](../trainer/examples/security-run-as-root/README.md) example, which shows what a container running as root can do to a bind-mounted host directory - exactly the scenario `USER` and `--cap-drop` protect you against.

</details>

<details>
<summary>Secure networking: isolate what doesn't need to be public</summary>

The [09-multi-container](09-multi-container.md) exercise already showed that putting containers on a **user-defined network** avoids exposing the database port to the host at all - only containers on that network can reach it, by name.

You can go one step further with Compose's `internal` networks, which additionally block the network from reaching the outside world (no default route out), useful for a database tier that should never need to make outbound connections either.

- In [advanced-compose](advanced-compose/), add a dedicated network for the database:

  ```yaml
  networks:
    backend:
      internal: true

  services:
    mysql-container:
      networks:
        - backend
    wordpress-container:
      networks:
        - backend       # needs to reach mysql-container
        # wordpress also needs default networking to be reachable from your browser,
        # so it should stay on the default network too, or you need a reverse proxy in front of it.
  ```

- Try to reach the internet from inside the mysql container, and confirm it fails:

  ```bash
  docker compose exec mysql-container ping -c 1 8.8.8.8
  ```

- For a deeper look at exactly how bridge networks, `iptables`, and DNS resolution work, revisit [10-docker-architecture](10-docker-architecture.md).

</details>

### Clean up

```bash
docker compose down
docker image rm myfirstapp:nonroot 2>/dev/null
```
