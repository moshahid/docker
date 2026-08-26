# Unix Username Generator — Docker

## Start with Docker Compose

```bash
tar -xzf unix-username-generator-docker.tar.gz
cd unix-username-generator-docker
docker compose up -d --build
```

Open `http://UBUNTU_SERVER_IP:8080` in your browser.

## Check the container

```bash
docker compose ps
docker compose logs --tail=50
curl http://localhost:8080/health
```

## Stop or update

```bash
docker compose down

# After changing the files:
docker compose up -d --build
```

## Run without Compose

```bash
docker build -t unix-username-generator:1.0 .
docker run -d --name username-generator \
  --restart unless-stopped \
  -p 8080:80 \
  unix-username-generator:1.0
```

The browser app generates names only. It does not create Linux users or check
the host's `/etc/passwd`. Check a selected name on the Ubuntu host with:

```bash
getent passwd GENERATED_USERNAME
```
