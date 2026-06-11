# Guía corporativa para servidor Linux seguro con PostgreSQL en Docker, monitoreo y backups

Documento técnico para instalar desde cero una plataforma PostgreSQL profesional sobre Debian usando Docker y Prometheus en el servidor de base de datos, Grafana en un servidor separado, backups automáticos y buenas prácticas de seguridad.

> **Sistema base recomendado:** Debian 12 Bookworm o Debian 13 Trixie x86_64.
> **Usuario ejemplo:** `opsadmin`.
> **Servidor PostgreSQL/Prometheus ejemplo:** `203.0.113.10` público, `10.10.10.10` privado/VPN.
> **Servidor Grafana ejemplo:** `203.0.113.20` público, `10.10.10.20` privado/VPN.
> **Directorio base:** `/opt/postgres-platform`.
> **Importante:** todo valor marcado como `CAMBIAR_CLIENTE` debe reemplazarse antes de producción.

---

## 1. Preparación inicial del servidor Linux

### 1.1 Actualizar el sistema

**Copiar y pegar:**

```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y autoremove
sudo reboot
```

Validación después del reinicio:

```bash
uname -a
lsb_release -a
```

### 1.2 Crear usuario administrador seguro

**Copiar y pegar:**

```bash
sudo adduser opsadmin
sudo usermod -aG sudo opsadmin
id opsadmin
```

Configurar acceso por llave SSH para el nuevo usuario desde el equipo local:

```bash
ssh-copy-id opsadmin@203.0.113.10
```

Validar acceso:

```bash
ssh opsadmin@203.0.113.10
sudo whoami
```

Debe responder:

```text
root
```

### 1.3 Configurar zona horaria

**Copiar y pegar:**

```bash
sudo timedatectl set-timezone America/Guatemala
timedatectl
```

### 1.4 Instalar herramientas base

**Copiar y pegar:**

```bash
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  software-properties-common \
  apt-transport-https \
  vim \
  nano \
  htop \
  iotop \
  iftop \
  unzip \
  zip \
  jq \
  git \
  tree \
  net-tools \
  dnsutils \
  chrony \
  ufw \
  fail2ban
```

Validación:

```bash
chronyc tracking
which curl jq git tree ufw fail2ban-client
```

### 1.5 Configuración segura de SSH

Antes de editar SSH, mantenga abierta una sesión actual y abra una segunda sesión de prueba. No cierre la sesión actual hasta validar que puede entrar con el nuevo usuario.

Crear respaldo:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F-%H%M)
```

Editar archivo:

```bash
sudo nano /etc/ssh/sshd_config
```

Asegurar estos valores:

```text
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
X11Forwarding no
AllowUsers opsadmin
ClientAliveInterval 300
ClientAliveCountMax 2
MaxAuthTries 3
```

Validar sintaxis y reiniciar SSH:

```bash
sudo sshd -t
sudo systemctl reload ssh
sudo systemctl status ssh --no-pager
```

Validar desde otra terminal:

```bash
ssh opsadmin@203.0.113.10
```

Buenas prácticas iniciales:

- Usar llaves SSH, no contraseñas.
- No permitir login directo como `root`.
- No compartir usuarios entre administradores.
- Usar `sudo` con trazabilidad.
- Mantener el sistema actualizado.
- Documentar quién tiene acceso y desde qué IP.
- Activar backups antes de cargar datos reales.

---

## 2. Seguridad y firewall

### 2.1 Puertos usados por la plataforma

| Puerto | Servicio | Exposición recomendada | Comentario |
|---:|---|---|---|
| 22 | SSH | Solo IP administrativa o VPN | Administración del servidor |
| 5432 | PostgreSQL | No público; solo app/VPN/IP whitelist | Base de datos |
| 3000 | Grafana | Solo en servidor Grafana; VPN, proxy HTTPS o IP whitelist | Dashboards |
| 19090 | Prometheus publicado en host | Solo red privada/VPN desde servidor Grafana | Métricas internas; se usa `19090` para evitar conflicto con Prometheus local en `9090` |
| 9090 | Prometheus dentro del contenedor | No se expone directamente | Puerto interno del contenedor |
| 9100 | Node Exporter | No público | Métricas del servidor, solo red Docker interna |
| 9187 | PostgreSQL Exporter | No público | Métricas PostgreSQL, solo red Docker interna |
| 5050 | pgAdmin opcional | VPN, proxy HTTPS o IP whitelist | Administración visual |

> **No exponer públicamente:** `9090`, `9100`, `9187`, `19090`.
> **Evitar exponer públicamente:** `5432`, `5050`, `3000`. Usar VPN, IP whitelist o reverse proxy con HTTPS y autenticación.

### 2.2 Configurar UFW

Ejemplo con IP administrativa `198.51.100.20`. Cambiar por la IP pública real del administrador o rango VPN.

**Copiar y pegar:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH solo desde IP administrativa
sudo ufw allow from 198.51.100.20 to any port 22 proto tcp comment 'SSH admin'

# PostgreSQL solo desde servidor de aplicación o VPN
sudo ufw allow from 198.51.100.30 to any port 5432 proto tcp comment 'PostgreSQL app server'

# Prometheus solo desde el servidor Grafana por red privada/VPN
sudo ufw allow from 10.10.10.20 to any port 19090 proto tcp comment 'Prometheus para Grafana'

# pgAdmin opcional, solo administración
sudo ufw allow from 198.51.100.20 to any port 5050 proto tcp comment 'pgAdmin admin'

sudo ufw enable
sudo ufw status verbose
```

En el servidor Grafana separado, permitir el puerto `3000` solo a administradores o al reverse proxy:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 198.51.100.20 to any port 22 proto tcp comment 'SSH admin Grafana'
sudo ufw allow from 198.51.100.20 to any port 3000 proto tcp comment 'Grafana admin'
sudo ufw enable
sudo ufw status verbose
```

Si todavía no conoce las IP autorizadas, use temporalmente SSH abierto y cierre después:

```bash
sudo ufw allow 22/tcp comment 'SSH temporal'
```

Luego reemplace por whitelist:

```bash
sudo ufw delete allow 22/tcp
sudo ufw allow from 198.51.100.20 to any port 22 proto tcp
```

### 2.3 Nota importante sobre Docker y UFW

Docker manipula reglas `iptables` y los puertos publicados con `ports:` pueden quedar accesibles aunque UFW parezca bloquearlos. Para producción:

- Publique servicios administrativos solo en `127.0.0.1` si se accederán por túnel SSH o reverse proxy.
- No publique exporters hacia Internet.
- Use la cadena `DOCKER-USER` para controles estrictos si necesita filtrar tráfico Docker a nivel host.
- Prefiera VPN privada para PostgreSQL, Prometheus, exporters, pgAdmin y la comunicación Grafana -> Prometheus.

### 2.4 Instalar y configurar Fail2Ban

Crear configuración local:

```bash
sudo nano /etc/fail2ban/jail.local
```

**Copiar y pegar:**

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
backend = systemd
destemail = admin@cliente.com
sender = fail2ban@cliente.com
action = %(action_)s

[sshd]
enabled = true
port = 22
filter = sshd
logpath = %(sshd_log)s
maxretry = 3
bantime = 24h
```

Reiniciar y validar:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Recomendaciones para producción:

- Activar MFA donde aplique: VPN, panel cloud, repositorios y cuentas administrativas.
- Usar VPN tipo WireGuard, Tailscale, Cloudflare Zero Trust o red privada del proveedor.
- Permitir PostgreSQL solo desde servidores de aplicación autorizados.
- Usar contraseñas largas, únicas y almacenadas en gestor de secretos.
- Rotar credenciales al cambio de personal.
- Configurar alertas de disco, CPU, memoria y fallos de backup.

---

## 3. Instalación de Docker y Docker Compose en Debian

Las instrucciones siguen el método recomendado por Docker para Debian: repositorio oficial `apt` y plugin moderno `docker compose`.

### 3.1 Instalar Docker Engine en Debian

**Copiar y pegar en el servidor PostgreSQL/Prometheus:**

```bash
# Confirmar Debian soportado: Debian 12 Bookworm o Debian 13 Trixie recomendado.
. /etc/os-release
echo "$PRETTY_NAME"

# Remover paquetes conflictivos si existen.
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y "$pkg" 2>/dev/null || true
done

# Instalar dependencias.
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Agregar llave GPG oficial de Docker para Debian.
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Agregar repositorio oficial de Docker para Debian.
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine y Compose moderno.
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.2 Activar Docker al iniciar

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

### 3.3 Agregar usuario al grupo Docker

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

> Seguridad: pertenecer al grupo `docker` equivale prácticamente a privilegios de root. Solo usuarios administradores deben pertenecer a este grupo.

### 3.4 Validar instalación

```bash
docker version
docker compose version
docker run --rm hello-world
```

---

## 4. Estructura corporativa del proyecto

Crear estructura:

```bash
sudo mkdir -p /opt/postgres-platform/{backups,postgres/init,prometheus,scripts,docs,logs}
sudo chown -R "$USER":"$USER" /opt/postgres-platform
chmod 750 /opt/postgres-platform
cd /opt/postgres-platform
```

Estructura esperada:

```text
Servidor PostgreSQL/Prometheus:
/opt/postgres-platform/
├── docker-compose.yml
├── .env
├── backups/
├── postgres/
│   └── init/
├── prometheus/
│   └── prometheus.yml
├── scripts/
├── logs/
└── docs/
```

Uso de carpetas:

| Ruta | Uso |
|---|---|
| `/opt/postgres-platform/docker-compose.yml` | Definición de servicios Docker |
| `/opt/postgres-platform/.env` | Variables sensibles y parámetros del stack |
| `/opt/postgres-platform/backups/` | Backups comprimidos de PostgreSQL |
| `/opt/postgres-platform/postgres/init/` | Scripts SQL opcionales de inicialización |
| `/opt/postgres-platform/prometheus/` | Configuración de Prometheus |
| `/opt/postgres-platform/scripts/` | Scripts operativos de backup/restauración |
| `/opt/postgres-platform/logs/` | Logs generados por scripts |
| `/opt/postgres-platform/docs/` | Documentación interna del cliente |

---

## 5. Archivo `.env`

Crear archivo:

```bash
nano /opt/postgres-platform/.env
```

**Copiar y pegar:**

```dotenv
###############################################################################
# Plataforma PostgreSQL Docker
# Cambiar todos los valores CAMBIAR_CLIENTE antes de producción.
###############################################################################

# Zona horaria
TZ=America/Guatemala

# Proyecto
COMPOSE_PROJECT_NAME=postgres-platform

###############################################################################
# PostgreSQL
###############################################################################
POSTGRES_IMAGE=postgres:16-alpine
POSTGRES_CONTAINER_NAME=postgres-db
POSTGRES_HOST_PORT=5432
POSTGRES_INTERNAL_PORT=5432
POSTGRES_DB=cliente_app_prod
POSTGRES_USER=app_owner
POSTGRES_PASSWORD=CAMBIAR_CLIENTE_UseUnaClaveLarga_32Chars_Min

# Usuario de solo lectura para reportes o BI, crear manualmente si se necesita
POSTGRES_READONLY_USER=app_readonly
POSTGRES_READONLY_PASSWORD=CAMBIAR_CLIENTE_ReadOnly_ClaveLarga

# Ruta de datos persistentes
POSTGRES_DATA_VOLUME=postgres_data

###############################################################################
# PostgreSQL Exporter
###############################################################################
POSTGRES_EXPORTER_IMAGE=prometheuscommunity/postgres-exporter:v0.15.0
POSTGRES_EXPORTER_CONTAINER_NAME=postgres-exporter
POSTGRES_EXPORTER_PORT=9187

###############################################################################
# Prometheus en servidor PostgreSQL
###############################################################################
PROMETHEUS_IMAGE=prom/prometheus:v2.54.1
PROMETHEUS_CONTAINER_NAME=postgres-platform-prometheus

# Se publica en 19090 para evitar conflicto con instalaciones Prometheus existentes en 9090.
# Usar IP privada/VPN del servidor PostgreSQL. Ejemplo: 10.10.10.10.
# Si Grafana NO está en otro servidor, puede usarse 127.0.0.1.
PROMETHEUS_BIND_ADDRESS=10.10.10.10
PROMETHEUS_HOST_PORT=19090
PROMETHEUS_INTERNAL_PORT=9090
PROMETHEUS_RETENTION_TIME=30d
PROMETHEUS_RETENTION_SIZE=10GB


###############################################################################
# pgAdmin opcional
###############################################################################
PGADMIN_ENABLED=true
PGADMIN_IMAGE=dpage/pgadmin4:8.12
PGADMIN_CONTAINER_NAME=pgadmin
PGADMIN_HOST_PORT=5050
PGADMIN_DEFAULT_EMAIL=admin@cliente.com
PGADMIN_DEFAULT_PASSWORD=CAMBIAR_CLIENTE_pgAdmin_ClaveLarga

###############################################################################
# Backups
###############################################################################
BACKUP_DIR=/opt/postgres-platform/backups
BACKUP_RETENTION_DAYS=14
BACKUP_LOG_DIR=/opt/postgres-platform/logs
BACKUP_COMPRESSION_LEVEL=6

###############################################################################
# Seguridad operativa
###############################################################################
# Si PostgreSQL no debe exponerse externamente, cambiar en docker-compose.yml:
# "5432:5432" por "127.0.0.1:5432:5432" o eliminar ports y usar solo red interna.
# Prometheus debe quedar accesible solo desde el servidor Grafana por IP privada/VPN.
```

Proteger archivo:

```bash
chmod 600 /opt/postgres-platform/.env
```

Validar:

```bash
grep -n 'CAMBIAR_CLIENTE' /opt/postgres-platform/.env
```

El resultado debe mostrar valores pendientes. Antes de producción, no debe quedar ningún `CAMBIAR_CLIENTE`.

---

## 6. Docker Compose completo para servidor PostgreSQL/Prometheus

Este `docker-compose.yml` se ejecuta en el servidor Debian de base de datos. No incluye Grafana, porque Grafana se instalará en otro servidor.

Crear archivo:

```bash
nano /opt/postgres-platform/docker-compose.yml
```

**Copiar y pegar:**

```yaml
services:
  postgres:
    image: ${POSTGRES_IMAGE}
    container_name: ${POSTGRES_CONTAINER_NAME}
    restart: unless-stopped
    environment:
      TZ: ${TZ}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      # Producción recomendada: restringir por firewall, VPN o publicar en 127.0.0.1.
      - "${POSTGRES_HOST_PORT}:${POSTGRES_INTERNAL_PORT}"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    networks:
      - db_internal
      - monitoring
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
    security_opt:
      - no-new-privileges:true

  postgres-exporter:
    image: ${POSTGRES_EXPORTER_IMAGE}
    container_name: ${POSTGRES_EXPORTER_CONTAINER_NAME}
    restart: unless-stopped
    environment:
      TZ: ${TZ}
      DATA_SOURCE_NAME: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"
    expose:
      - "${POSTGRES_EXPORTER_PORT}"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - monitoring
      - db_internal
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "3"
    security_opt:
      - no-new-privileges:true

  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: node-exporter
    restart: unless-stopped
    command:
      - "--path.rootfs=/host"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    pid: host
    volumes:
      - /:/host:ro,rslave
    expose:
      - "9100"
    networks:
      - monitoring
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "3"
    security_opt:
      - no-new-privileges:true

  prometheus:
    image: ${PROMETHEUS_IMAGE}
    container_name: ${PROMETHEUS_CONTAINER_NAME}
    restart: unless-stopped
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=${PROMETHEUS_RETENTION_TIME}"
      - "--storage.tsdb.retention.size=${PROMETHEUS_RETENTION_SIZE}"
      - "--web.enable-lifecycle"
    ports:
      # Host 19090 -> contenedor 9090 evita conflicto con Prometheus del host en 9090.
      # PROMETHEUS_BIND_ADDRESS debe ser IP privada/VPN, no 0.0.0.0 en producción.
      - "${PROMETHEUS_BIND_ADDRESS}:${PROMETHEUS_HOST_PORT}:${PROMETHEUS_INTERNAL_PORT}"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    depends_on:
      - node-exporter
      - postgres-exporter
    networks:
      - monitoring
    healthcheck:
      test: ["CMD", "promtool", "check", "config", "/etc/prometheus/prometheus.yml"]
      interval: 30s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
    security_opt:
      - no-new-privileges:true

  pgadmin:
    image: ${PGADMIN_IMAGE}
    container_name: ${PGADMIN_CONTAINER_NAME}
    restart: unless-stopped
    profiles:
      - admin
    environment:
      TZ: ${TZ}
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
      PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION: "True"
      PGADMIN_CONFIG_CONSOLE_LOG_LEVEL: "30"
    ports:
      - "${PGADMIN_HOST_PORT}:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - db_internal
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
    security_opt:
      - no-new-privileges:true

networks:
  db_internal:
    driver: bridge
  monitoring:
    driver: bridge

volumes:
  postgres_data:
    name: ${POSTGRES_DATA_VOLUME}
  prometheus_data:
  pgadmin_data:
```

Validar sintaxis:

```bash
cd /opt/postgres-platform
docker compose config
```

> pgAdmin está bajo perfil `admin`. Para levantar pgAdmin use `docker compose --profile admin up -d`.

> Si `docker compose up -d` falla por bind de Prometheus, revise que `PROMETHEUS_BIND_ADDRESS` exista en el servidor con `ip addr` y que `PROMETHEUS_HOST_PORT` no esté ocupado.

---

## 7. Configuración de PostgreSQL

### 7.1 Levantar servicios principales

```bash
cd /opt/postgres-platform
docker compose up -d
docker compose ps
```

Con pgAdmin:

```bash
docker compose --profile admin up -d
```

### 7.2 Validar PostgreSQL

```bash
docker exec -it postgres-db pg_isready -U app_owner -d cliente_app_prod
docker logs --tail=100 postgres-db
```

Conectarse desde terminal:

```bash
docker exec -it postgres-db psql -U app_owner -d cliente_app_prod
```

Comandos útiles dentro de `psql`:

```sql
\l
\dt
\du
SELECT version();
SELECT now();
\q
```

### 7.3 Conexión desde cliente externo

Datos:

```text
Host: IP_DEL_SERVIDOR
Puerto: 5432
Base de datos: cliente_app_prod
Usuario: app_owner
Contraseña: definida en .env
SSL: prefer/requerido si usa proxy o TLS externo
```

Recomendación: no conectar clientes externos por Internet abierto. Usar VPN, red privada, túnel SSH o IP whitelist.

Túnel SSH ejemplo:

```bash
ssh -L 5432:127.0.0.1:5432 opsadmin@203.0.113.10
```

Luego conectar el cliente a:

```text
Host: 127.0.0.1
Puerto: 5432
```

### 7.4 Crear bases, usuarios y permisos

Entrar a PostgreSQL:

```bash
docker exec -it postgres-db psql -U app_owner -d cliente_app_prod
```

Crear base de datos:

```sql
CREATE DATABASE ventas_prod;
```

Crear usuario de aplicación:

```sql
CREATE USER ventas_app WITH PASSWORD 'CAMBIAR_CLIENTE_ClaveLargaVentas';
GRANT CONNECT ON DATABASE ventas_prod TO ventas_app;
```

Asignar permisos sobre esquema:

```sql
\c ventas_prod
GRANT USAGE, CREATE ON SCHEMA public TO ventas_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO ventas_app;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA public TO ventas_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ventas_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO ventas_app;
```

Usuario solo lectura:

```sql
CREATE USER reportes_readonly WITH PASSWORD 'CAMBIAR_CLIENTE_ClaveLargaReadOnly';
GRANT CONNECT ON DATABASE ventas_prod TO reportes_readonly;
\c ventas_prod
GRANT USAGE ON SCHEMA public TO reportes_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reportes_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO reportes_readonly;
```

Buenas prácticas de roles:

- No usar el usuario administrador de PostgreSQL para la aplicación diaria.
- Crear un usuario por aplicación o sistema.
- Crear usuarios separados para lectura, reportes y migraciones.
- No compartir contraseñas entre ambientes.
- Rotar credenciales periódicamente.
- Auditar permisos con `\du` y `information_schema`.

### 7.5 Configuración recomendada para producción

Para producción crítica, ajustar PostgreSQL según RAM, CPU y patrón de uso. Como base:

- Activar backups y probar restauración antes de cargar datos reales.
- Configurar `max_connections` de acuerdo con la aplicación.
- Usar pool de conexiones como PgBouncer si hay muchas conexiones.
- Monitorear locks, crecimiento de disco y queries lentas.
- Considerar TLS si PostgreSQL se expone fuera de red local/VPN.
- Configurar mantenimiento: `VACUUM`, índices y análisis periódico.

### 7.6 Si PostgreSQL no inicia

```bash
docker compose ps
docker logs --tail=200 postgres-db
docker inspect postgres-db --format '{{json .State.Health}}' | jq
docker volume ls
df -h
```

Causas comunes:

- Contraseña vacía o mal definida en `.env`.
- Volumen con permisos incorrectos.
- Puerto `5432` ocupado.
- Disco lleno.
- Datos corruptos por apagado abrupto.
- Cambio de versión mayor de PostgreSQL sin migración.

---

## 8. Configuración de Prometheus

Crear archivo:

```bash
nano /opt/postgres-platform/prometheus/prometheus.yml
```

**Copiar y pegar:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  # Prometheus monitoreándose a sí mismo.
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
        labels:
          service: "prometheus"
          environment: "production"

  # Métricas del servidor Linux: CPU, RAM, disco, red, uptime.
  - job_name: "node-exporter"
    scrape_interval: 15s
    static_configs:
      - targets: ["node-exporter:9100"]
        labels:
          service: "linux-server"
          environment: "production"

  # Métricas internas de PostgreSQL.
  - job_name: "postgres-exporter"
    scrape_interval: 15s
    static_configs:
      - targets: ["postgres-exporter:9187"]
        labels:
          service: "postgresql"
          environment: "production"
```

Validar:

```bash
cd /opt/postgres-platform
set -a
source .env
set +a
docker compose restart prometheus
curl -s http://${PROMETHEUS_BIND_ADDRESS}:${PROMETHEUS_HOST_PORT}/-/healthy
```

Ver targets:

```bash
cd /opt/postgres-platform
set -a
source .env
set +a
curl -s http://${PROMETHEUS_BIND_ADDRESS}:${PROMETHEUS_HOST_PORT}/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'
```

Buenas prácticas:

- No exponer Prometheus a Internet.
- Usar retención acorde al disco disponible.
- Separar monitoreo centralizado si hay varios servidores.
- Crear alertas para disco, memoria, caídas de exporters y fallos de backup.

---

## 9. Instalación y configuración de Grafana en servidor separado

Grafana se instala en un segundo servidor Debian. Este servidor consulta Prometheus en el servidor PostgreSQL por red privada/VPN usando `http://10.10.10.10:19090` como ejemplo.

### 9.1 Preparar servidor Grafana

Actualizar Debian e instalar herramientas base:

```bash
sudo apt update
sudo apt -y upgrade
sudo apt install -y ca-certificates curl gnupg apt-transport-https ufw fail2ban nano jq
sudo timedatectl set-timezone America/Guatemala
```

Aplicar firewall mínimo:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 198.51.100.20 to any port 22 proto tcp comment 'SSH admin Grafana'
sudo ufw allow from 198.51.100.20 to any port 3000 proto tcp comment 'Grafana admin'
sudo ufw enable
sudo ufw status verbose
```

Validar conectividad desde el servidor Grafana hacia Prometheus:

```bash
curl -s http://10.10.10.10:19090/-/healthy
curl -s http://10.10.10.10:19090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

### 9.2 Instalar Grafana con repositorio oficial APT

**Copiar y pegar en el servidor Grafana:**

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server --no-pager
```

Validar:

```bash
curl -s http://127.0.0.1:3000/api/health | jq
```

### 9.3 Acceso a Grafana

Abrir desde una IP autorizada:

```text
http://IP_SERVIDOR_GRAFANA:3000
```

Credenciales iniciales por defecto:

```text
Usuario: admin
Contraseña: admin
```

Grafana solicitará cambiar la contraseña en el primer ingreso. Use una contraseña larga y única.

### 9.4 Agregar Prometheus como datasource

Opción UI:

1. Entrar a Grafana.
2. Ir a `Connections` -> `Data sources`.
3. Seleccionar `Prometheus`.
4. URL: `http://10.10.10.10:19090`.
5. Guardar y probar con `Save & test`.

Opción provisioning por archivo:

```bash
sudo mkdir -p /etc/grafana/provisioning/datasources
sudo nano /etc/grafana/provisioning/datasources/prometheus.yml
```

**Copiar y pegar:**

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: Prometheus
    type: prometheus
    access: proxy
    url: http://10.10.10.10:19090
    isDefault: true
    editable: true
```

Reiniciar Grafana:

```bash
sudo systemctl restart grafana-server
sudo systemctl status grafana-server --no-pager
```

### 9.5 Provisionar carpeta local para dashboards

```bash
sudo mkdir -p /var/lib/grafana/dashboards/postgres-platform
sudo chown -R grafana:grafana /var/lib/grafana/dashboards
sudo nano /etc/grafana/provisioning/dashboards/postgres-platform.yml
```

**Copiar y pegar:**

```yaml
apiVersion: 1

providers:
  - name: "PostgreSQL Platform"
    orgId: 1
    folder: "PostgreSQL Platform"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/postgres-platform
```

El dashboard JSON de la sección siguiente debe guardarse en:

```text
/var/lib/grafana/dashboards/postgres-platform/postgresql-platform-overview.json
```

Recomendaciones:

- No instalar Grafana dentro del compose del servidor PostgreSQL.
- Proteger Grafana con VPN, IP whitelist o reverse proxy HTTPS.
- Deshabilitar usuarios anónimos.
- Crear usuarios nominales, no compartidos.
- Organizar dashboards por carpetas: `Infraestructura`, `PostgreSQL`, `Backups`, `Aplicaciones`.

---

## 10. Dashboard de Grafana

Crear archivo:

```bash
sudo nano /var/lib/grafana/dashboards/postgres-platform/postgresql-platform-overview.json
```

**Copiar y pegar:**

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {"type": "grafana", "uid": "-- Grafana --"},
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "percent", "thresholds": {"mode": "absolute", "steps": [{"color": "green"}, {"color": "orange", "value": 70}, {"color": "red", "value": 90}]}}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 0, "y": 0},
      "id": 1,
      "options": {"reduceOptions": {"calcs": ["lastNotNull"], "fields": "", "values": false}, "showThresholdLabels": false, "showThresholdMarkers": true},
      "targets": [{"expr": "100 - (avg by(instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)", "legendFormat": "CPU", "refId": "A"}],
      "title": "Uso de CPU",
      "type": "gauge"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "percent", "thresholds": {"mode": "absolute", "steps": [{"color": "green"}, {"color": "orange", "value": 75}, {"color": "red", "value": 90}]}}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 6, "y": 0},
      "id": 2,
      "options": {"reduceOptions": {"calcs": ["lastNotNull"], "fields": "", "values": false}, "showThresholdLabels": false, "showThresholdMarkers": true},
      "targets": [{"expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100", "legendFormat": "RAM", "refId": "A"}],
      "title": "Uso de memoria RAM",
      "type": "gauge"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "percent", "thresholds": {"mode": "absolute", "steps": [{"color": "green"}, {"color": "orange", "value": 75}, {"color": "red", "value": 90}]}}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 12, "y": 0},
      "id": 3,
      "options": {"reduceOptions": {"calcs": ["lastNotNull"], "fields": "", "values": false}, "showThresholdLabels": false, "showThresholdMarkers": true},
      "targets": [{"expr": "100 - ((node_filesystem_avail_bytes{fstype!~\"tmpfs|overlay\",mountpoint=\"/\"} * 100) / node_filesystem_size_bytes{fstype!~\"tmpfs|overlay\",mountpoint=\"/\"})", "legendFormat": "Disco /", "refId": "A"}],
      "title": "Uso de disco raíz",
      "type": "gauge"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "s", "thresholds": {"mode": "absolute", "steps": [{"color": "green"}]}}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 18, "y": 0},
      "id": 4,
      "options": {"reduceOptions": {"calcs": ["lastNotNull"], "fields": "", "values": false}},
      "targets": [{"expr": "time() - node_boot_time_seconds", "legendFormat": "Uptime", "refId": "A"}],
      "title": "Uptime del servidor",
      "type": "stat"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 0, "y": 6},
      "id": 5,
      "targets": [{"expr": "node_load1", "legendFormat": "Load 1m", "refId": "A"}, {"expr": "node_load5", "legendFormat": "Load 5m", "refId": "B"}, {"expr": "node_load15", "legendFormat": "Load 15m", "refId": "C"}],
      "title": "Carga del sistema",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "percent"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 8, "y": 6},
      "id": 6,
      "targets": [{"expr": "(1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)) * 100", "legendFormat": "Swap", "refId": "A"}],
      "title": "Uso de swap",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "Bps"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 16, "y": 6},
      "id": 7,
      "targets": [{"expr": "sum(rate(node_network_receive_bytes_total{device!~\"lo|veth.*|docker.*|br.*\"}[5m]))", "legendFormat": "RX", "refId": "A"}, {"expr": "sum(rate(node_network_transmit_bytes_total{device!~\"lo|veth.*|docker.*|br.*\"}[5m]))", "legendFormat": "TX", "refId": "B"}],
      "title": "Tráfico de red",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 0, "y": 13},
      "id": 8,
      "targets": [{"expr": "node_procs_running", "legendFormat": "Running", "refId": "A"}, {"expr": "node_procs_blocked", "legendFormat": "Blocked", "refId": "B"}],
      "title": "Procesos activos",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 6, "y": 13},
      "id": 9,
      "targets": [{"expr": "pg_up", "legendFormat": "PostgreSQL up", "refId": "A"}],
      "title": "Estado PostgreSQL",
      "type": "stat"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 12, "y": 13},
      "id": 10,
      "targets": [{"expr": "sum(pg_stat_activity_count)", "legendFormat": "Activas", "refId": "A"}],
      "title": "Conexiones activas",
      "type": "stat"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 6, "w": 6, "x": 18, "y": 13},
      "id": 11,
      "targets": [{"expr": "pg_settings_max_connections", "legendFormat": "Máximas", "refId": "A"}],
      "title": "Conexiones máximas",
      "type": "stat"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "ops"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 0, "y": 19},
      "id": 12,
      "targets": [{"expr": "rate(pg_stat_database_xact_commit[5m])", "legendFormat": "commits {{datname}}", "refId": "A"}, {"expr": "rate(pg_stat_database_xact_rollback[5m])", "legendFormat": "rollbacks {{datname}}", "refId": "B"}],
      "title": "Transacciones",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 8, "y": 19},
      "id": 13,
      "targets": [{"expr": "sum(pg_locks_count)", "legendFormat": "Locks", "refId": "A"}],
      "title": "Bloqueos",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "bytes"}, "overrides": []},
      "gridPos": {"h": 7, "w": 8, "x": 16, "y": 19},
      "id": 14,
      "targets": [{"expr": "pg_database_size_bytes", "legendFormat": "{{datname}}", "refId": "A"}],
      "title": "Tamaño de base de datos",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "ops"}, "overrides": []},
      "gridPos": {"h": 7, "w": 12, "x": 0, "y": 26},
      "id": 15,
      "targets": [{"expr": "rate(pg_stat_database_tup_fetched[5m])", "legendFormat": "fetched {{datname}}", "refId": "A"}, {"expr": "rate(pg_stat_database_tup_inserted[5m])", "legendFormat": "inserted {{datname}}", "refId": "B"}, {"expr": "rate(pg_stat_database_tup_updated[5m])", "legendFormat": "updated {{datname}}", "refId": "C"}, {"expr": "rate(pg_stat_database_tup_deleted[5m])", "legendFormat": "deleted {{datname}}", "refId": "D"}],
      "title": "Tuplas leídas/escritas",
      "type": "timeseries"
    },
    {
      "datasource": {"type": "prometheus", "uid": "Prometheus"},
      "fieldConfig": {"defaults": {"unit": "short"}, "overrides": []},
      "gridPos": {"h": 7, "w": 12, "x": 12, "y": 26},
      "id": 16,
      "targets": [{"expr": "sum by (state) (pg_stat_activity_count)", "legendFormat": "{{state}}", "refId": "A"}],
      "title": "Actividad PostgreSQL por estado",
      "type": "timeseries"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 39,
  "tags": ["postgresql", "linux", "prometheus"],
  "templating": {"list": []},
  "time": {"from": "now-6h", "to": "now"},
  "timepicker": {},
  "timezone": "browser",
  "title": "PostgreSQL Platform Overview",
  "uid": "postgres-platform-overview",
  "version": 1,
  "weekStart": ""
}
```

> Consultas lentas: PostgreSQL no expone queries lentas por defecto en estas métricas. Para monitorearlas correctamente, habilitar `pg_stat_statements` y extender el exporter con queries customizadas.

Reiniciar Grafana en el servidor Grafana:

```bash
sudo chown grafana:grafana /var/lib/grafana/dashboards/postgres-platform/postgresql-platform-overview.json
sudo systemctl restart grafana-server
```

---

## 11. Backups automáticos con crontab

### 11.1 Script `backup_postgres.sh`

Crear archivo:

```bash
nano /opt/postgres-platform/scripts/backup_postgres.sh
```

**Copiar y pegar:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

BASE_DIR="/opt/postgres-platform"
ENV_FILE="${BASE_DIR}/.env"

if [[ ! -f "${ENV_FILE}" ]]; then
  echo "ERROR: no existe ${ENV_FILE}" >&2
  exit 1
fi

set -a
source "${ENV_FILE}"
set +a

TIMESTAMP="$(date +%F_%H-%M-%S)"
BACKUP_FILE="${BACKUP_DIR}/${POSTGRES_DB}_${TIMESTAMP}.sql.gz"
LOG_FILE="${BACKUP_LOG_DIR}/backup_postgres.log"
TMP_FILE="${BACKUP_FILE}.tmp"

mkdir -p "${BACKUP_DIR}" "${BACKUP_LOG_DIR}"

log() {
  echo "[$(date '+%F %T')] $*" | tee -a "${LOG_FILE}"
}

cleanup_tmp() {
  rm -f "${TMP_FILE}"
}
trap cleanup_tmp EXIT

log "Iniciando backup de base ${POSTGRES_DB}"

if ! docker ps --format '{{.Names}}' | grep -qx "${POSTGRES_CONTAINER_NAME}"; then
  log "ERROR: contenedor ${POSTGRES_CONTAINER_NAME} no está corriendo"
  exit 1
fi

if ! docker exec "${POSTGRES_CONTAINER_NAME}" pg_isready -U "${POSTGRES_USER}" -d "${POSTGRES_DB}" >/dev/null 2>&1; then
  log "ERROR: PostgreSQL no responde a pg_isready"
  exit 1
fi

docker exec \
  -e PGPASSWORD="${POSTGRES_PASSWORD}" \
  "${POSTGRES_CONTAINER_NAME}" \
  pg_dump -U "${POSTGRES_USER}" -d "${POSTGRES_DB}" --format=plain --no-owner --no-privileges \
  | gzip -"${BACKUP_COMPRESSION_LEVEL:-6}" > "${TMP_FILE}"

if [[ ! -s "${TMP_FILE}" ]]; then
  log "ERROR: backup vacío o no generado"
  exit 1
fi

mv "${TMP_FILE}" "${BACKUP_FILE}"
sha256sum "${BACKUP_FILE}" > "${BACKUP_FILE}.sha256"

log "Backup generado: ${BACKUP_FILE}"
log "Tamaño: $(du -h "${BACKUP_FILE}" | awk '{print $1}')"

find "${BACKUP_DIR}" -type f -name "${POSTGRES_DB}_*.sql.gz" -mtime +"${BACKUP_RETENTION_DAYS}" -print -delete | tee -a "${LOG_FILE}" || true
find "${BACKUP_DIR}" -type f -name "${POSTGRES_DB}_*.sql.gz.sha256" -mtime +"${BACKUP_RETENTION_DAYS}" -print -delete | tee -a "${LOG_FILE}" || true

log "Backup finalizado correctamente"
```

Permisos:

```bash
chmod 750 /opt/postgres-platform/scripts/backup_postgres.sh
```

Ejecutar prueba:

```bash
/opt/postgres-platform/scripts/backup_postgres.sh
ls -lh /opt/postgres-platform/backups
tail -n 50 /opt/postgres-platform/logs/backup_postgres.log
```

### 11.2 Script `restore_postgres.sh`

Crear archivo:

```bash
nano /opt/postgres-platform/scripts/restore_postgres.sh
```

**Copiar y pegar:**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

BASE_DIR="/opt/postgres-platform"
ENV_FILE="${BASE_DIR}/.env"

if [[ $# -ne 1 ]]; then
  echo "Uso: $0 /ruta/backup.sql.gz" >&2
  exit 1
fi

BACKUP_FILE="$1"

if [[ ! -f "${BACKUP_FILE}" ]]; then
  echo "ERROR: no existe backup ${BACKUP_FILE}" >&2
  exit 1
fi

if [[ ! -f "${ENV_FILE}" ]]; then
  echo "ERROR: no existe ${ENV_FILE}" >&2
  exit 1
fi

set -a
source "${ENV_FILE}"
set +a

LOG_FILE="${BACKUP_LOG_DIR}/restore_postgres.log"
mkdir -p "${BACKUP_LOG_DIR}"

log() {
  echo "[$(date '+%F %T')] $*" | tee -a "${LOG_FILE}"
}

log "ADVERTENCIA: se restaurará ${BACKUP_FILE} sobre ${POSTGRES_DB}"
read -r -p "Escriba RESTAURAR para continuar: " CONFIRM

if [[ "${CONFIRM}" != "RESTAURAR" ]]; then
  log "Restauración cancelada"
  exit 1
fi

if [[ -f "${BACKUP_FILE}.sha256" ]]; then
  sha256sum -c "${BACKUP_FILE}.sha256"
fi

docker exec \
  -e PGPASSWORD="${POSTGRES_PASSWORD}" \
  "${POSTGRES_CONTAINER_NAME}" \
  psql -U "${POSTGRES_USER}" -d postgres \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='${POSTGRES_DB}' AND pid <> pg_backend_pid();"

docker exec \
  -e PGPASSWORD="${POSTGRES_PASSWORD}" \
  "${POSTGRES_CONTAINER_NAME}" \
  dropdb -U "${POSTGRES_USER}" --if-exists "${POSTGRES_DB}"

docker exec \
  -e PGPASSWORD="${POSTGRES_PASSWORD}" \
  "${POSTGRES_CONTAINER_NAME}" \
  createdb -U "${POSTGRES_USER}" "${POSTGRES_DB}"

gunzip -c "${BACKUP_FILE}" | docker exec -i \
  -e PGPASSWORD="${POSTGRES_PASSWORD}" \
  "${POSTGRES_CONTAINER_NAME}" \
  psql -U "${POSTGRES_USER}" -d "${POSTGRES_DB}"

log "Restauración finalizada correctamente"
```

Permisos:

```bash
chmod 750 /opt/postgres-platform/scripts/restore_postgres.sh
```

Ejemplo de restauración:

```bash
/opt/postgres-platform/scripts/restore_postgres.sh /opt/postgres-platform/backups/cliente_app_prod_2026-06-04_02-00-00.sql.gz
```

### 11.3 Configurar crontab

Editar crontab del usuario administrador:

```bash
crontab -e
```

**Copiar y pegar:**

```cron
# Backup diario PostgreSQL a las 02:00 hora local.
0 2 * * * /opt/postgres-platform/scripts/backup_postgres.sh >> /opt/postgres-platform/logs/backup_cron.log 2>&1
```

Validar:

```bash
crontab -l
grep CRON /var/log/syslog | tail -n 20
```

Validación de backup exitoso:

```bash
ls -lh /opt/postgres-platform/backups
sha256sum -c /opt/postgres-platform/backups/*.sha256
tail -n 100 /opt/postgres-platform/logs/backup_postgres.log
```

Retención recomendada:

- Cliente pequeño: 14 a 30 días local, 3 a 6 meses externo.
- Cliente mediano: 30 días local, 12 meses externo mensual.
- Producción crítica: backups diarios, semanales, mensuales y pruebas de restauración calendarizadas.

---

## 12. Redundancia

### 12.1 Opciones

| Opción | Complejidad | Recomendación |
|---|---:|---|
| Backups locales | Baja | Obligatorio |
| Backups externos S3/R2/Backblaze/NAS | Media | Muy recomendado |
| Snapshots del VPS | Baja | Recomendado como complemento |
| Volúmenes persistentes Docker | Baja | Obligatorio |
| Replicación primaria/secundaria | Alta | Para producción crítica |
| Alta disponibilidad automática | Alta | Solo si el negocio requiere RTO bajo |

### 12.2 Recomendación por fases

Fase 1, cliente pequeño/mediano:

- PostgreSQL en Docker con volumen persistente.
- Backups diarios locales comprimidos.
- Copia externa diaria a almacenamiento S3 compatible.
- Snapshot diario del VPS.
- Monitoreo con Grafana y revisión de alertas.
- Prueba de restauración mensual.

Fase 2, producción importante:

- Separar servidor de aplicación y servidor de base de datos.
- Red privada o VPN entre aplicación y PostgreSQL.
- Backups externos cifrados.
- Alertas por correo, Slack o Teams.
- Prueba de restauración quincenal.
- Documentar RPO/RTO.

Fase 3, producción crítica:

- Replicación PostgreSQL primaria/secundaria.
- Backups continuos con WAL archiving.
- Failover documentado y probado.
- Monitoreo centralizado.
- Infraestructura como código.
- Plan de recuperación ante desastres.

---

## 13. Mantenimiento y monitoreo

Estado de contenedores:

```bash
cd /opt/postgres-platform
docker compose ps
docker stats
```

Ver logs:

```bash
docker compose logs -f --tail=100 postgres
docker compose logs -f --tail=100 prometheus
```

Reiniciar servicios:

```bash
docker compose restart postgres
docker compose restart prometheus
```

Validar recursos:

```bash
htop
free -h
df -h
du -sh /var/lib/docker /opt/postgres-platform/backups
```

Validar backups:

```bash
ls -lh /opt/postgres-platform/backups
tail -n 100 /opt/postgres-platform/logs/backup_postgres.log
```

Actualizar imágenes Docker de forma segura:

```bash
cd /opt/postgres-platform
/opt/postgres-platform/scripts/backup_postgres.sh
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Procedimiento recomendado de actualización:

1. Notificar ventana de mantenimiento.
2. Confirmar espacio en disco.
3. Ejecutar backup manual.
4. Descargar imágenes nuevas.
5. Levantar contenedores.
6. Validar PostgreSQL, Prometheus y backups en el servidor de base de datos.
7. Validar Grafana por separado en el servidor Grafana.
8. Documentar versión anterior y nueva.

No hacer upgrade mayor de PostgreSQL, por ejemplo 15 a 16, sin procedimiento específico de migración y backup probado.

---

## 14. Problemas comunes y solución

### Docker no inicia

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker -n 200 --no-pager
sudo systemctl restart docker
```

Revisar instalación, espacio en disco y conflictos con `containerd`.

### PostgreSQL no levanta

```bash
docker logs --tail=200 postgres-db
docker inspect postgres-db --format '{{json .State.Health}}' | jq
df -h
```

Revisar `.env`, volumen, permisos, versión de imagen y puerto.

### Puerto ocupado

```bash
sudo ss -ltnp | grep ':5432\|:5050\|:19090\|:9090\|:3000'
```

Cambiar puerto en `.env`, ajustar `PROMETHEUS_HOST_PORT=19090` si ya está ocupado, o detener el servicio que ocupa el puerto.

### Error de permisos en volúmenes

```bash
docker compose down
docker volume inspect postgres_data
docker compose up -d
```

No cambiar permisos internos del volumen sin diagnóstico. PostgreSQL requiere ownership específico dentro del contenedor.

### Grafana no conecta a Prometheus

En el servidor Grafana:

```bash
curl -v http://10.10.10.10:19090/-/healthy
curl -s http://10.10.10.10:19090/api/v1/targets | jq
sudo journalctl -u grafana-server -n 100 --no-pager
```

Validar que el datasource use `http://10.10.10.10:19090`, que el firewall del servidor PostgreSQL permita la IP privada/VPN del servidor Grafana y que `PROMETHEUS_BIND_ADDRESS` no sea `127.0.0.1` cuando Grafana está en otro servidor.

### Prometheus no detecta exporters

```bash
docker exec -it prometheus wget -qO- http://node-exporter:9100/metrics | head
docker exec -it prometheus wget -qO- http://postgres-exporter:9187/metrics | head
curl -s http://${PROMETHEUS_BIND_ADDRESS}:${PROMETHEUS_HOST_PORT}/api/v1/targets | jq
```

Revisar `prometheus.yml`, nombres de servicios y healthchecks.

### Backup vacío

```bash
tail -n 100 /opt/postgres-platform/logs/backup_postgres.log
docker exec -it postgres-db psql -U app_owner -d cliente_app_prod -c '\dt'
```

Confirmar que la base correcta contiene datos y que el usuario tiene permisos.

### Error de autenticación

```bash
docker exec -it postgres-db psql -U app_owner -d cliente_app_prod
```

Revisar credenciales en `.env`. Si el volumen ya existía, cambiar `POSTGRES_PASSWORD` en `.env` no cambia automáticamente la contraseña interna. Debe cambiarse con SQL:

```sql
ALTER USER app_owner WITH PASSWORD 'NUEVA_CLAVE_SEGURA';
```

### Disco lleno

```bash
df -h
du -h --max-depth=1 /var/lib/docker | sort -h
du -h --max-depth=1 /opt/postgres-platform | sort -h
docker system df
```

Limpiar con cuidado:

```bash
docker image prune -a
```

No borrar volúmenes sin backup.

### Contenedor reiniciándose

```bash
docker compose ps
docker logs --tail=200 NOMBRE_CONTENEDOR
docker inspect NOMBRE_CONTENEDOR --format '{{json .State}}' | jq
```

Revisar variables, puertos, permisos, memoria insuficiente y dependencias.

---

## 15. Recomendaciones adicionales

- Configurar HTTPS para Grafana en su servidor separado y para pgAdmin mediante Nginx, Caddy o Traefik si se accede fuera de VPN.
- Usar VPN o red privada para PostgreSQL, Prometheus y exporters.
- Implementar alertas: disco mayor a 80%, PostgreSQL caído, backups fallidos, CPU sostenida alta, memoria baja.
- Enviar backups a almacenamiento externo cifrado: S3, Cloudflare R2, Backblaze B2 o NAS.
- Cifrar discos o volúmenes si el proveedor lo permite.
- Documentar responsables de acceso y rotación de credenciales.
- Probar restauración en un ambiente separado antes de considerar el sistema listo.
- Mantener inventario de versiones: Debian, Docker, PostgreSQL, Grafana y Prometheus.
- Evaluar `pg_stat_statements` para análisis de consultas lentas.
- Evaluar PgBouncer si la aplicación abre muchas conexiones.
- Definir RPO y RTO con el negocio.

---

## 16. Checklist final de entrega

| Validación | Estado |
|---|---|
| Sistema Linux actualizado | ☐ |
| Usuario administrador creado | ☐ |
| SSH protegido y root deshabilitado | ☐ |
| Zona horaria configurada | ☐ |
| UFW activo con reglas revisadas | ☐ |
| Fail2Ban activo | ☐ |
| Docker funcionando | ☐ |
| Docker Compose moderno funcionando | ☐ |
| Estructura `/opt/postgres-platform` creada en servidor PostgreSQL/Prometheus | ☐ |
| `.env` configurado sin `CAMBIAR_CLIENTE` | ☐ |
| PostgreSQL funcionando | ☐ |
| PostgreSQL no expuesto públicamente sin control | ☐ |
| Prometheus recolectando métricas y publicado en `19090` solo por red privada/VPN | ☐ |
| Grafana instalado en servidor separado, accesible y protegido | ☐ |
| Dashboard importado | ☐ |
| Backup manual probado | ☐ |
| Crontab activo | ☐ |
| Restauración probada | ☐ |
| Puertos revisados | ☐ |
| Accesos protegidos por VPN/IP whitelist | ☐ |
| Procedimiento de mantenimiento documentado | ☐ |

---

## 17. Referencias oficiales consultadas

- [Docker Engine en Debian](https://docs.docker.com/engine/install/debian/)
- [Docker Compose plugin en Linux](https://docs.docker.com/compose/install/linux/)
- [Instalación de Grafana en Debian o Ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
- [Referencia de Docker Compose](https://docs.docker.com/reference/compose-file/)
