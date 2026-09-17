# n8n — Deploy & Update Guide

Instancia dockerizada de n8n en `https://n8n.ewcam.co`, compartiendo VPS con Taiga.

---

## Versionamiento

Versionado semántico (SemVer). Versión actual en `VERSION`. Historial de cambios por versión en `docs/changelog/<version>/CHANGELOG.md` (formato Keep a Changelog).

- **MAJOR**: cambios incompatibles (ej. estructura de workflow, formato de .env).
- **MINOR**: nueva funcionalidad compatible (ej. nuevo proyecto/canal en un workflow).
- **PATCH**: fixes sin nueva funcionalidad.

---

## Deploy inicial

### 1. Prerrequisitos en el VPS
- Docker + Docker Compose instalados
- Nginx instalado y corriendo (ya gestiona Taiga)
- Certbot instalado
- Puerto 80/443 abierto, 5678 **cerrado** (nginx hace el proxy)

### 2. Copiar archivos al VPS

```bash
scp -r /ruta/local/n8n/ user@VPS_IP:/opt/n8n
```

### 3. Crear `.env` en el VPS

**No copies el `.env` local.** Crealo nuevo en `/opt/n8n/.env`:

```env
N8N_HOST=n8n.ewcam.co
N8N_PROTOCOL=https
N8N_WEBHOOK_URL=https://n8n.ewcam.co/

N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=PASSWORD_FUERTE

# Generá con: openssl rand -hex 32
# NUNCA cambiar después de crear workflows/credenciales
N8N_ENCRYPTION_KEY=GENERAR_CON_OPENSSL

TIMEZONE=America/Bogota
```

### 4. Permisos del volumen

```bash
cd /opt/n8n
mkdir -p data
sudo chown -R 1000:1000 data
```

### 5. Nginx

Crear `/etc/nginx/sites-available/n8n`:

```nginx
server {
    listen 80;
    server_name n8n.ewcam.co;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name n8n.ewcam.co;

    ssl_certificate /etc/letsencrypt/live/n8n.ewcam.co/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.ewcam.co/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/n8n
sudo nginx -t && sudo systemctl reload nginx
```

### 6. SSL

```bash
sudo certbot --nginx -d n8n.ewcam.co
```

### 7. Levantar

```bash
cd /opt/n8n
docker compose up -d
```

### 8. Verificar

```bash
docker compose ps
curl https://n8n.ewcam.co/healthz
# respuesta esperada: {"status":"ok"}
```

---

## Actualizar n8n

```bash
cd /opt/n8n
docker compose pull
docker compose down && docker compose up -d
```

> `restart: unless-stopped` garantiza que el contenedor vuelve solo tras reboot del VPS.

---

## Backup

Los datos viven en `./data/`. Respaldar esa carpeta es suficiente.

```bash
# desde el VPS
tar -czf n8n-backup-$(date +%Y%m%d).tar.gz /opt/n8n/data /opt/n8n/.env
```

## Restore

```bash
cd /opt/n8n
docker compose down
tar -xzf n8n-backup-YYYYMMDD.tar.gz -C /
sudo chown -R 1000:1000 data
docker compose up -d
```

---

## Workflow: Taiga → Discord

Notifica eventos de Taiga (crear, editar, eliminar, comentar) al canal de Discord correspondiente. Enruta primero por el atributo personalizado **DEPARTAMENTO** (si el objeto lo trae) y si no, por el nombre del proyecto.

### Flujo

```
Taiga (webhook POST) → n8n (Verify Signature) → n8n (Build Embed) → Discord (webhook POST)
```

### 0. Habilitar módulo `crypto` en el Code node

El nodo **Verify Taiga Signature** usa `require('crypto')` para validar la firma HMAC del webhook. Ya está en `docker-compose.yml` (`NODE_FUNCTION_ALLOW_BUILTIN=crypto`) — solo falta aplicar:

```bash
docker compose down && docker compose up -d
```

### 1. Configurar webhooks de Discord

En el nodo **Build Discord Embed**, reemplazar cada `__WEBHOOK-URL__` con el webhook real del canal de Discord (por proyecto y/o por departamento — ver `DISCORD_WEBHOOKS` y `DEPARTMENT_WEBHOOKS` en el código del nodo):

```js
const DISCORD_WEBHOOKS = {
  'Nuevo SIGW':     'https://discord.com/api/webhooks/ID/TOKEN',
  'Soporte':        'https://discord.com/api/webhooks/ID/TOKEN',
  'Desarrollo':     'https://discord.com/api/webhooks/ID/TOKEN',
  'Redna Models':   'https://discord.com/api/webhooks/ID/TOKEN',
  'GESTIÓN HUMANA': 'https://discord.com/api/webhooks/ID/TOKEN',
  'Contabilidad':   'https://discord.com/api/webhooks/ID/TOKEN',
  'Tienda Webcam':  'https://discord.com/api/webhooks/ID/TOKEN',
  'Tesoreria':      'https://discord.com/api/webhooks/ID/TOKEN',
  'Fotografia':     'https://discord.com/api/webhooks/ID/TOKEN',
  'Asesores':       'https://discord.com/api/webhooks/ID/TOKEN',
  'Marketing':      'https://discord.com/api/webhooks/ID/TOKEN',
  '_default':       'https://discord.com/api/webhooks/ID/TOKEN',
};
```

> `_default` recibe eventos de proyectos/departamentos sin canal propio.

**Obtener un webhook de Discord:** Canal → Editar canal → Integraciones → Webhooks → Crear webhook → Copiar URL.

### 2. Configurar Taiga

En cada proyecto de Taiga: **Ajustes → Integraciones → Webhooks → Añadir webhook**

- **URL:** `https://n8n.ewcam.co/webhook/taiga`
- **Secret:** definir una key (NO dejar vacío) — copiarla en `TAIGA_WEBHOOK_SECRET` del nodo **Verify Taiga Signature**
- Activar todos los eventos

> El nombre del proyecto en Taiga debe coincidir exactamente con las claves del objeto `DISCORD_WEBHOOKS`. El atributo personalizado **DEPARTAMENTO** (si existe en el proyecto) debe coincidir con las claves de `DEPARTMENT_WEBHOOKS`.

### 3. Importar el workflow en n8n

1. n8n UI → **Workflows → Import from file**
2. Seleccionar `workflows/taiga-discord.json`
3. Nodo **Verify Taiga Signature**: reemplazar `__TAIGA_WEBHOOK_SECRET__` con la key configurada en Taiga
4. Nodo **Build Discord Embed**: reemplazar los `__WEBHOOK-URL__` con las URLs reales
5. Activar el workflow (toggle en la esquina superior derecha)

### 4. Verificar

Crear un item en Taiga → debe aparecer un embed en el canal de Discord correspondiente. Si la firma no coincide, la ejecución falla en **Verify Taiga Signature** (revisar en **Executions**) y no llega nada a Discord.

---

## Comandos útiles

```bash
# Logs en vivo
docker compose logs -f n8n

# Reiniciar
docker compose restart n8n

# Estado
docker compose ps
```
