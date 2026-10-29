# KR2 variant 2

Student: Maxim Bairashny. Domain: notes.devops.local. Application port: 8002.

## Template defects

Line numbers refer to the original Dockerfile in template main.

| Location | Defect | Consequence | Correction |
|---|---|---|---|
| Dockerfile:1 | python:latest | Base version changes between builds | Reuse python:3.12-slim from PR7 |
| Dockerfile:5 | COPY . . without .dockerignore | Tracked .env enters image layers | Exclude .env and copy only required files |
| Dockerfile:5-6 | Code copied before dependency installation | Code changes reinstall dependencies | Copy requirements, install, then copy app.py |
| Dockerfile:1-9 | No USER | Application runs as root | Create and use app user |
| Dockerfile:9 | Shell-form CMD | Shell may not forward termination signals | Exec-form CMD |
| Dockerfile:9 | Bind 127.0.0.1 inside container | Published port cannot reach app | Bind 0.0.0.0:8002 |
| Dockerfile:8-9 | Port 8000 | Wrong variant port | Use 8002 |
| Tracked .env | Secret published in Git | Template password is disclosed | Remove tracked file, ignore it, generate a new server secret |

## Deployment

Application logic and requirements are unchanged. Only the required identifying comment was added.

Deliver with rsync --delete --exclude=.env --exclude=.git, first using --dry-run. Server path: /home/devops/kr2. Generate REDIS_PASSWORD on the server using openssl rand -hex 16 and save only to .env (owner devops, mode 600). Do not export REDIS_PASSWORD in the shell. Start with docker compose up -d --build.

Redis requires authentication and uses AOF persistence in kr2_redis_data. Redis has no published port; the app is published only on 127.0.0.1:8002. Do not use down -v. Nginx configuration is nginx-kr2.conf, installed as /etc/nginx/sites-available/kr2. Use a separate certificate with SAN DNS:notes.devops.local. Windows hosts needs 192.168.56.101 notes.devops.local.

## Validation

- app UID 999, exec-form CMD, /health responds ok; container stop took 0.660 seconds.
- Rebuild: user, WORKDIR, requirements copy and install CACHED; app.py copy reran.
- Redis healthy; both process UIDs nonzero; unauthenticated PING returns NOAUTH.
- Counter before down: 4; after recreation: 5. Named volume retained.
- HTTP returns 301; HTTPS verifies with kr2.crt; SAN matches notes.devops.local.
- /whoami: scheme https, client 192.168.56.1, host notes.devops.local. Client address is forwarded by Nginx, because the application's TCP peer is the proxy.
- Original static site and PR7 app respond; original Nginx checksum and UFW rules unchanged.
- Extended audit passes 19 checks. Deliberate external publication causes exit 1; restoration returns exit 0.

Template password in inherited main history is not the new server password. The KR2 pull request must remain unmerged until the defense.
