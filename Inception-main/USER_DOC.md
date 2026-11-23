# USER_DOC.md — User Documentation

This guide explains how to use the Inception stack as an end user or administrator.

## 1. What services are provided

The stack runs three containerized services on an isolated Docker network:

| Service       | Role                                                        | Host exposure        |
|---------------|-------------------------------------------------------------|----------------------|
| **NGINX**     | Entry point: HTTPS/TLS termination and reverse proxy        | Port `443` only      |
| **WordPress** | Web application, served by PHP-FPM                          | None (internal only) |
| **MariaDB**   | Database used by WordPress                                  | None (internal only) |

All external traffic enters through NGINX on port 443. WordPress and MariaDB are
only reachable from inside the Docker network and cannot be accessed directly
from the host or the internet.

## 2. Starting and stopping the project

All commands are run from the root of the repository.

```bash
make            # Build the images and start the full stack (equivalent to build + up)
make up         # Start the services (without rebuilding)
make down       # Stop and remove the containers and network
make restart    # Not provided: use `make down && make up`
make fclean     # Full cleanup: containers, images, volumes and local data (DESTROYS DATA)
```

On the first `make`, the data directories are created automatically at
`~/data/mariadb` and `~/data/wordpress`.

## 3. Accessing the website and the administration panel

The domain must resolve to the machine running the stack. For a local setup,
`/etc/hosts` should contain:

```
127.0.0.1 jocalder.42.fr
```

- **Website:** https://jocalder.42.fr (HTTPS only; a self-signed certificate
  warning is expected — accept it to continue)
- **Login page:** https://jocalder.42.fr/wp-login.php
- **Admin dashboard:** log in with the administrator account defined in
  `secrets/wp_admin_password.txt` (username is set in `srcs/.env`, e.g.
  `WP_ADMIN_USER`). **Do not use a username containing "admin"** — it is
  rejected by the project rules.

> HTTP (port 80) is intentionally disabled: `http://jocalder.42.fr` will not
> connect. All traffic is encrypted with TLS.

## 4. Locating and managing credentials

No password is hard-coded in any file of the stack. Credentials are stored in
plain-text secret files that are injected into the containers at runtime:

| File                              | Used for                              |
|-----------------------------------|---------------------------------------|
| `secrets/db_root_password.txt`    | MariaDB root password                 |
| `secrets/db_password.txt`         | MariaDB user password (WordPress DB)  |
| `secrets/wp_admin_password.txt`   | WordPress administrator password      |
| `secrets/wp_user_password.txt`    | WordPress editor/user password        |

Non-sensitive configuration (domain, database name, usernames) lives in
`srcs/.env`.

**To change a password**, edit the corresponding file under `secrets/` and then
rebuild/restart the affected service:

```bash
make down && make up
```

For the database passwords, note that MariaDB applies credentials at first
initialization only. To force a re-initialization you must wipe the database
volume (this deletes all site data):

```bash
make fclean   # then run `make` again
```

## 5. Checking that the services are running correctly

Show the state of all containers:

```bash
make status
```

All three containers (`nginx`, `wordpress`, `mariadb`) should show
`Up` / `running`. Equivalent raw command:

```bash
docker-compose -f srcs/docker-compose.yml ps
```

Follow the logs of the whole stack or a single service:

```bash
make logs                            # all services, follow mode
docker logs nginx                    # single container
docker logs wordpress
docker logs mariadb
```

Quick functional checks:

```bash
curl -k https://jocalder.42.fr       # website responds over HTTPS
curl http://jocalder.42.fr           # must FAIL (HTTP is not served)
docker volume ls                     # two volumes (database + wordpress)
docker network ls                    # the project's internal network is listed
```

If a container keeps restarting, inspect its logs (`docker logs <name>`) and
verify that the secret files exist and are not empty.
