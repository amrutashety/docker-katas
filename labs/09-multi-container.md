# Multi container setups

## Learning Goals

- Understand why linked containers need a shared Docker network
- Define a multi-container application declaratively with Docker Compose
- Control startup order safely with a healthcheck, instead of just "container started"
- Keep configuration and secrets out of your compose file, using `.env` files and Compose `secrets`

## Introduction

In this scenario, we are going to deploy the CMS system called Wordpress.

> WordPress is a free and open source blogging tool and a content management system (CMS) based on PHP and MySQL, which runs on a web hosting service.

So we need two containers:

- One container that can serve the Wordpress PHP files
- One container that can serve as a MySQL database for Wordpress.

Both containers already exists on the dockerhub: [Wordpress](https://hub.docker.com/_/wordpress/) and [Mysql](https://hub.docker.com/_/mysql/).

## Exercise

### Overview

- Run the two containers separately, and see why they need a shared network to talk to each other
- Define the same setup declaratively with Docker Compose
- Harden the compose setup: healthcheck-based startup order, `.env` config, and secrets instead of plain-text passwords

### Step by step instructions

<details>
<summary>Running containers manually, and the networking problem</summary>

To start a mysql container, issue the following command

```bash
docker run --name mysql-container --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=wordpress -e MYSQL_DATABASE=wordpressdb -d mysql:5.7.36
```

Let's recap what this command does:

- `--name mysql-container` gives the new container a name for better referencing
- `--rm` tells docker to remove the container after it is stopped
- `-p 3306:3306` mounts the host port 3306, to the containers port 3306
- `-e MYSQL_ROOT_PASSWORD=wordpress` / `-e MYSQL_DATABASE=wordpressdb` inject environment variables into the container
- `-d` runs the container detached, in the background

MySQL is now exposing its port 3306 on the host, and everybody can attach to it — **so do not do this in production without proper security settings**.

If we now started a wordpress container pointed at the host's IP, it would work, but only because we exposed MySQL's port to the whole host. That's the problem: to talk to each other, containers shouldn't need to be exposed to the outside world at all.

The fix is a **docker network**: a private channel that lets containers reach each other without touching the host network.

```bash
docker network create if_wordpress

docker run --name mysql-container --rm --network if_wordpress -e MYSQL_ROOT_PASSWORD=wordpress -e MYSQL_DATABASE=wordpressdb -d mysql:5.7.36

docker run --name wordpress-container --rm --network if_wordpress -e WORDPRESS_DB_HOST=mysql-container -e WORDPRESS_DB_PASSWORD=wordpress -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_NAME=wordpressdb -p 8080:80 -d wordpress:5.7.2-apache
```

Notice the `WORDPRESS_DB_HOST` env variable: when a container joins a network, it automatically gets its container name as a DNS name too, so containers can discover each other by name. That DNS name (and the private IP address, usually `172.x.x.x`) is only visible to other containers on the same network — if you don't publish a port with `-p`, the container isn't reachable from outside Docker at all.

- Browse to `http://<host-ip>:8080` — Wordpress is up, and MySQL is no longer exposed to the host.
- Take a look at the network with `docker network inspect if_wordpress`.
- Clean up: `docker stop wordpress-container mysql-container`

</details>

<details>
<summary>The same setup with Docker Compose</summary>

Writing out full `docker run` commands like above gets old fast, especially for multi-container apps. [Docker Compose](https://docs.docker.com/compose/install/) lets you describe your whole application — containers, network, volumes — in one YAML file, and manage it as a single entity.

- `docker-compose.yaml` — the YAML file where the configuration lives
- `docker compose up -d` — creates and starts every service in the file
- `docker compose down` — stops and removes the containers and the default network
- `docker compose ps` / `docker compose logs` — same idea as `docker ps` / `docker logs`, but for the whole stack

> For more information on docker compose yaml files, head over to the [documentation](https://docs.docker.com/compose/compose-file/).

Head over to `cd labs/multi-container` and open `docker-compose.yaml`:

```yaml
services:
  #  wordpress-container:

  mysql-container:
    image: mysql:5.7.36
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PASSWORD: wordpress
      MYSQL_DATABASE: wordpressdb
```

Compare this to the `docker run` command for `mysql-container` above — `ports`, `environment` and `image` map directly onto the `-p`, `-e` and image-name arguments. Compose also creates a network for you automatically (named after the folder), so you don't need `docker network create` anymore.

- Run `docker compose up -d` and confirm both the network and the container were created (`docker network ls`, `docker compose ps`).
- Now add the `wordpress-container` service yourself: uncomment it, and translate the `docker run` command from the previous section into YAML the same way. Also **remove the MySQL port mapping** — it no longer needs to be reachable from the host.
- Run `docker compose up -d` again and browse to `http://<host-ip>:8080`.

> **Hint**: If you are stuck, look at [docker-compose_final.yaml](multi-container/docker-compose_final.yaml) in the same folder.

- Clean up: `docker compose down`

</details>

<details>
<summary>Hardening the setup: healthchecks, .env, and secrets</summary>

Two things about the setup above aren't production-ready:

- `depends_on: [mysql-container]` (which you may have added above) only waits for the **container to start**, not for MySQL to actually be ready to accept connections — Wordpress can fail on its very first request.
- `MYSQL_ROOT_PASSWORD: wordpress` is a plain-text password sitting in a file that normally ends up committed to source control.

Head over to [advanced-compose](advanced-compose/), which has a starter `docker-compose.yaml` for the same Wordpress/MySQL stack plus a `db_password.txt` file.

- **Healthcheck-based startup order.** The `mysql-container` service already has a healthcheck defined:

  ```yaml
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 5s
    timeout: 5s
    retries: 10
  ```

  Add `wordpress-container` yourself, and instead of the short `depends_on: [mysql-container]`, use the long form so Wordpress only starts once MySQL is actually healthy:

  ```yaml
  depends_on:
    mysql-container:
      condition: service_healthy
  ```

  Run `docker compose up -d` then immediately `docker compose ps` — you'll see `mysql-container` pass through a `starting` health state before `wordpress-container` is even created.

- **Secrets instead of plain-text passwords.** Both the `mysql` and `wordpress` images support a `_FILE` suffix convention: `MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db_password` reads the password from a mounted file instead of an environment variable. Declare the secret once at the top level, and reference it in both services:

  ```yaml
  secrets:
    db_password:
      file: ./db_password.txt
  ```

  ```yaml
  secrets:
    - db_password
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_password      # on mysql-container
    WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password     # on wordpress-container
  ```

  Recreate the stack (`docker compose up -d --force-recreate`) and confirm the secret is mounted read-only, but not visible as a plain environment variable:

  ```bash
  docker compose exec mysql-container ls /run/secrets
  docker compose exec mysql-container env | grep MYSQL_ROOT_PASSWORD
  ```

  > :bulb: `db_password.txt` here is only a training placeholder - never commit real secret files to git. Inject them at deploy time instead (a secrets manager, or Kubernetes Secrets).

- **`.env` for non-secret configuration.** Rename [.env.example](advanced-compose/.env.example) to `.env`, and reference the port it defines instead of hard-coding it:

  ```yaml
  ports:
    - "${WORDPRESS_PORT}:80"
  ```

  Compose loads `.env` automatically — change `WORDPRESS_PORT` there and re-run `docker compose up -d`, no edits to the compose file needed.

> **Hint**: If you are stuck, look at [docker-compose_final.yaml](advanced-compose/docker-compose_final.yaml) in the same folder.

</details>

### Clean up

```bash
docker compose down
```

