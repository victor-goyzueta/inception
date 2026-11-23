# DEV_DOC.md — Developer Documentation

This guide describes how to set up, build, manage and debug the Inception stack
from a clean machine.

## 1. Prerequisites

- A Linux machine or VM (Debian/Ubuntu recommended) with:
  - **Docker Engine**
  - **Docker Compose v1** (the `docker-compose` command — this project uses the
    legacy CLI as specified in the Makefile)
  - **Git**, **make**
- Your user added to the `docker` group (`sudo usermod -aG docker $USER`, then
  re-login).
- Ports `443` free on the host.
- Domain resolution for `jocalder.42.fr` → add `127.0.0.1 jocalder.42.fr` to
  `/etc/hosts` for local runs.

## 2. Project layout

```
├── Makefile
├── README.md / USER_DOC.md / DEV_DOC.md
├── secrets/                  # credentials injected into containers
│   ├── db_password.txt
│   ├── db_root_password.txt
│   ├── wp_admin_password.txt
│   └── wp_user_password.txt
└── srcs/
    ├── .env                  # non-sensitive configuration
    ├── docker-compose.yml    # orchestration of the three services
    └── requirements/
        ├── nginx/Dockerfile + tools/nginx.conf
        ├── wordpress/Dockerfile + tools/setup_wordpress.sh, www.conf
        └── mariadb/Dockerfile + tools/init_db.sh, 50-server.cnf
```

## 3. Configuration from scratch

1. **Clone the repository:**
   ```bash
   git clone https://github.com/jocalder/Inception.git
   cd Inception
   ```

2. **Fill the secret files** (one password per file, no trailing spaces issues —
   a single line is enough):
   ```bash
   echo "secure_db_password"       > secrets/db_password.txt
   echo "secure_root_password"     > secrets/db_root_password.txt
   echo "secure_wp_admin_password" > secrets/wp_admin_password.txt
   echo "secure_wp_user_password"  > secrets/wp_user_password.txt
   ```

3. **Fill `srcs/.env`** with the non-sensitive values, e.g.:
   ```text
   DOMAIN_NAME=jocalder.42.fr
   MYSQL_DATABASE=wordpress
   MYSQL_USER=wordpress
   WP_ADMIN_USER=jocalder         # must NOT contain "admin"
   WP_USER=editor
   WP_USER_EMAIL=editor@jocalder.42.fr
   ```

## 4. Build and launch

Everything goes through the Makefile, which wraps `docker-compose`:

| Command       | Effect                                                             |
|---------------|--------------------------------------------------------------------|
| `make`        | Create data dirs, build images, start the stack                    |
| `make build`  | Build (or rebuild) the images only                                 |
| `make up`     | Start containers in detached mode                                  |
| `make down`   | Stop and remove containers + network (data kept)                   |
| `make logs`   | Follow logs of all services                                        |
| `make status` | `docker-compose ps` — container states                             |
| `make clean`  | `down` + remove all images built by the project                    |
| `make fclean` | `clean` + remove volumes, delete `~/data`, full `docker system prune -af` |
| `make re`     | `fclean` followed by a full `make`                                 |
| `make help`   | List the available rules                                           |

The raw compose equivalents (no Makefile):

```bash
docker-compose -f srcs/docker-compose.yml build
docker-compose -f srcs/docker-compose.yml up -d
docker-compose -f srcs/docker-compose.yml down
```

## 5. Managing containers, images and volumes

```bash
docker ps -a                                                                                 # all containers
docker images                                                                                # built images (nginx, wordpress, mariadb)
docker logs -f <container>                                                                   # live logs of one service
docker exec -it wordpress bash                                                               # open a shell inside a container
docker-compose -f srcs/docker-compose.yml exec mariadb     mariadb -u root -p                # open a MariaDB shell (reads secret)
docker network ls                                                                            # the project network (bridge driver)
docker volume inspect srcs_mariadb                                                           # shows the mountpoint under ~/data
```

Restarting a single service after a config change:

```bash
docker-compose -f srcs/docker-compose.yml up -d --build <service>
```

## 6. Where data lives and how it persists

The stack keeps **all** persistent data on the host, under:

```
~/data/
├── mariadb/     # MariaDB data directory (volume of the mariadb service)
└── wordpress/   # WordPress core files + uploads (volume of the wordpress service)
```

- The directories are created by the Makefile (`$(HOME)/data`) before the first
  build, and mounted into the containers as volumes.
- Containers are ephemeral: `make down` destroys them, but the data in
  `~/data/` survives. `make up` recreates the containers and re-attaches the
  same directories — the site and database come back exactly as they were.
- `make fclean` is the **only** destructive operation: it removes the volumes,
  deletes `~/data/` and prunes all Docker artifacts. After it, the next `make`
  re-initializes WordPress and the database from scratch.

Backup/restore is therefore a simple matter of copying `~/data/`:

```bash
cp -r ~/data ~/data.backup      # backup
rm -rf ~/data && cp -r ~/data.backup ~/data   # restore
```

## 7. Troubleshooting quick reference

| Symptom                                   | Likely cause / fix                                                   |
|-------------------------------------------|----------------------------------------------------------------------|
| Container keeps restarting                | `docker logs <name>`; check secrets exist and `.env` is complete     |
| WordPress shows install page              | Database not reachable; check mariadb logs and that `init_db.sh` ran |
| HTTPS unreachable                         | Port 443 in use, or `/etc/hosts` entry missing                       |
| Certificate warning                       | Expected — certificate is self-signed                                |
| Changes lost after reboot                 | Data must be under `~/data/`; check `docker volume inspect`          |
| MariaDB: root login without password works| Root account not secured; re-check `init_db.sh` on a fresh volume    |
