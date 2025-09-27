# 🚀 Quay Setup on Docker with Postgres, Redis, and MinIO (HTTPS)

This guide explains how to set up [Quay](https://quay.io/) on an Ubuntu server using Docker.  
We’ll use **docker-compose** for Postgres, Redis, and MinIO (with HTTPS), and run Quay separately via `docker run`.  
Configuration is done through Quay’s browser wizard.

---

## 📋 Prerequisites

Install Docker and Docker Compose:

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker
```
---
## 📂 Directory Structure

Create Workspace
```bash
mkdir -p ~/quay-registry/{postgres/{data,initdb.d},redis/data,minio/{data,certs},quay/{config,storage}}
cd ~/quay-registry
```
---
## ⚙️ Environment Variables
Create a .env file:
```.env
# Postgres
POSTGRES_USER=quay
POSTGRES_PASSWORD=quaypass
POSTGRES_DB=quay
POSTGRES_PORT=5432

# Redis
REDIS_PORT=6379

# MinIO (HTTPS on 9000)
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minio123
MINIO_S3_PORT=9000
MINIO_CONSOLE_PORT=9001
MINIO_BUCKET=quay-storage

# Quay
QUAY_HTTP_PORT=8080
QUAY_CONFIG_PORT=8081
```

Load it:
```bash
set -a; source .env; set +a
```
---
## 🔒 Generate TLS Certificates for MinIO
```bash
cd ~/quay-registry/minio/certs

# Root CA
openssl genrsa -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 \
  -subj "/CN=QuayDemo-RootCA" -out rootCA.pem

# Server cert for quay-minio
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=quay-minio" -out server.csr

# SAN file
cat > san.cnf <<EOF
subjectAltName = @alt_names
[alt_names]
DNS.1 = quay-minio
EOF

# Sign server cert
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial \
  -out public.crt -days 825 -sha256 -extfile san.cnf

# MinIO expects these names
cp server.key private.key
```
---
## 🐳 docker-compose.yml
`~/quay-registry/docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: quay-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
      - ./postgres/initdb.d:/docker-entrypoint-initdb.d
    networks: [quay-net]

  redis:
    image: redis:7-alpine
    container_name: quay-redis
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "${REDIS_PORT}:6379"
    volumes:
      - ./redis/data:/data
    networks: [quay-net]

  minio:
    image: minio/minio:latest
    container_name: quay-minio
    restart: unless-stopped
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    command: server /data --console-address ":${MINIO_CONSOLE_PORT}" --address ":${MINIO_S3_PORT}"
    ports:
      - "${MINIO_S3_PORT}:${MINIO_S3_PORT}"
      - "${MINIO_CONSOLE_PORT}:${MINIO_CONSOLE_PORT}"
    volumes:
      - ./minio/data:/data
      - ./minio/certs:/root/.minio/certs
    networks: [quay-net]

networks:
  quay-net: {}
```

Start services:
```bash
docker compose -p quay-registry --env-file .env up -d
```
---
## 🪣 Create MinIO Bucket

You have two options to create the bucket:

Option 1: Using MinIO CLI (mc)
```bash
NET=quay-registry_quay-net

# Create bucket
docker run --rm --network "$NET" \
  -e MC_HOST_quayminio="https://${MINIO_ROOT_USER}:${MINIO_ROOT_PASSWORD}@quay-minio:${MINIO_S3_PORT}" \
  minio/mc:latest --insecure mb -p "quayminio/${MINIO_BUCKET}"

# List all buckets
docker run --rm --network "$NET" \
  -e MC_HOST_quayminio="https://${MINIO_ROOT_USER}:${MINIO_ROOT_PASSWORD}@quay-minio:${MINIO_S3_PORT}" \
  minio/mc:latest --insecure ls quayminio

# List objects inside the bucket
docker run --rm --network "$NET" \
  -e MC_HOST_quayminio="https://${MINIO_ROOT_USER}:${MINIO_ROOT_PASSWORD}@quay-minio:${MINIO_S3_PORT}" \
  minio/mc:latest --insecure ls "quayminio/${MINIO_BUCKET}"

```

Option 2: Using MinIO Console (Web UI)
1. Open your browser:
👉 `https://<EC2-PUBLIC-IP>:9001` (default console port = 9001)

2. Login with credentials (from .env):
  - Username: `minioadmin`
  - Password: `minio123`

3. Go to Buckets → Create Bucket

4. Enter name: `quay-storage` (must match `.env`)

5. Save ✅
---
## 🗄 Enable Postgres Extension
```bash
docker exec -it quay-postgres \
  psql -U "${POSTGRES_USER}" -d "${POSTGRES_DB}" \
  -c 'CREATE EXTENSION IF NOT EXISTS pg_trgm;'
```
---
## ⚙️ Run Quay Config Wizard
```bash
mkdir -p ~/quay-registry/quay/config/extra_ca_certs
cp ~/quay-registry/minio/certs/rootCA.pem ~/quay-registry/quay/config/extra_ca_certs/

docker run --rm --name quay-config \
  --network quay-registry_quay-net \
  -p ${QUAY_CONFIG_PORT}:8080 \
  -v "$(pwd)/quay/config:/conf/stack" \
  -v "$(pwd)/quay/storage:/datastorage" \
  quay.io/projectquay/quay:latest config secret
```

Open in browser:
👉 `http://<EC2-PUBLIC-IP>:8081`
Login: `quayconfig / secret`
[Config Login](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/config_login.PNG)

![Config Page](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/config_page.PNG)

Fill wizard values (S3 = MinIO, DB = Postgres, Redis, etc.), then download config tarball.
1. Server Configuration & Database:
![Server DB Conf](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/server_db_conf.PNG)

2. Redis
![Redis Conf](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/redis_conf.PNG)

3. Registry Storage:
![Registry Storage Setup](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/registry_storage_setup.PNG)

4. Click on `Validation Configuration Changes` and Download the tar file.
![Validation](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/validation.PNG)
---
## 📦 Extract Config
```bash
tar -xzvf ~/quay-config.tar.gz -C ~/quay-registry/quay/config --no-same-owner
ls -la ~/quay-registry/quay/config
```
## Edit Config file
Comment this section in config.yaml:
```bash
vim quay/config/config.yaml
```
```
DB_CONNECTION_ARGS:
  keepalives: 0
  keepalivescount: 0
  keepalivesidle: 0
  keepalivesinterval: 0
  sslcompression: 0
  tcpusertimeout: 0
```

---
## ▶️ Run Quay in Normal Mode
```bash
docker rm -f quay 2>/dev/null || true

docker run -d --name quay \
  --restart=always \
  --network quay-registry_quay-net \
  -p ${QUAY_HTTP_PORT}:8080 \
  -v "$(pwd)/quay/config:/conf/stack:Z" \
  -v "$(pwd)/quay/storage:/datastorage:Z" \
  quay.io/projectquay/quay:latest
```
Open Quay:
👉 `http://<EC2-PUBLIC-IP>:8080`

Login Page:

![Quay Registry Login Page](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/quay_registry_login_page.PNG)

Create a user for Quay registry. Here, `username: quayadmin`, `email:<any_email>`, `password:<set_password>`,

![Registry User Create Quayadmin](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/registry_usercreate_quayadmin.PNG)

Loggin with created user `quayadmin`:

![Loggedin With Quayadmin](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/loggedin_with_quayadmin.PNG)

---
## ✅ Health Checks
```bash
docker run --rm --network quay-registry_quay-net curlimages/curl -skI https://quay-minio:9000/minio/health/ready
docker logs --tail=100 quay
```
---
## 📤 Test Push & Pull with Quay
If Quay is running on HTTP (8080), you must allow Docker to treat it as an insecure registry:
```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["<EC2-PRIVATE-IP>:8080"]
}
EOF

sudo systemctl restart docker
```
![Insecure Registry Add](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/insecure_registry_add.PNG)

Then test:
# Login to Quay
Example: `docker login <EC2-PRIVATE-IP>:8080`
```bash
docker login 172.31.28.163:8080
```

![Quay Registry CLI Loggedin](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/quay_registry_cli_loggedin.PNG)

# Pull a sample image
Example: `docker pull <sample_image>`
```bash
docker pull busybox
```

# Tag it for Quay
Example: `docker image tag busybox:latest <EC2-PRIVATE-IP>:8080/<youruser>/hello:latest`
```bash
docker image tag busybox:latest 172.31.28.163:8080/quayadmin/testrepo/busybox:v1
```

# Push it to Quay
Example: `docker push <EC2-PRIVATE-IP>:8080/<youruser>/hello:latest`
```bash
docker image push 172.31.28.163:8080/quayadmin/testrepo/busybox:v1
```

![Push Docker Image To Quay](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/push_docker_image_to_quay.PNG)

![Pushed Busybox From CLI](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/pushed_busybox_from_cli.PNG)

# Pull it back
Example: `docker pull <EC2-PRIVATE-IP>:8080/<youruser>/hello:latest`

```bash
docker image pull 172.31.28.163:8080/quayadmin/testrepo/busybox:v1
```

![Pull Docker Image From Quay](https://github.com/yashshrivastav22/Images/blob/main/Quay-Registry/pull_docker_image_from_quay.PNG)

---
## 📂 Final Layout
```kotline
~/quay-registry/
├─ .env
├─ docker-compose.yml
├─ postgres/
│  ├─ data/
│  └─ initdb.d/
├─ redis/
│  └─ data/
├─ minio/
│  ├─ data/
│  └─ certs/           # public.crt, private.key, rootCA.pem
└─ quay/
   ├─ config/
   │  ├─ config.yaml
   │  └─ extra_ca_certs/rootCA.pem
   └─ storage/
```
---
