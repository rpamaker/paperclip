# Paperclip en DigitalOcean Droplet

## Infraestructura

| Servicio | Detalle |
|---|---|
| Droplet | Ubuntu 24.04, 4GB RAM / 2 vCPU, sfo3 |
| IP | `64.23.131.110` |
| Dominio | `agents.pometrix.ai` |
| Docker Compose | `docker/docker-compose.quickstart.yml` |
| Data dir | `/opt/paperclip/data/docker-paperclip` |

---

## 1. Crear el Droplet

- OS: **Ubuntu 24.04**
- Plan: **Basic, 4GB RAM / 2 vCPU** (~$24/mes)
- Región: sfo3
- SSH key: agregar public key desde `~/.ssh/id_ed25519.pub`

---

## 2. Instalar Docker

```bash
ssh root@<IP>
curl -fsSL https://get.docker.com | sh
apt install -y unattended-upgrades
```

---

## 3. Clonar el repo

```bash
git clone https://github.com/rpamaker/paperclip.git /opt/paperclip
cd /opt/paperclip
```

---

## 4. Configurar variables de entorno

```bash
cat > /opt/paperclip/docker/.env <<EOF
BETTER_AUTH_SECRET=$(openssl rand -hex 32)
PAPERCLIP_PUBLIC_URL=https://agents.pometrix.ai
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_DB_BACKUP_ENABLED=false
EOF
```

---

## 5. Crear directorios y permisos

```bash
mkdir -p /opt/paperclip/data/docker-paperclip
chmod 777 /opt/paperclip/data/docker-paperclip
```

---

## 6. Levantar Paperclip

```bash
docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env up -d
```

---

## 7. Crear config.json

```bash
mkdir -p /opt/paperclip/data/docker-paperclip/instances/default
cat > /opt/paperclip/data/docker-paperclip/instances/default/config.json <<'EOF'
{
  "$meta": { "version": 1, "updatedAt": "2026-01-01T00:00:00.000Z", "source": "onboard" },
  "server": { "deploymentMode": "authenticated", "exposure": "public", "host": "0.0.0.0", "port": 3100 },
  "database": { "mode": "embedded-postgres" },
  "logging": { "mode": "file", "logDir": "/paperclip/instances/default/logs" },
  "auth": { "baseUrlMode": "explicit", "publicBaseUrl": "https://agents.pometrix.ai" },
  "secrets": { "provider": "local_encrypted", "strictMode": false },
  "telemetry": { "enabled": false }
}
EOF
chmod -R 777 /opt/paperclip/data/docker-paperclip/instances/default
chmod 700 /opt/paperclip/data/docker-paperclip/instances/default/db 2>/dev/null || true
```

---

## 8. Reiniciar y bootstrap del primer admin

```bash
docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env up -d --force-recreate

sleep 5 && docker exec -it docker-paperclip-1 node --import ./server/node_modules/tsx/dist/loader.mjs cli/src/index.ts auth bootstrap-ceo --base-url https://agents.pometrix.ai
```

Abrí la URL que devuelve en el browser para crear tu cuenta de admin.

---

## 9. SSL con Caddy

```bash
apt install -y caddy

cat > /etc/caddy/Caddyfile <<EOF
agents.pometrix.ai {
    reverse_proxy localhost:3100
}
EOF

systemctl reload caddy
```

Actualizá también el `PAPERCLIP_PUBLIC_URL` y el `config.json` si cambiás de IP a dominio:

```bash
sed -i 's|http://64.23.131.110:3100|https://agents.pometrix.ai|g' /opt/paperclip/data/docker-paperclip/instances/default/config.json
sed -i 's|PAPERCLIP_PUBLIC_URL=.*|PAPERCLIP_PUBLIC_URL=https://agents.pometrix.ai|' /opt/paperclip/docker/.env
docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env up -d --force-recreate
```

---

## 10. Autenticación OAuth Codex (una sola vez)

Las credenciales de Codex se guardan en el volumen y persisten entre deploys.

Desde tu **Mac**:

```bash
# Crear directorio en el Droplet
ssh root@64.23.131.110 "mkdir -p /opt/paperclip/data/docker-paperclip/.codex"

# Copiar auth.json desde tu Mac al Droplet
scp ~/.codex/auth.json root@64.23.131.110:/opt/paperclip/data/docker-paperclip/.codex/auth.json

# Corregir permisos
ssh root@64.23.131.110 "chown 1000:1000 /opt/paperclip/data/docker-paperclip/.codex/auth.json"
```

> El token de Codex se renueva automáticamente. Solo necesitás repetir este paso si expira la sesión.

---

## 11. Deploy automático con GitHub Actions

Creá `.github/workflows/deploy.yml` en el repo:

```yaml
name: Deploy

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Droplet
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DROPLET_IP }}
          username: root
          key: ${{ secrets.DROPLET_SSH_KEY }}
          script: |
            cd /opt/paperclip
            git pull origin master
            docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env build --pull
            docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env up -d
```

Agregá en GitHub → Settings → Secrets:
- `DROPLET_IP` → `64.23.131.110`
- `DROPLET_SSH_KEY` → contenido de `~/.ssh/id_ed25519`

---

## Mantenimiento

| Tarea | Comando |
|---|---|
| Ver logs | `docker compose -f docker/docker-compose.quickstart.yml logs -f` |
| Reiniciar | `docker compose -f docker/docker-compose.quickstart.yml --env-file docker/.env up -d --force-recreate` |
| Detener | `docker compose -f docker/docker-compose.quickstart.yml down` |
| Ver estado | `docker ps` |

---

## Arquitectura de storage

| Qué | Dónde |
|---|---|
| Issues, usuarios, tasks, comments | PostgreSQL embebido en `/paperclip/instances/default/db` |
| Archivos adjuntos, assets | `/paperclip/instances/default/storage` |
| Config, secrets, credenciales OAuth | `/opt/paperclip/data/docker-paperclip` (bind mount) |
