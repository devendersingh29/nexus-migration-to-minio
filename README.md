# nexus-migration-to-minio

# Nexus Docker Registry Migration to On-Premise MinIO

A step-by-step guide to running a new Nexus Repository (3.85.0) with its blob storage on an on-premise MinIO server, fronted by nginx, and migrating Docker images from the old Nexus.

**Hostnames used in this guide**

| Hostname | Goes to | Used for |
|----------|---------|----------|
| `old.repo.nexus.tech` | nginx, then the docker **group** (port 9006) and the Nexus UI | **Pull** |
| `new.repo.nexus.tech` | nginx, then the docker **hosted** repo (port 9007) | **Push** |

Both hostnames point at the **new** Nexus through nginx. The names show the side each one serves: pulling through `old.repo.nexus.tech` still finds the old images (via the proxy), and pushing to `new.repo.nexus.tech` puts images into the new hosted repo.

---

## 1. Architecture

```
          Clients (docker / Kubernetes / CI)

              │ pull                            │ push
              ▼                                 ▼
     old.repo.nexus.tech               new.repo.nexus.tech
              └──────── nginx :443 (TLS) ───────┘
              │ /v2/ → :9006 (group)            │ /v2/ → :9007 (hosted)
              ▼                                 ▼
        ┌───────────────── New Nexus 3.85.0 ─────────────────┐
        │  docker-group (port 9006)                          │
        │    1st: docker-hosted (9007 HTTP / 9008 HTTPS)     │
        │    2nd: docker-proxy ──► OLD Nexus :9009           │
        │                                                    │
        │  Blob store (S3 type) ──► MinIO                    │
        └──────────────────────────┬─────────────────────────┘
                                   ▼
                         MinIO :9000 (S3 API)
                                   ▼
                         External SSD (/data)
```

**New repo: pull and push.** Pull checks the new repo first, then falls back to the old Nexus. Push goes only to the new repo.

**How it works**

- **Pulls** go to the group. The group looks in the new hosted repo first, then falls back to the proxy, which fetches from the old Nexus and caches the image in MinIO. Migration happens lazily as images are pulled.
- **Pushes** go straight to the hosted repo through a separate hostname, so writes always land on the real push target.
- **Bulk copy** (Step 8) moves everything that nobody has pulled yet, so the old Nexus can be retired.

---

## 2. Port Reference

### Host machine running the new Nexus (192.168.1.24 in this setup)

| Port | Service | Purpose |
|------|---------|---------|
| 8081 | Nexus | Web UI and REST API. nginx proxies the Nexus UI here. |
| 9006 | Nexus Docker connector | **docker-group**: the pull endpoint. Serves hosted first, then proxy. |
| 9007 | Nexus Docker connector | **docker-hosted (HTTP)**: the push endpoint. nginx uses this one. |
| 9008 | Nexus Docker connector | **docker-hosted (HTTPS)**: only works if Nexus itself has a TLS keystore configured. Optional, because nginx terminates TLS. |
| 9005 | Nexus Docker connector | Published in the compose file but **not used** by this design. Keep it as a spare, or remove it. |
| 9009 | Nexus Docker connector | Published in the compose file but **not used by the new Nexus** in this design. The 9009 in the proxy config is the **old** Nexus's Docker port. |

**Why so many ports?** The Docker registry API always lives at `/v2/` on the root of a host. Nexus therefore gives every Docker repository its own connector port, because it cannot tell repositories apart by path. Each port maps to exactly one repository.

### MinIO host

| Port | Purpose |
|------|---------|
| 9000 | S3 API. Nexus connects here. |
| 9001 | MinIO web console (admin UI). |

### nginx host (192.168.1.134 in this setup)

| Port | Purpose |
|------|---------|
| 443 | HTTPS entry point for both `old.repo.nexus.tech` (pull) and `new.repo.nexus.tech` (push). TLS is terminated here. Forwards to 9006, 9007 and 8081. |

### Old Nexus

| Port | Purpose |
|------|---------|
| 9009 | Docker connector that the new Nexus proxy repo pulls from. |

Firewall: open 443 to clients. Open 9000 from the Nexus host to MinIO, and 9009 from the new Nexus to the old Nexus. Ports 9005 to 9008 and 8081 only need to be reachable from nginx (and admins).

---

## 3. Prerequisites

- Docker and Docker Compose on the Nexus host and the MinIO host (they can be the same machine).
- Old Nexus reachable from the new Nexus on its Docker port (9009), with an account that can read the Docker repository.
- A TLS certificate that covers **both** hostnames (`old.repo.nexus.tech` and `new.repo.nexus.tech`). A `*.repo.nexus.tech` wildcard does this.
- DNS records (or hosts-file entries while testing) for both hostnames pointing at the nginx host.
- Enough RAM on the Nexus host for `-Xmx8g` plus `MaxDirectMemorySize=4g`.
- The `mc` (MinIO client) and `skopeo` tools on an admin machine.

---

## 4. Step 1: Start MinIO

### 4.1 Prepare the external SSD

- Format the SSD as **XFS or ext4**. exFAT and NTFS cause problems with MinIO.
- Mount it through `/etc/fstab` so it is always mounted before Docker starts. If it is not mounted at boot, MinIO would write to the empty mountpoint on the root disk.
- Make the data directory writable by the container user (UID/GID 1000):

```bash
sudo mkdir -p "/media/test/Extreme SSD/minio-data"
sudo chown -R 1000:1000 "/media/test/Extreme SSD/minio-data"
```

### 4.2 Credentials file

Create `.env` next to the compose file. **The root password must be at least 8 characters**, or MinIO will refuse to start.

```env
MINIO_ROOT_USER=<admin-user>
MINIO_ROOT_PASSWORD=<strong-password-8+chars>
```

### 4.3 `minio-docker-compose.yaml`

```yaml
services:
  minio:
    image: quay.io/minio/minio:RELEASE.2024-12-18T13-15-44Z
    container_name: minio
    restart: unless-stopped
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web console
    user: "1000:1000"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - "/media/test/Extreme SSD/minio-data:/data"
    command: server /data --console-address ":9001"
```

```bash
docker compose -f minio-docker-compose.yaml up -d
docker logs minio        # should show the API and console URLs, no errors
```

---

## 5. Step 2: Create the Bucket and an Access Key

Do not use the root account for Nexus. Create a dedicated user limited to one bucket.

```bash
# Register the server with mc
mc alias set local http://<minio-host>:9000 <admin-user> '<admin-password>'

# Create the bucket (lowercase, no underscores)
mc mb local/nexus-blobs

# Bucket-scoped policy
cat > nexus-blobs-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::nexus-blobs",
        "arn:aws:s3:::nexus-blobs/*"
      ]
    }
  ]
}
EOF

mc admin policy create local nexus-blobs-rw nexus-blobs-policy.json

# In MinIO, the user name is the access key and the password is the secret key
mc admin user add local nexus '<nexus-secret-8+chars>'
mc admin policy attach local nexus-blobs-rw --user nexus
```

Nexus also sets a lifecycle rule on the bucket to expire deleted blobs, so the user needs bucket-level permissions (the policy above includes them).

---

## 6. Step 3: Start the New Nexus

### 6.1 Prepare the data directory

The container runs as UID/GID 200, so that user must own the data directory:

```bash
sudo mkdir -p /opt/latest-nexus
sudo chown -R 200:200 /opt/latest-nexus
```

### 6.2 `nexus-docker-compose.yaml`

```yaml
services:
  nexus:
    image: sonatype/nexus3:3.85.0
    ports:
      - "8081:8081"   # Web UI / API
      - "9005:9005"   # spare
      - "9006:9006"   # docker-group (pull)
      - "9007:9007"   # docker-hosted HTTP (push)
      - "9008:9008"   # docker-hosted HTTPS
      - "9009:9009"   # spare
    volumes:
      - /opt/latest-nexus:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms4g -Xmx8g -XX:MaxDirectMemorySize=4g -XX:+UseG1GC -XX:MaxGCPauseMillis=500 -Djava.util.prefs.userRoot=/tmp
    restart: unless-stopped
    user: "200:200"
```

The `deploy:` block from the original file is Swarm syntax and overlaps with `restart: unless-stopped`, so it can be dropped.

```bash
docker compose -f nexus-docker-compose.yaml up -d
docker logs -f nexus      # wait for "Started Sonatype Nexus"
```

Open `http://<nexus-host>:8081`. The initial admin password is in `/opt/latest-nexus/admin.password`.

**Important:** the Nexus database (component metadata, users, repository config) stays in `/opt/latest-nexus`. MinIO only holds the blobs. **Back up both** if you need a restorable system.

---

## 7. Step 4: Create the S3 Blob Store (MinIO)

**Settings → Repository → Blob Stores → Create blob store → type S3**

| Field | Value |
|-------|-------|
| Name | `minio` |
| Region | `us-east-1` (any value works for MinIO) |
| Bucket | `nexus-blobs` |
| Authentication: Access Key ID | `nexus` |
| Authentication: Secret Access Key | the secret from Step 2 |
| Advanced → Endpoint URL | `http://<minio-host>:9000` |
| Advanced → **Use path-style access** | **enabled** (required for MinIO) |

If MinIO is behind TLS with an internal CA, import that CA into the Nexus Java truststore first.

Save. A green status means Nexus can reach the bucket. If it fails, see Troubleshooting.

---

## 8. Step 5: Create the Repositories

### 8.1 Enable the Docker Bearer Token Realm

**Settings → Security → Realms**: move **Docker Bearer Token Realm** to the Active list and save. Without it, `docker login` fails.

### 8.2 docker-hosted (the push target)

**Repositories → Create → docker (hosted)**

| Field | Value |
|-------|-------|
| Name | `docker-hosted` |
| HTTP connector port | `9007` |
| HTTPS connector port | `9008` (only if Nexus has its own TLS keystore; otherwise leave empty) |
| Blob store | `minio` |
| Deployment policy | Allow redeploy (if you reuse tags such as `latest`) |
| Allow anonymous docker pull | your choice |

### 8.3 docker-proxy (pulls from the old Nexus)

**Repositories → Create → docker (proxy)**

| Field | Value |
|-------|-------|
| Name | `docker-proxy` |
| Remote storage | `http://<old-nexus-host>:9009` (the old Nexus's Docker port) |
| Docker index | Use registry (not Docker Hub) |
| Authentication | username and password of the old Nexus account |
| Blob store | `minio` (so cached images land in MinIO, not on local disk) |

The proxy does not need its own connector port. Clients reach it through the group.

### 8.4 docker-group (the pull endpoint)

**Repositories → Create → docker (group)**

| Field | Value |
|-------|-------|
| Name | `docker-group` |
| HTTP connector port | `9006` |
| Blob store | `minio` |
| Member repositories, in this order | **1. `docker-hosted`, 2. `docker-proxy`** |

Order matters. Nexus searches members top to bottom, so anything already migrated or newly pushed is served from the new hosted repo, and only missing images are fetched from the old Nexus.

If your Nexus edition offers a writable member on the docker group, you can select `docker-hosted` there and push through the group as well. The separate push hostname below works either way.

---

## 9. Step 6: nginx (TLS and Hostname Routing)

Docker requires HTTPS for remote registries, so nginx terminates TLS on port 443.

Two hostnames keep pulls and pushes cleanly separated:

- **Pull host:** `old.repo.nexus.tech` goes to the group (9006) and also serves the Nexus UI.
- **Push host:** `new.repo.nexus.tech` goes to the hosted repo (9007).

Both hostnames give access to the same image paths, so an image pushed to one is visible from the other. Routing by hostname avoids the problem you get when routing by HTTP method: `HEAD` requests are used by both pushes (blob checks) and pulls (manifest checks in containerd, skopeo and others). Sending all `HEAD`s to the hosted repo would make pulls of not-yet-migrated images fail with 404.

### 9.1 Shared settings: `/etc/nginx/snippets/nexus-docker.conf`

```nginx
proxy_request_buffering off;
proxy_buffering off;
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
client_max_body_size 0;
proxy_connect_timeout 600;
proxy_send_timeout 600;
proxy_read_timeout 600;
```

### 9.2 Server blocks (in `nginx.conf` or a file under `conf.d/`)

```nginx
# ---- Pull host: docker group + Nexus UI ----
server {
    listen 192.168.1.134:443 ssl;
    server_name old.repo.nexus.tech;

    ssl_certificate     "/path/to/cert-covering-repo.nexus.tech.crt";   # e.g. *.repo.nexus.tech
    ssl_certificate_key "/path/to/its.key";

    location /v2/ {
        proxy_pass http://192.168.1.24:9006;      # docker-group
        include /etc/nginx/snippets/nexus-docker.conf;
    }

    location / {
        proxy_pass http://192.168.1.24:8081;      # Nexus UI
        include /etc/nginx/snippets/nexus-docker.conf;
        client_max_body_size 10G;
    }
}

# ---- Push host: docker-hosted ----
server {
    listen 192.168.1.134:443 ssl;
    server_name new.repo.nexus.tech;

    ssl_certificate     "/path/to/cert-covering-repo.nexus.tech.crt";
    ssl_certificate_key "/path/to/its.key";

    location /v2/ {
        proxy_pass http://192.168.1.24:9007;      # docker-hosted
        include /etc/nginx/snippets/nexus-docker.conf;
    }
}
```

Notes:

- The `listen` line must include **`ssl`**. Without it, nginx serves plain HTTP on 443 and ignores the certificate.
- The certificate must cover both hostnames. The old `*.walkingtree.tech` certificate does **not** cover `*.repo.nexus.tech`, so use a certificate (or wildcard) for `repo.nexus.tech`, or your own CA distributed to the Docker hosts.
- Add DNS records for both hostnames pointing to the nginx host.
- Remove any earlier `docker-stage-repo.map.conf` and old `stage-repo.walkingtree.tech` server block so nothing conflicts.

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 10. Step 7: Test

```bash
# 1. Log in and push to the push host
docker login new.repo.nexus.tech
docker pull alpine:3.20
docker tag alpine:3.20 new.repo.nexus.tech/test/alpine:3.20
docker push new.repo.nexus.tech/test/alpine:3.20

# 2. Pull the same image through the pull host (from another machine if possible)
docker pull old.repo.nexus.tech/test/alpine:3.20

# 3. Pull an image that exists ONLY on the old Nexus (tests the proxy fallback)
docker pull old.repo.nexus.tech/<old-only-image>:<tag>

# 4. Manifest HEAD check, the way containerd does it
curl -sI -u <user>:<pass> \
  -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' \
  https://old.repo.nexus.tech/v2/<old-only-image>/manifests/<tag>
# expect: HTTP/1.1 200

# 5. Confirm data is landing in MinIO
mc du local/nexus-blobs
mc ls --recursive local/nexus-blobs | head
```

If you use Kubernetes/containerd nodes, also test with `crictl pull old.repo.nexus.tech/<image>:<tag>` on a node.

---

## 11. Step 8: Bulk-Copy the Remaining Images

Pull-through only copies images that someone requests. To retire the old Nexus you must copy everything else. `skopeo copy --all` preserves digests and multi-arch manifests.

```bash
OLD=<old-nexus-host>:9009
NEW=new.repo.nexus.tech          # push host (goes to docker-hosted)

for repo in $(curl -s -u <user>:<pass> "http://$OLD/v2/_catalog?n=1000" | jq -r '.repositories[]'); do
  for tag in $(curl -s -u <user>:<pass> "http://$OLD/v2/$repo/tags/list" | jq -r '.tags[]?'); do
    echo "Copying $repo:$tag"
    skopeo copy --all \
      --src-tls-verify=false \
      --src-creds <user>:<pass> \
      --dest-creds <user>:<pass> \
      docker://$OLD/$repo:$tag docker://$NEW/$repo:$tag
  done
done
```

- Remove `--src-tls-verify=false` if the old Nexus uses valid HTTPS. Keep it for plain HTTP.
- If the catalog is large, follow the `Link` header for pagination.
- Run it during a quiet period. It reads and writes every layer.

Then compare the two sides:

```bash
curl -s -u <user>:<pass> "http://<old-nexus-host>:9009/v2/_catalog?n=1000" | jq '.repositories | length'
curl -s -u <user>:<pass> "https://new.repo.nexus.tech/v2/_catalog?n=1000" | jq '.repositories | length'
```

---

## 12. Step 9: Cutover and Decommission

1. Point CI/CD pipelines, Kubernetes manifests, Helm values and developer instructions at the new hostnames (`old.repo.nexus.tech` for pulls, `new.repo.nexus.tech` for pushes).
2. Keep the old Nexus running during a transition period so the proxy fallback still works for anything missed.
3. After the bulk copy is verified, edit `docker-group` and **remove `docker-proxy`** from its members. Delete the proxy repo when nothing depends on it.
4. Shut down the old Nexus and keep its data as a backup for a while.
5. Set up backups: `/opt/latest-nexus` (Nexus database and config) and the MinIO data on the SSD (or `mc mirror` to another location).

---

## 13. Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| MinIO container exits right after start | Root password shorter than 8 characters, or the data directory is not writable by UID 1000. Check `docker logs minio`. |
| MinIO data ends up on the wrong disk | The SSD was not mounted when the container started. Mount via `/etc/fstab` and confirm before starting. |
| Nexus blob store creation fails | Endpoint URL wrong, or **path-style access** not enabled. Bucket missing or the user lacks permission. `localhost` inside the container is Nexus itself, so use the MinIO host IP or a shared Docker network. |
| TLS or certificate errors on the blob store | Import the MinIO CA into the Nexus Java truststore, or use `http://` internally. |
| Nexus fails to start with permission errors | `/opt/latest-nexus` is not owned by 200:200. |
| `docker login` returns 401 | Docker Bearer Token Realm is not active. Check the credentials and the nginx `Host` and `X-Forwarded-Proto` headers. |
| `x509: certificate is valid for ..., not ...` | The certificate does not cover the hostname you are using. |
| Plain HTTP served on 443, or TLS handshake errors | `listen` line is missing `ssl`. |
| `blob unknown` on push | Push went through the group or a proxy instead of the hosted repo. Push only to `new.repo.nexus.tech` (9007). |
| Pull of an old-only image returns 404 | `docker-proxy` is not a group member, the remote URL or credentials are wrong, or you pulled through the push host instead of `old.repo.nexus.tech`. |
| Pushing large layers fails or times out | Missing `client_max_body_size 0` or `proxy_request_buffering off` in nginx. |

---

## 14. Quick Checklist

- [ ] SSD formatted (XFS/ext4), auto-mounted, owned by 1000:1000
- [ ] MinIO running with a root password of 8+ characters
- [ ] Bucket `nexus-blobs` and a dedicated `nexus` user created
- [ ] `/opt/latest-nexus` owned by 200:200 and Nexus running
- [ ] S3 blob store `minio` created (path-style enabled)
- [ ] Docker Bearer Token Realm active
- [ ] `docker-hosted` (9007), `docker-proxy` (old Nexus 9009), `docker-group` (9006) created, all on the `minio` blob store
- [ ] Group members ordered: hosted first, proxy second
- [ ] nginx with `ssl`, certificate covering both hostnames, DNS records in place
- [ ] Push, pull and old-only-image pull tested
- [ ] Bulk copy done and counts compared
- [ ] Clients switched, proxy removed from the group, old Nexus retired
- [ ] Backups configured for `/opt/latest-nexus` and MinIO data
