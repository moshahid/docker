**DOCKER LAB PRACTICE MANUAL**

**Basic to Expert**

Hands-on exercises for Ubuntu in VirtualBox  
Docker Engine 29 • Docker Compose v2 • Linux amd64

Lab host: Test-VirtualBox \| SSH: 192.168.56.10 \| User: moshahid

# How to use this manual

| **LAB RULE** Run commands one block at a time. Read the output before continuing. Commands that open an interactive shell, such as docker exec -it, pause the host shell until you type exit. |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

| **Stage**  | **Modules** | **Target capability**                                                  |
|------------|-------------|------------------------------------------------------------------------|
| Foundation | 1–3         | Run, inspect, stop, restart, and remove containers                     |
| Builder    | 4–7         | Use storage, networks, Dockerfiles, BuildKit, and Compose              |
| Operator   | 8–10        | Troubleshoot, monitor, secure, and maintain Docker                     |
| Expert     | 11–13       | Registries, remote contexts, advanced builds, orchestration, capstones |

## Your lab baseline

| **Component**  | **Value**                                   |
|----------------|---------------------------------------------|
| Guest OS       | Ubuntu 25.10 (Questing), x86_64             |
| Virtualization | VirtualBox with Guest Additions             |
| Adapter 1      | NAT for internet access                     |
| Adapter 2      | Host-only, static 192.168.56.10/24          |
| Windows host   | 192.168.56.1                                |
| Docker         | Ubuntu docker.io package, Engine 29.1.3     |
| Compose        | Docker Compose v2.40.3                      |
| TLS inspection | Fortinet CA installed in Ubuntu trust store |

| **SNAPSHOT** Before Module 1, take a VirtualBox snapshot named docker-baseline. This gives you a clean recovery point. |
|------------------------------------------------------------------------------------------------------------------------|

# Module 1 — Foundations: images and containers

## Lab 1.1 — Verify the Docker platform

| **Level** | **Time** | **Primary outcome**                                                    |
|-----------|----------|------------------------------------------------------------------------|
| Basic     | 15 min   | Prove the client, daemon, runtime, networking, and registry path work. |

**Concepts:** daemon, client, Unix socket, containerd, runc, registry

**Step 1 —** Capture versions and service health.

> docker version  
> docker compose version  
> systemctl is-active docker  
> systemctl is-enabled docker

**Step 2 —** Inspect the host-facing Docker summary.

> docker info

**Step 3 —** Run the disposable test image.

> docker run --rm hello-world

**Step 4 —** Observe that --rm leaves no stopped container.

> docker ps  
> docker ps -a

| **PASS CHECK** docker version shows both Client and Server; hello-world completes; docker ps -a contains no hello-world container. |
|------------------------------------------------------------------------------------------------------------------------------------|

| **STRETCH** Identify which output lines correspond to the client, daemon, containerd, and runc. |
|-------------------------------------------------------------------------------------------------|

## Lab 1.2 — Container lifecycle

| **Level** | **Time** | **Primary outcome**                               |
|-----------|----------|---------------------------------------------------|
| Basic     | 25 min   | Create and manage a long-running Nginx container. |

**Concepts:** create, start, stop, restart, status, port publishing

**Step 1 —** Create the container and publish host port 8080 to container port 80.

> docker run -d --name nginx-lab --restart unless-stopped -p 8080:80 nginx:alpine

**Step 2 —** Inspect running containers and test locally.

> docker ps  
> curl http://localhost:8080

**Step 3 —** Test from Windows.

**Step 4 —** Stop it, observe the Exited state, then start it.

> docker stop nginx-lab  
> docker ps -a  
> docker start nginx-lab  
> docker ps

**Step 5 —** Restart and inspect its restart count.

> docker restart nginx-lab  
> docker inspect -f '{{.RestartCount}}' nginx-lab

| **PASS CHECK** Windows can open http://192.168.56.10:8080 and docker ps reports nginx-lab as Up. |
|--------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f nginx-lab

## Lab 1.3 — Create versus run

| **Level** | **Time** | **Primary outcome**                               |
|-----------|----------|---------------------------------------------------|
| Basic     | 20 min   | Separate container creation from process startup. |

**Concepts:** docker create, docker start, attach, foreground versus detached

**Step 1 —** Create a container without starting it.

> docker create --name sleeper alpine:latest sleep 300

**Step 2 —** Confirm its Created state.

> docker ps -a --filter name=sleeper

**Step 3 —** Start and inspect it.

> docker start sleeper  
> docker top sleeper  
> docker inspect -f '{{.State.Status}} PID={{.State.Pid}}' sleeper

**Step 4 —** Stop with a short grace period.

> docker stop -t 2 sleeper

| **PASS CHECK** The state transitions Created → Running → Exited. |
|------------------------------------------------------------------|

**Cleanup**

> docker rm sleeper

# Module 2 — Inspection, processes, logs, and commands

## Lab 2.1 — Logs and interactive access

| **Level** | **Time** | **Primary outcome**                                              |
|-----------|----------|------------------------------------------------------------------|
| Basic     | 25 min   | Inspect application output and enter a running container safely. |

**Concepts:** stdout/stderr, logs, exec, PID 1, exit

**Step 1 —** Start Nginx.

> docker run -d --name inspect-web -p 8081:80 nginx:alpine

**Step 2 —** Generate traffic and read timestamps.

> curl http://localhost:8081 \>/dev/null  
> docker logs --timestamps --tail 20 inspect-web

**Step 3 —** Inspect processes from the host.

> docker top inspect-web

**Step 4 —** Enter the container. Run only the commands shown, then type exit.

> docker exec -it inspect-web sh

**Step 5 —** Inside the container, inspect the OS and process list.

> cat /etc/alpine-release  
> ps  
> ls -la /usr/share/nginx/html  
> exit

**Step 6 —** Run a noninteractive command.

> docker exec inspect-web nginx -t

| **PASS CHECK** nginx -t reports syntax is ok and successful; the host prompt returns after exit. |
|--------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f inspect-web

## Lab 2.2 — Environment variables and command overrides

| **Level** | **Time** | **Primary outcome**                    |
|-----------|----------|----------------------------------------|
| Basic     | 20 min   | Control container behavior at runtime. |

**Concepts:** environment, ENTRYPOINT, CMD, command override

**Step 1 —** Pass environment variables.

> docker run --rm -e LAB_NAME=docker101 -e OWNER=moshahid alpine env \| sort

**Step 2 —** Override the image default command.

> docker run --rm alpine sh -c 'echo Host=\$(hostname); echo Kernel=\$(uname -r)'

**Step 3 —** Inspect image defaults.

> docker image inspect alpine --format 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}}'

| **PASS CHECK** LAB_NAME and OWNER appear; the command override prints a container hostname and kernel. |
|--------------------------------------------------------------------------------------------------------|

| **STRETCH** Explain why the container kernel matches the host kernel family even though its user space is Alpine. |
|-------------------------------------------------------------------------------------------------------------------|

## Lab 2.3 — Inspect and format output

| **Level** | **Time** | **Primary outcome**                                           |
|-----------|----------|---------------------------------------------------------------|
| Basic     | 25 min   | Extract useful metadata without reading large JSON documents. |

**Concepts:** inspect JSON, Go templates, filters, formatting

**Step 1 —** Create a labeled container.

> docker run -d --name meta-web --label purpose=training --label owner=moshahid -p 8082:80 nginx:alpine

**Step 2 —** Extract its IP, image, restart policy, and labels.

> docker inspect -f 'IP={{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' meta-web  
> docker inspect -f 'Image={{.Config.Image}} Restart={{.HostConfig.RestartPolicy.Name}}' meta-web  
> docker inspect -f '{{json .Config.Labels}}' meta-web

**Step 3 —** Use filters.

> docker ps --filter label=purpose=training --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'

| **PASS CHECK** The filtered table contains only meta-web and exposes host port 8082. |
|--------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f meta-web

# Module 3 — Images, tags, layers, and cleanup

## Lab 3.1 — Images and immutable layers

| **Level** | **Time** | **Primary outcome**                                                 |
|-----------|----------|---------------------------------------------------------------------|
| Basic     | 30 min   | Understand image identity, tags, digests, history, and layer reuse. |

**Concepts:** image, tag, digest, layer, content addressability

**Step 1 —** Pull and inspect two tags.

> docker pull alpine:3.21  
> docker pull alpine:latest  
> docker images alpine

**Step 2 —** Inspect history and RepoDigests.

> docker history alpine:3.21  
> docker image inspect alpine:3.21 --format '{{json .RepoDigests}}'

**Step 3 —** Create a local alias.

> docker tag alpine:3.21 local/alpine:lab  
> docker images local/alpine

**Step 4 —** Remove only the alias.

> docker rmi local/alpine:lab

| **PASS CHECK** Removing the alias does not necessarily delete shared layers used by alpine:3.21. |
|--------------------------------------------------------------------------------------------------|

| **STRETCH** Compare IMAGE IDs for alpine:3.21 and alpine:latest. Record whether they point to the same content. |
|-----------------------------------------------------------------------------------------------------------------|

## Lab 3.2 — Container writable layer

| **Level**    | **Time** | **Primary outcome**                               |
|--------------|----------|---------------------------------------------------|
| Intermediate | 25 min   | Observe data that exists only inside a container. |

**Concepts:** copy-on-write, writable layer, docker diff, commit anti-pattern

| **CAUTION** Do not use docker commit as your normal build workflow. Reproducible images come from Dockerfiles. |
|----------------------------------------------------------------------------------------------------------------|

**Step 1 —** Create an Alpine container and write a file.

> docker run -d --name layer-lab alpine sleep 600  
> docker exec layer-lab sh -c 'echo temporary-data \> /lab.txt'

**Step 2 —** Show filesystem changes.

> docker diff layer-lab  
> docker exec layer-lab cat /lab.txt

**Step 3 —** Delete and recreate the container.

> docker rm -f layer-lab  
> docker run -d --name layer-lab alpine sleep 600

**Step 4 —** Prove the file is gone.

> docker exec layer-lab sh -c 'test ! -e /lab.txt && echo file-is-gone'

| **PASS CHECK** The recreated container prints file-is-gone. |
|-------------------------------------------------------------|

**Cleanup**

> docker rm -f layer-lab

## Lab 3.3 — Safe cleanup

| **Level**    | **Time** | **Primary outcome**                                     |
|--------------|----------|---------------------------------------------------------|
| Intermediate | 20 min   | Measure and reclaim unused Docker objects deliberately. |

**Concepts:** disk usage, dangling objects, prune safety

| **CAUTION** Do not use docker system prune -a --volumes casually. It can remove cached images and unused volumes containing lab data. |
|---------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Review usage before changing anything.

> docker system df -v

**Step 2 —** List stopped containers and dangling images.

> docker ps -a --filter status=exited  
> docker images --filter dangling=true

**Step 3 —** Remove selected test objects by name or ID.

> docker container prune

**Step 4 —** Review usage again.

> docker system df

| **PASS CHECK** You can explain exactly which objects were removed. |
|--------------------------------------------------------------------|

# Module 4 — Persistent storage

## Lab 4.1 — Named volumes

| **Level**    | **Time** | **Primary outcome**                                  |
|--------------|----------|------------------------------------------------------|
| Intermediate | 35 min   | Persist data independently of a container lifecycle. |

**Concepts:** volume, mount point, persistence, ownership

**Step 1 —** Create and inspect a named volume.

> docker volume create lab-data  
> docker volume inspect lab-data

**Step 2 —** Write data through one container.

> docker run --rm -v lab-data:/data alpine sh -c 'date \> /data/created.txt; echo persistent \>\> /data/created.txt'

**Step 3 —** Read it through another container.

> docker run --rm -v lab-data:/data:ro alpine cat /data/created.txt

**Step 4 —** Show the volume still exists.

> docker volume ls  
> docker system df -v

| **PASS CHECK** A second disposable container reads the file written by the first. |
|-----------------------------------------------------------------------------------|

**Cleanup**

> docker volume rm lab-data

## Lab 4.2 — Bind mounts and live web content

| **Level**    | **Time** | **Primary outcome**                                     |
|--------------|----------|---------------------------------------------------------|
| Intermediate | 35 min   | Serve host-managed content without rebuilding an image. |

**Concepts:** bind mount, read-only mount, host path, live update

**Step 1 —** Create a project directory and page.

> mkdir -p ~/docker-labs/bind-web/html  
> printf '\<h1\>Docker bind-mount lab\</h1\>\n\<p\>Served from Ubuntu.\</p\>\n' \> ~/docker-labs/bind-web/html/index.html

**Step 2 —** Run Nginx with a read-only bind mount.

> docker run -d --name bind-web -p 8083:80 -v ~/docker-labs/bind-web/html:/usr/share/nginx/html:ro nginx:alpine

**Step 3 —** Test, edit the host file, and test again.

> curl http://localhost:8083  
> printf '\<h1\>Updated without rebuild\</h1\>\n' \> ~/docker-labs/bind-web/html/index.html  
> curl http://localhost:8083

**Step 4 —** Inspect the mount.

> docker inspect -f '{{json .Mounts}}' bind-web

| **PASS CHECK** The second curl response changes without recreating the container. |
|-----------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f bind-web

## Lab 4.3 — Backup and restore a volume

| **Level** | **Time** | **Primary outcome**                                        |
|-----------|----------|------------------------------------------------------------|
| Advanced  | 40 min   | Create a portable backup and restore it into a new volume. |

**Concepts:** tar backup, restore, disposable utility container

**Step 1 —** Create source data.

> docker volume create source-data  
> docker run --rm -v source-data:/data alpine sh -c 'mkdir -p /data/app; echo critical \> /data/app/data.txt'  
> mkdir -p ~/docker-labs/backups

**Step 2 —** Back up the volume.

> docker run --rm -v source-data:/source:ro -v ~/docker-labs/backups:/backup alpine tar czf /backup/source-data.tgz -C /source .

**Step 3 —** Restore into a different volume.

> docker volume create restored-data  
> docker run --rm -v restored-data:/target -v ~/docker-labs/backups:/backup alpine tar xzf /backup/source-data.tgz -C /target

**Step 4 —** Validate the restored data.

> docker run --rm -v restored-data:/data:ro alpine cat /data/app/data.txt

| **PASS CHECK** The restored volume returns critical. |
|------------------------------------------------------|

**Cleanup**

> docker volume rm source-data restored-data

# Module 5 — Container networking

## Lab 5.1 — Default bridge versus user-defined bridge

| **Level**    | **Time** | **Primary outcome**                                      |
|--------------|----------|----------------------------------------------------------|
| Intermediate | 40 min   | Use Docker DNS for container-to-container communication. |

**Concepts:** bridge, embedded DNS, service discovery, isolation

**Step 1 —** Create a user-defined network.

> docker network create app-net  
> docker network inspect app-net

**Step 2 —** Start a web server attached to it.

> docker run -d --name backend --network app-net nginx:alpine

**Step 3 —** Resolve and call it by container name.

> docker run --rm --network app-net alpine sh -c 'wget -qO- http://backend \| head'

**Step 4 —** Show isolation from an unrelated network.

> docker network create isolated-net  
> docker run --rm --network isolated-net alpine sh -c 'wget -T 2 -qO- http://backend \|\| echo backend-not-reachable'

| **PASS CHECK** Name resolution works on app-net and fails from isolated-net. |
|------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f backend  
> docker network rm app-net isolated-net

## Lab 5.2 — Port publishing and address binding

| **Level**    | **Time** | **Primary outcome**                                    |
|--------------|----------|--------------------------------------------------------|
| Intermediate | 30 min   | Control which host interfaces expose a container port. |

**Concepts:** publish, host IP, container port, 0.0.0.0, loopback

| **CAUTION** Published Docker ports may bypass expected UFW behavior. Use the DOCKER-USER chain for policy enforcement in production designs. |
|----------------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Publish to all host interfaces.

> docker run -d --name public-web -p 8084:80 nginx:alpine  
> docker port public-web  
> sudo ss -lntp \| grep 8084

**Step 2 —** Replace it with a loopback-only binding.

> docker rm -f public-web  
> docker run -d --name local-web -p 127.0.0.1:8084:80 nginx:alpine

**Step 3 —** Compare local and Windows reachability.

> curl http://127.0.0.1:8084

| **PASS CHECK** Ubuntu curl succeeds; Windows access to http://192.168.56.10:8084 fails while bound to 127.0.0.1. |
|------------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f local-web

## Lab 5.3 — Custom subnet and static container address

| **Level** | **Time** | **Primary outcome**                                                         |
|-----------|----------|-----------------------------------------------------------------------------|
| Advanced  | 30 min   | Create deterministic lab IPAM while preserving DNS-based service discovery. |

**Concepts:** IPAM, subnet, gateway, static address

**Step 1 —** Create a custom bridge.

> docker network create --driver bridge --subnet 172.30.0.0/24 --gateway 172.30.0.1 custom-net

**Step 2 —** Start a container at a fixed address.

> docker run -d --name fixed-web --network custom-net --ip 172.30.0.10 nginx:alpine

**Step 3 —** Inspect and test it from the same network.

> docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' fixed-web  
> docker run --rm --network custom-net alpine wget -qO- http://fixed-web

| **PASS CHECK** Inspection reports 172.30.0.10 and DNS name fixed-web works. |
|-----------------------------------------------------------------------------|

| **STRETCH** Explain why application configuration should normally use fixed-web rather than 172.30.0.10. |
|----------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f fixed-web  
> docker network rm custom-net

# Module 6 — Dockerfiles and image builds

## Lab 6.1 — Build a custom static website image

| **Level**    | **Time** | **Primary outcome**                      |
|--------------|----------|------------------------------------------|
| Intermediate | 45 min   | Create a reproducible image from source. |

**Concepts:** Dockerfile, build context, FROM, COPY, tag

**Step 1 —** Create the project.

> mkdir -p ~/docker-labs/custom-web  
> cd ~/docker-labs/custom-web  
> printf '\<h1\>My first Docker image\</h1\>\n' \> index.html

**Step 2 —** Create Dockerfile with your editor.

> nano Dockerfile

**Step 3 —** Enter the Dockerfile content.

> FROM nginx:alpine  
> COPY index.html /usr/share/nginx/html/index.html

**Step 4 —** Build and inspect the image.

> docker build -t custom-web:v1 .  
> docker images custom-web  
> docker history custom-web:v1

**Step 5 —** Run and verify.

> docker run -d --name custom-web -p 8085:80 custom-web:v1  
> curl http://localhost:8085

| **PASS CHECK** The response says My first Docker image and docker history shows the COPY layer. |
|-------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f custom-web

## Lab 6.2 — Build cache and .dockerignore

| **Level**    | **Time** | **Primary outcome**                     |
|--------------|----------|-----------------------------------------|
| Intermediate | 40 min   | Make builds smaller, safer, and faster. |

**Concepts:** build context, cache, invalidation, .dockerignore

**Step 1 —** Add files that must not enter the build context.

> cd ~/docker-labs/custom-web  
> mkdir -p logs  
> printf 'secret-demo-only\n' \> local-secret.txt  
> printf '\*.log\nlocal-secret.txt\n.git\n' \> .dockerignore  
> printf 'noise\n' \> logs/debug.log

**Step 2 —** Build twice and observe CACHED steps.

> docker build -t custom-web:v2 .  
> docker build -t custom-web:v2 .

**Step 3 —** Change index.html and rebuild.

> printf '\<h1\>Cache invalidation test\</h1\>\n' \> index.html  
> docker build -t custom-web:v3 .

**Step 4 —** Prove ignored content is absent.

> docker run --rm custom-web:v3 sh -c 'test ! -e /local-secret.txt && echo ignored-successfully'

| **PASS CHECK** The second unchanged build uses cache; local-secret.txt is not in the image. |
|---------------------------------------------------------------------------------------------|

| **STRETCH** Use docker build --no-cache once and compare elapsed time. |
|------------------------------------------------------------------------|

## Lab 6.3 — Run as a non-root user

| **Level** | **Time** | **Primary outcome**                        |
|-----------|----------|--------------------------------------------|
| Advanced  | 50 min   | Build a least-privilege application image. |

**Concepts:** USER, ownership, unprivileged port, runtime identity

**Step 1 —** Create a simple Python project.

> mkdir -p ~/docker-labs/nonroot-app  
> cd ~/docker-labs/nonroot-app  
> printf 'from http.server import HTTPServer, SimpleHTTPRequestHandler\nHTTPServer(("0.0.0.0", 8000), SimpleHTTPRequestHandler).serve_forever()\n' \> app.py  
> nano Dockerfile

**Step 2 —** Enter the Dockerfile.

> FROM python:3.13-alpine  
> WORKDIR /app  
> COPY --chown=10001:10001 app.py .  
> USER 10001:10001  
> EXPOSE 8000  
> CMD \["python", "app.py"\]

**Step 3 —** Build and run.

> docker build -t nonroot-app:v1 .  
> docker run -d --name nonroot-app -p 8086:8000 nonroot-app:v1

**Step 4 —** Verify runtime identity.

> docker exec nonroot-app id  
> docker inspect -f 'Configured user={{.Config.User}}' nonroot-app  
> curl http://localhost:8086

| **PASS CHECK** id reports uid=10001 and the web request succeeds. |
|-------------------------------------------------------------------|

**Cleanup**

> docker rm -f nonroot-app

## Lab 6.4 — Multi-stage build

| **Level** | **Time** | **Primary outcome**                                 |
|-----------|----------|-----------------------------------------------------|
| Advanced  | 60 min   | Separate build dependencies from the runtime image. |

**Concepts:** multi-stage build, artifact copy, attack surface, image size

**Step 1 —** Create a Go application.

> mkdir -p ~/docker-labs/multistage  
> cd ~/docker-labs/multistage  
> printf 'package main\nimport ("fmt"; "net/http")\nfunc main(){http.HandleFunc("/",func(w http.ResponseWriter,r \*http.Request){fmt.Fprintln(w,"multi-stage works")});http.ListenAndServe(":8080",nil)}\n' \> main.go  
> nano Dockerfile

**Step 2 —** Enter the multi-stage Dockerfile.

> FROM golang:1.25-alpine AS build  
> WORKDIR /src  
> COPY main.go .  
> RUN CGO_ENABLED=0 go build -o /out/app main.go  
>   
> FROM scratch  
> COPY --from=build /out/app /app  
> USER 65532:65532  
> EXPOSE 8080  
> ENTRYPOINT \["/app"\]

**Step 3 —** Build and compare sizes.

> docker build -t multistage-app:v1 .  
> docker images multistage-app golang

**Step 4 —** Run and test.

> docker run -d --name multistage-app -p 8087:8080 multistage-app:v1  
> curl http://localhost:8087

| **PASS CHECK** The app responds multi-stage works and its final image is far smaller than golang:1.25-alpine. |
|---------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f multistage-app

# Module 7 — Docker Compose

## Lab 7.1 — Convert Nginx to Compose

| **Level**    | **Time** | **Primary outcome**                   |
|--------------|----------|---------------------------------------|
| Intermediate | 45 min   | Declare and manage a service as code. |

**Concepts:** compose.yaml, service, ports, bind mount, project lifecycle

**Step 1 —** Create the project.

> mkdir -p ~/docker-labs/compose-web/html  
> cd ~/docker-labs/compose-web  
> printf '\<h1\>Managed by Compose\</h1\>\n' \> html/index.html  
> nano compose.yaml

**Step 2 —** Enter the Compose file.

> services:  
> web:  
> image: nginx:alpine  
> ports:  
> - "8088:80"  
> volumes:  
> - ./html:/usr/share/nginx/html:ro  
> restart: unless-stopped

**Step 3 —** Validate and start.

> docker compose config  
> docker compose up -d  
> docker compose ps  
> curl http://localhost:8088

**Step 4 —** Inspect logs and stop without deleting.

> docker compose logs --tail 20 web  
> docker compose stop  
> docker compose ps -a

**Step 5 —** Start again.

> docker compose start

| **PASS CHECK** docker compose config succeeds and Windows can open http://192.168.56.10:8088. |
|-----------------------------------------------------------------------------------------------|

**Cleanup**

> cd ~/docker-labs/compose-web && docker compose down

## Lab 7.2 — Multi-container application with database

| **Level** | **Time** | **Primary outcome**                                         |
|-----------|----------|-------------------------------------------------------------|
| Advanced  | 75 min   | Operate an application and PostgreSQL on a private network. |

**Concepts:** service DNS, environment, named volume, dependency, database persistence

| **CAUTION** The password is intentionally simple for an isolated training VM only. Never commit real secrets to Compose files. |
|--------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Create the project and Compose file.

> mkdir -p ~/docker-labs/compose-db  
> cd ~/docker-labs/compose-db  
> nano compose.yaml

**Step 2 —** Enter the Compose file.

> services:  
> db:  
> image: postgres:17-alpine  
> environment:  
> POSTGRES_DB: labdb  
> POSTGRES_USER: labuser  
> POSTGRES_PASSWORD: labpass-change-me  
> volumes:  
> - pgdata:/var/lib/postgresql/data  
> healthcheck:  
> test: \["CMD-SHELL", "pg_isready -U labuser -d labdb"\]  
> interval: 5s  
> timeout: 3s  
> retries: 10  
> adminer:  
> image: adminer:latest  
> ports:  
> - "8089:8080"  
> depends_on:  
> db:  
> condition: service_healthy  
> volumes:  
> pgdata:

**Step 3 —** Validate and launch.

> docker compose config  
> docker compose up -d  
> docker compose ps

**Step 4 —** Create and query a table.

> docker compose exec db psql -U labuser -d labdb -c 'CREATE TABLE IF NOT EXISTS notes(id serial primary key, body text);'  
> docker compose exec db psql -U labuser -d labdb -c "INSERT INTO notes(body) VALUES ('persistent row');"  
> docker compose exec db psql -U labuser -d labdb -c 'SELECT \* FROM notes;'

**Step 5 —** Recreate services while preserving the volume.

> docker compose down  
> docker compose up -d  
> docker compose exec db psql -U labuser -d labdb -c 'SELECT \* FROM notes;'

| **PASS CHECK** The persistent row remains after docker compose down/up. |
|-------------------------------------------------------------------------|

**Cleanup**

> cd ~/docker-labs/compose-db && docker compose down -v

## Lab 7.3 — Compose overrides, profiles, and scaling

| **Level** | **Time** | **Primary outcome**                                   |
|-----------|----------|-------------------------------------------------------|
| Advanced  | 60 min   | Adapt one Compose model for multiple operating modes. |

**Concepts:** profiles, override files, replicas, project names

**Step 1 —** Create a profile-based stack.

> mkdir -p ~/docker-labs/compose-profiles  
> cd ~/docker-labs/compose-profiles  
> nano compose.yaml

**Step 2 —** Enter the Compose file.

> services:  
> web:  
> image: nginx:alpine  
> debug:  
> image: nicolaka/netshoot  
> command: sleep infinity  
> profiles: \["debug"\]

**Step 3 —** Start only the default service.

> docker compose up -d  
> docker compose ps

**Step 4 —** Start the debug profile.

> docker compose --profile debug up -d  
> docker compose ps

**Step 5 —** Scale a service without publishing a fixed host port.

> docker compose up -d --scale web=3  
> docker compose ps

| **PASS CHECK** Three web replicas and one debug container are running. |
|------------------------------------------------------------------------|

**Cleanup**

> cd ~/docker-labs/compose-profiles && docker compose --profile debug down

# Module 8 — Operations and troubleshooting

## Lab 8.1 — Events, stats, and process troubleshooting

| **Level** | **Time** | **Primary outcome**                                |
|-----------|----------|----------------------------------------------------|
| Advanced  | 45 min   | Correlate lifecycle events with resource behavior. |

**Concepts:** events, stats, top, inspect, OOM state

**Step 1 —** Open a second SSH session and watch events.

> docker events --filter type=container

**Step 2 —** In the first session, generate lifecycle events.

> docker run -d --name ops-lab alpine sleep 300  
> docker pause ops-lab  
> docker unpause ops-lab  
> docker restart ops-lab

**Step 3 —** Review live and one-shot resource data.

> docker stats ops-lab  
> docker stats --no-stream ops-lab  
> docker top ops-lab

**Step 4 —** Inspect exit and OOM fields.

> docker inspect -f 'Status={{.State.Status}} Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Error={{.State.Error}}' ops-lab

| **PASS CHECK** The second session records create/start/pause/unpause/restart events. |
|--------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f ops-lab

## Lab 8.2 — Health checks

| **Level** | **Time** | **Primary outcome**                                       |
|-----------|----------|-----------------------------------------------------------|
| Advanced  | 45 min   | Distinguish a running process from a healthy application. |

**Concepts:** HEALTHCHECK, status transitions, health logs

**Step 1 —** Run Nginx with a runtime health check.

> docker run -d --name health-web -p 8090:80 --health-cmd='wget -qO- http://127.0.0.1/ \>/dev/null \|\| exit 1' --health-interval=5s --health-timeout=2s --health-retries=3 nginx:alpine

**Step 2 —** Watch health transition.

> watch -n 2 "docker ps --filter name=health-web"

**Step 3 —** Exit watch with Ctrl+C and inspect health logs.

> docker inspect -f '{{json .State.Health}}' health-web

| **PASS CHECK** The status becomes Up ... (healthy) and health output shows exit code 0. |
|-----------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f health-web

## Lab 8.3 — Resource limits and OOM behavior

| **Level** | **Time** | **Primary outcome**                                |
|-----------|----------|----------------------------------------------------|
| Advanced  | 45 min   | Constrain CPU and memory and diagnose enforcement. |

**Concepts:** memory limit, CPU quota, OOMKilled, resource accounting

| **CAUTION** This lab intentionally causes one disposable container to fail. It should not exhaust the VM because Docker limits it to 32 MiB. |
|----------------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Run a memory-limited container.

> docker run -d --name limited --memory 64m --cpus 0.50 alpine sleep 600

**Step 2 —** Inspect limits.

> docker inspect -f 'Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}' limited  
> docker stats --no-stream limited

**Step 3 —** Trigger an intentional memory failure using a separate container.

> docker run --name oom-test --memory 32m alpine sh -c 'x=\$(head -c 100M /dev/zero \| base64); sleep 5' \|\| true

**Step 4 —** Diagnose termination.

> docker inspect -f 'Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}}' oom-test

| **PASS CHECK** limited remains running with configured limits; oom-test reports OOMKilled=true or an OOM-related nonzero exit. |
|--------------------------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f limited oom-test

## Lab 8.4 — A repeatable troubleshooting workflow

| **Level** | **Time** | **Primary outcome**                         |
|-----------|----------|---------------------------------------------|
| Advanced  | 50 min   | Diagnose a container that starts and exits. |

**Concepts:** exit code, logs, configuration, mounts, ports, DNS

**Step 1 —** Create a deliberately broken container.

> docker run --name broken-web nginx:alpine nginx -g 'bad_directive;' \|\| true

**Step 2 —** Apply the standard evidence sequence.

> docker ps -a --filter name=broken-web  
> docker inspect -f 'Exit={{.State.ExitCode}} Error={{.State.Error}}' broken-web  
> docker logs broken-web  
> docker inspect broken-web

**Step 3 —** Remove and correct it.

> docker rm broken-web  
> docker run -d --name fixed-web -p 8091:80 nginx:alpine  
> curl http://localhost:8091

| **PASS CHECK** You identify the invalid Nginx directive from logs, then the corrected container returns HTTP 200. |
|-------------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f fixed-web

# Module 9 — Security hardening

## Lab 9.1 — Capabilities, read-only filesystems, and tmpfs

| **Level** | **Time** | **Primary outcome**                                        |
|-----------|----------|------------------------------------------------------------|
| Advanced  | 60 min   | Reduce container privileges without breaking the workload. |

**Concepts:** Linux capabilities, read-only rootfs, tmpfs, no-new-privileges

**Step 1 —** Run a hardened Nginx container.

> docker run -d --name secure-web -p 8092:8080 --read-only --tmpfs /var/cache/nginx --tmpfs /var/run --tmpfs /tmp --cap-drop ALL --cap-add NET_BIND_SERVICE --security-opt no-new-privileges:true nginxinc/nginx-unprivileged:alpine

**Step 2 —** Inspect security settings.

> docker inspect -f 'Readonly={{.HostConfig.ReadonlyRootfs}} CapDrop={{json .HostConfig.CapDrop}} CapAdd={{json .HostConfig.CapAdd}}' secure-web

**Step 3 —** Test the service and prove rootfs is read-only.

> curl http://localhost:8092  
> docker exec secure-web sh -c 'touch /should-fail' \|\| echo write-blocked

| **PASS CHECK** HTTP succeeds and writing to / fails. |
|------------------------------------------------------|

**Cleanup**

> docker rm -f secure-web

## Lab 9.2 — Secrets without baking them into images

| **Level** | **Time** | **Primary outcome**                                                  |
|-----------|----------|----------------------------------------------------------------------|
| Advanced  | 50 min   | Mount sensitive runtime material without storing it in image layers. |

**Concepts:** secret file, bind mount, permissions, image history

| **CAUTION** Do not print real secrets to the terminal, logs, shell history, or build output. |
|----------------------------------------------------------------------------------------------|

**Step 1 —** Create a demo secret with restrictive permissions.

> mkdir -p ~/docker-labs/secrets  
> printf 'demo-only-value\n' \> ~/docker-labs/secrets/api_key  
> chmod 600 ~/docker-labs/secrets/api_key

**Step 2 —** Mount it read-only at runtime.

> docker run --rm -v ~/docker-labs/secrets/api_key:/run/secrets/api_key:ro alpine sh -c 'test -r /run/secrets/api_key && echo secret-mounted'

**Step 3 —** Confirm the secret is not part of the image metadata.

> docker image inspect alpine --format '{{json .Config.Env}}'  
> docker history alpine

| **PASS CHECK** The container can read the mounted file, while image inspection/history does not contain its value. |
|--------------------------------------------------------------------------------------------------------------------|

## Lab 9.3 — Docker socket risk

| **Level** | **Time** | **Primary outcome**                                                 |
|-----------|----------|---------------------------------------------------------------------|
| Expert    | 35 min   | Demonstrate why docker-group and socket access are root-equivalent. |

**Concepts:** docker.sock, daemon authority, privilege boundary

| **CAUTION** Do not expose /var/run/docker.sock inside arbitrary containers and never publish the unauthenticated Docker API on TCP/2375. |
|------------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Inspect socket ownership.

> ls -l /var/run/docker.sock  
> id

**Step 2 —** Observe that a docker-group user can mount host paths.

> docker run --rm -v /etc/hostname:/host-hostname:ro alpine cat /host-hostname

**Step 3 —** Review which users belong to the group.

> getent group docker

| **PASS CHECK** You can explain why only trusted administrators should belong to docker. |
|-----------------------------------------------------------------------------------------|

# Module 10 — Daemon administration and reliability

## Lab 10.1 — Configure bounded local logs

| **Level** | **Time** | **Primary outcome**                                        |
|-----------|----------|------------------------------------------------------------|
| Expert    | 45 min   | Prevent unbounded container logs from filling the VM disk. |

**Concepts:** daemon.json, json-file rotation, daemon validation

| **CAUTION** A malformed daemon.json can prevent Docker from starting. Validate JSON first and keep your SSH session open while testing. |
|-----------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Inspect the current logging driver.

> docker info --format 'Logging={{.LoggingDriver}}'  
> sudo test -f /etc/docker/daemon.json && sudo cat /etc/docker/daemon.json \|\| true

**Step 2 —** Create or merge this configuration carefully using an editor.

> sudo nano /etc/docker/daemon.json

**Step 3 —** Use this minimal JSON if the file did not exist.

> {  
> "log-driver": "json-file",  
> "log-opts": {  
> "max-size": "10m",  
> "max-file": "3"  
> }  
> }

**Step 4 —** Validate JSON before restarting.

> python3 -m json.tool /etc/docker/daemon.json \>/dev/null && echo valid-json  
> sudo systemctl restart docker  
> docker info --format 'Logging={{.LoggingDriver}}'

**Step 5 —** Confirm options on a newly created container.

> docker run -d --name log-test alpine sleep 60  
> docker inspect -f '{{json .HostConfig.LogConfig}}' log-test

| **PASS CHECK** JSON validation succeeds, Docker restarts, and log-test inherits max-size 10m and max-file 3. |
|--------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f log-test

## Lab 10.2 — Restart policies and daemon restart

| **Level** | **Time** | **Primary outcome**                              |
|-----------|----------|--------------------------------------------------|
| Expert    | 35 min   | Predict service behavior across daemon restarts. |

**Concepts:** restart policy, unless-stopped, daemon lifecycle

**Step 1 —** Create two containers with different policies.

> docker run -d --name no-restart alpine sleep 3600  
> docker run -d --name auto-restart --restart unless-stopped alpine sleep 3600

**Step 2 —** Restart Docker.

> sudo systemctl restart docker  
> sleep 3  
> docker ps -a --format 'table {{.Names}}\t{{.Status}}'

**Step 3 —** Manually start the no-policy container if needed.

> docker start no-restart

| **PASS CHECK** auto-restart returns automatically; no-restart behavior is recorded and explained. |
|---------------------------------------------------------------------------------------------------|

**Cleanup**

> docker rm -f no-restart auto-restart

## Lab 10.3 — Diagnose daemon failures

| **Level** | **Time** | **Primary outcome**                                           |
|-----------|----------|---------------------------------------------------------------|
| Expert    | 40 min   | Collect authoritative evidence when the Docker service fails. |

**Concepts:** systemd, journal, socket, daemon configuration

**Step 1 —** Capture current service and socket state.

> systemctl status docker --no-pager  
> systemctl status docker.socket --no-pager  
> sudo ss -lx \| grep docker.sock

**Step 2 —** Read recent daemon logs.

> sudo journalctl -u docker --since '-15 minutes' --no-pager

**Step 3 —** Validate the daemon configuration.

> sudo python3 -m json.tool /etc/docker/daemon.json \>/dev/null && echo daemon-json-valid

**Step 4 —** Review storage and inode pressure.

> df -h / /var/lib/docker  
> df -i / /var/lib/docker  
> docker system df

| **PASS CHECK** You have a four-part incident checklist: systemd state, journal, configuration validity, and storage capacity. |
|-------------------------------------------------------------------------------------------------------------------------------|

# Module 11 — Registries, BuildKit, and remote administration

## Lab 11.1 — Run a private local registry

| **Level** | **Time** | **Primary outcome**                          |
|-----------|----------|----------------------------------------------|
| Expert    | 60 min   | Push and pull images through a lab registry. |

**Concepts:** registry, tag, push, pull, distribution

| **CAUTION** An insecure HTTP registry is acceptable only in this isolated training network. Production registries require TLS and authentication. |
|---------------------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** Start a registry bound only to the host-only address.

> docker run -d --name registry --restart unless-stopped -p 192.168.56.10:5000:5000 registry:2

**Step 2 —** Because this lab registry is HTTP, add it explicitly to daemon.json by merging this key.

> "insecure-registries": \["192.168.56.10:5000"\]

**Step 3 —** Validate and restart Docker.

> sudo python3 -m json.tool /etc/docker/daemon.json \>/dev/null  
> sudo systemctl restart docker

**Step 4 —** Tag and push the custom image.

> docker tag custom-web:v3 192.168.56.10:5000/custom-web:v3  
> docker push 192.168.56.10:5000/custom-web:v3

**Step 5 —** Remove the local tag, then pull it back.

> docker rmi 192.168.56.10:5000/custom-web:v3  
> docker pull 192.168.56.10:5000/custom-web:v3

| **PASS CHECK** The image is pulled back from your local registry. |
|-------------------------------------------------------------------|

**Cleanup**

> docker rm -f registry

## Lab 11.2 — BuildKit targets and build arguments

| **Level** | **Time** | **Primary outcome**                                              |
|-----------|----------|------------------------------------------------------------------|
| Expert    | 60 min   | Build distinct debug and production targets from one Dockerfile. |

**Concepts:** BuildKit, target, ARG, metadata, cache

**Step 1 —** Create a target-based Dockerfile.

> mkdir -p ~/docker-labs/build-targets  
> cd ~/docker-labs/build-targets  
> nano Dockerfile

**Step 2 —** Enter the Dockerfile.

> FROM alpine AS base  
> ARG BUILD_VERSION=dev  
> RUN echo \$BUILD_VERSION \> /version  
>   
> FROM base AS debug  
> RUN apk add --no-cache curl  
> CMD \["sh"\]  
>   
> FROM base AS production  
> CMD \["cat", "/version"\]

**Step 3 —** Build both targets.

> docker build --target debug --build-arg BUILD_VERSION=1.0 -t target-app:debug .  
> docker build --target production --build-arg BUILD_VERSION=1.0 -t target-app:prod .

**Step 4 —** Compare contents and sizes.

> docker images target-app  
> docker run --rm target-app:prod  
> docker run --rm target-app:debug curl --version \| head -1

| **PASS CHECK** The production target prints 1.0; only the debug target contains curl. |
|---------------------------------------------------------------------------------------|

| **STRETCH** Explain why ARG is not suitable for secrets even when the final stage does not visibly expose it. |
|---------------------------------------------------------------------------------------------------------------|

## Lab 11.3 — Manage the VM with a Docker SSH context

| **Level** | **Time** | **Primary outcome**                                                |
|-----------|----------|--------------------------------------------------------------------|
| Expert    | 50 min   | Control the remote Docker Engine securely from another Docker CLI. |

**Concepts:** context, SSH transport, remote daemon, socket authorization

| **CAUTION** Run this lab only if your Windows workstation has a Docker CLI. SSH is safer than exposing the daemon on an unauthenticated TCP socket. |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|

**Step 1 —** On a machine with Docker CLI and SSH access, create the context.

> docker context create ubuntu-lab --docker host=ssh://moshahid@192.168.56.10

**Step 2 —** Inspect and test it.

> docker context inspect ubuntu-lab  
> docker --context ubuntu-lab ps

**Step 3 —** Switch temporarily and switch back.

> docker context use ubuntu-lab  
> docker info --format '{{.Name}}'  
> docker context use default

| **PASS CHECK** The remote context lists containers from Test-VirtualBox over SSH. |
|-----------------------------------------------------------------------------------|

# Module 12 — Orchestration concepts with Swarm

## Lab 12.1 — Single-node Swarm services

| **Level** | **Time** | **Primary outcome**                                                |
|-----------|----------|--------------------------------------------------------------------|
| Expert    | 60 min   | Understand desired state, replicas, services, and rolling updates. |

**Concepts:** Swarm, service, replica, desired state, update

**Step 1 —** Initialize a single-node training swarm on the host-only IP.

> docker swarm init --advertise-addr 192.168.56.10

**Step 2 —** Create a replicated service.

> docker service create --name swarm-web --publish published=8093,target=80 --replicas 3 nginx:alpine

**Step 3 —** Inspect tasks and service state.

> docker service ls  
> docker service ps swarm-web

**Step 4 —** Scale and update.

> docker service scale swarm-web=5  
> docker service update --image nginx:stable-alpine swarm-web  
> docker service ps swarm-web

**Step 5 —** Test from Ubuntu and Windows.

> curl http://localhost:8093

| **PASS CHECK** Five replicas converge and the published service answers on port 8093. |
|---------------------------------------------------------------------------------------|

| **STRETCH** Compare Swarm service desired state with a standalone docker run container. |
|-----------------------------------------------------------------------------------------|

**Cleanup**

> docker service rm swarm-web  
> docker swarm leave --force

## Lab 12.2 — Swarm secrets and configs

| **Level** | **Time** | **Primary outcome**                                               |
|-----------|----------|-------------------------------------------------------------------|
| Expert    | 50 min   | Deliver runtime configuration without embedding it into an image. |

**Concepts:** Swarm secret, config, service mount

**Step 1 —** Initialize Swarm if Lab 12.1 cleanup was completed.

> docker swarm init --advertise-addr 192.168.56.10

**Step 2 —** Create a secret and config.

> printf 'training-secret-value' \| docker secret create app_secret -  
> printf 'feature=true\n' \| docker config create app_config -

**Step 3 —** Mount both into a service.

> docker service create --name secret-reader --secret app_secret --config app_config alpine sleep 600

**Step 4 —** Locate the task container and verify mounts.

> docker ps --filter name=secret-reader  
> docker exec \$(docker ps -q --filter name=secret-reader) sh -c 'ls -l /run/secrets; cat /app_config'

**Step 5 —** Remove the service, then the objects.

> docker service rm secret-reader  
> docker secret rm app_secret  
> docker config rm app_config

| **PASS CHECK** The task sees /run/secrets/app_secret and /app_config; objects are removed after the service. |
|--------------------------------------------------------------------------------------------------------------|

**Cleanup**

> docker swarm leave --force

# Module 13 — Capstone projects

| **CAPSTONE METHOD** For each project, design first, implement from source-controlled files, validate every requirement, intentionally break one component, diagnose it, repair it, and document cleanup. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Capstone A — Production-style web stack

| **Requirement** | **Acceptance test**                                                                                       |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| Reverse proxy   | Nginx publishes one host-only port and routes to the application by service name.                         |
| Application     | A non-root container exposes an internal unprivileged port.                                               |
| Database        | PostgreSQL stores data in a named volume and is not published to Windows.                                 |
| Reliability     | Health checks, restart policies, and dependency conditions are defined.                                   |
| Security        | Read-only filesystems where practical; secrets are runtime-mounted; unnecessary capabilities are dropped. |
| Operations      | Bounded logs, resource limits, backup and restore instructions, and a troubleshooting checklist exist.    |
| Delivery        | docker compose config passes; a clean docker compose up -d builds and starts the stack.                   |

**Deliverable —** Create ~/docker-labs/capstone-web containing compose.yaml, Dockerfiles, source files, .dockerignore, README.md, backup.sh, restore.sh, and evidence.txt.

**Failure drill —** Break the database hostname, capture the application error, use logs/inspect/network commands to find it, repair it, and record the evidence.

## Capstone B — Local build and registry pipeline

| **Requirement** | **Acceptance test**                                                                         |
|-----------------|---------------------------------------------------------------------------------------------|
| Build           | Multi-stage Dockerfile produces debug and production targets.                               |
| Quality         | A containerized test command runs before the production tag is created.                     |
| Metadata        | OCI labels record title, version, source, and revision.                                     |
| Registry        | The versioned image is pushed to the isolated local registry.                               |
| Rollback        | Two versions are retained and the previous version can be redeployed.                       |
| Evidence        | Image digest, history, size, SBOM/tool output if available, and rollback test are recorded. |

**Deliverable —** Create ~/docker-labs/capstone-pipeline with a Makefile or shell-based workflow, Dockerfile, source, tests, and README.md. Never place credentials in ARG, ENV, image layers, or Git.

## Capstone C — Docker operations incident

| **Injected fault**            | **Required diagnosis**                                                           |
|-------------------------------|----------------------------------------------------------------------------------|
| Container exits immediately   | Use ps -a, inspect State, exit code, and logs.                                   |
| Port unavailable              | Use docker port, inspect, ss, curl, and host binding analysis.                   |
| Service name does not resolve | Inspect networks and prove whether both containers share a user-defined network. |
| Disk pressure                 | Use df, inode checks, docker system df -v, and log-file inspection.              |
| Unhealthy service             | Read health-check history and test the check command manually.                   |
| Permission failure            | Compare container UID/GID, mount ownership, mode, and read-only settings.        |

**Deliverable —** Write an incident report with symptom, timeline, evidence, root cause, corrective action, validation, and prevention. Do not solve by disabling TLS, running privileged, or granting world-writable permissions.

# Final practical exam

| **Section**         | **Points** | **Evidence**                                                  |
|---------------------|------------|---------------------------------------------------------------|
| Container lifecycle | 10         | Create, inspect, stop/start, logs, exec, and cleanup          |
| Images and builds   | 15         | Reproducible Dockerfile, cache awareness, non-root runtime    |
| Storage             | 10         | Named-volume persistence plus tested backup/restore           |
| Networking          | 15         | Private service DNS and deliberate published-port binding     |
| Compose             | 15         | Validated multi-service application with health checks        |
| Security            | 15         | Least privilege, secrets handling, read-only design           |
| Operations          | 10         | Resource limits, bounded logs, evidence-based troubleshooting |
| Documentation       | 10         | README, validation results, rollback and cleanup instructions |

| **PASS STANDARD** 80/100 overall, with no critical security failure. Critical failures include committing a real secret, exposing Docker TCP/2375 without protection, using privileged as a generic fix, or deleting unknown volumes. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# Command reference

| **Task**           | **Command**                                     |
|--------------------|-------------------------------------------------|
| Running containers | docker ps                                       |
| All containers     | docker ps -a                                    |
| Images             | docker images                                   |
| Logs               | docker logs --tail 100 NAME                     |
| Enter container    | docker exec -it NAME sh                         |
| Inspect            | docker inspect NAME                             |
| Processes          | docker top NAME                                 |
| Resource usage     | docker stats --no-stream                        |
| Networks           | docker network ls                               |
| Volumes            | docker volume ls                                |
| Compose validate   | docker compose config                           |
| Compose start      | docker compose up -d                            |
| Compose remove     | docker compose down                             |
| Disk accounting    | docker system df -v                             |
| Daemon logs        | sudo journalctl -u docker --since "-15 minutes" |

# Troubleshooting decision path

**1. Is the daemon available? —** systemctl status docker; docker info

**2. Does the container exist? —** docker ps -a --filter name=NAME

**3. Why did it exit? —** docker inspect State and docker logs

**4. Is the port published correctly? —** docker port, inspect PortBindings, and ss -lntp

**5. Can containers resolve each other? —** inspect network membership and test service DNS from a peer container

**6. Is storage mounted as expected? —** inspect Mounts, host ownership, permissions, and read-only flags

**7. Is the application healthy? —** inspect State.Health and run the health command manually

**8. Is the host constrained? —** df -h, df -i, free -h, docker stats, docker system df -v

**9. Is TLS inspection involved? —** inspect the presented issuer; trust only the approved company CA; never bypass verification

# Recommended study schedule

| **Week** | **Focus**     | **Outcome**                                       |
|----------|---------------|---------------------------------------------------|
| 1        | Modules 1–3   | Fluent container and image lifecycle              |
| 2        | Modules 4–5   | Persistent data and service networking            |
| 3        | Module 6      | Reproducible and efficient image builds           |
| 4        | Module 7      | Multi-container applications with Compose         |
| 5        | Modules 8–9   | Troubleshooting, health, limits, and security     |
| 6        | Modules 10–11 | Daemon operations, registries, BuildKit, contexts |
| 7        | Module 12     | Desired-state orchestration concepts              |
| 8        | Module 13     | Complete all capstones and final exam             |
