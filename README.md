# Infraestructura: NGINX Proxy Manager

# NGINX Proxy Manager — Despliegue con Docker Compose

Este repositorio proporciona una forma sencilla de desplegar **NGINX Proxy Manager** usando Docker Compose, siguiendo las recomendaciones oficiales del proyecto.

## ¿Qué es NGINX Proxy Manager?

NGINX Proxy Manager es una herramienta de gestión de proxy inverso fácil de usar que te permite:

- 🌐 **Gestionar hosts proxy** sin conocimientos avanzados de NGINX
- 🔒 **Certificados SSL/TLS gratuitos** con Let's Encrypt automático
- 🔐 **Listas de acceso** y autenticación HTTP básica
- 📊 **Interfaz web moderna** basada en Tabler
- 🔄 **Redirecciones y streams** configurables fácilmente
- 👥 **Gestión de usuarios** y permisos

## Requisitos

- Docker y Docker Compose instalados en el host
- Puertos 80, 443 y 81 disponibles
- Dominio apuntando al servidor (para certificados Let's Encrypt)
- (Opcional) Acceso a los puertos 80/443 desde Internet para validación de certificados

## Qué incluye este stack

- **Servicio**: `nginx-proxy-manager` con la imagen `jc21/nginx-proxy-manager:latest`
- **Puertos**:
  - `80`: HTTP (redirige automáticamente a HTTPS)
  - `81`: Interfaz web de administración
  - `443`: HTTPS
- **Volúmenes**:
  - `npm_data`: Datos de configuración, base de datos y logs
  - `npm_letsencrypt`: Certificados SSL de Let's Encrypt
- **Red**: `proxy` (externa, compartida con otros servicios)

## Pasos de despliegue

### Opción 1: Docker Compose (Línea de comandos)

#### 1. Clonar el repositorio

```bash
git clone https://git.ictiberia.com/groales/npm
cd npm
```

#### 3. Levantar el stack

```bash
docker compose up -d
```

### Opción 2: Desplegar desde Portainer (Recomendado)

#### 1. Red proxy

La red `proxy` se creará automáticamente al desplegar el stack de NPM.

#### 2. Conectar Portainer a la red proxy

**Containers** → `portainer` → **Duplicate/Edit**
- En **Network** → **+ Connect to network** → Seleccionar `proxy`
- **Deploy**

O añadir a `docker-compose.override.yaml` de Portainer:
```yaml
services:
  portainer:
    networks:
      - proxy

networks:
  proxy:
    external: true
```

#### 3. Desplegar stack de NPM

**Stacks** → **Add stack**
- **Name**: `nginx-proxy-manager`
- **Build method**: 
  - **Git Repository**:
    - Repository URL: `https://git.ictiberia.com/groales/npm`
    - Repository reference: `refs/heads/main`
    - Compose path: `docker-compose.yaml`
  - O **Web editor**: Pegar contenido de `docker-compose.yaml`
- **Deploy the stack**

#### 4. Verificar despliegue

**Stacks** → `nginx-proxy-manager` → Ver logs del contenedor

### 3. Verificar el estado

```bash
docker ps --filter name=nginx-proxy-manager
```

### 4. Acceder a la interfaz de administración

Abre tu navegador en: **http://IP-del-servidor:81**

⏱️ **Primera vez**: El inicio puede tardar 1-2 minutos dependiendo del hardware.

### 5. Login inicial

**Credenciales por defecto**:
- **Email**: `admin@example.com`
- **Password**: `changeme`

🔐 **Importante**: Al primer login, se te pedirá cambiar estas credenciales inmediatamente.

## Uso básico

### Crear un Proxy Host

1. **Login** en la interfaz web (puerto 81)
2. Click en **"Hosts"** → **"Proxy Hosts"** → **"Add Proxy Host"**
3. Configurar:
   - **Domain Names**: `ejemplo.tudominio.com`
   - **Scheme**: `http` o `https`
   - **Forward Hostname/IP**: IP del servicio backend (ej: `192.168.1.100`)
   - **Forward Port**: Puerto del servicio (ej: `8080`)
4. Pestaña **"SSL"**:
   - Marcar **"Request a new SSL Certificate"**
   - Marcar **"Force SSL"**
   - Marcar **"HTTP/2 Support"**
   - Aceptar términos de Let's Encrypt
5. **Save**

✅ En pocos segundos tendrás un proxy HTTPS funcionando con certificado válido.

### Ejemplo: Proxy a Portainer (Contenedores en Red Compartida)

Si Portainer y NPM están en la misma red `proxy`:

1. **Crear Proxy Host** en NPM:
   - **Domain Names**: `portainer.tudominio.com`
   - **Scheme**: `https` (Portainer usa HTTPS)
   - **Forward Hostname/IP**: `portainer` (nombre del contenedor)
   - **Forward Port**: `9443`
   - **Cache Assets**: ❌
   - **Block Common Exploits**: ✅
   - **Websockets Support**: ✅ (necesario para Portainer)

2. **Pestaña SSL**:
   - ✅ **Request a new SSL Certificate**
   - ✅ **Force SSL**
   - ✅ **HTTP/2 Support**
   - Email: `tu@email.com`
   - ✅ **I Agree to Let's Encrypt ToS**

3. **Save**

Accede a: `https://portainer.tudominio.com` 🎉

💡 **Ventaja**: No necesitas exponer el puerto 9443 de Portainer, NPM gestiona todo el tráfico HTTPS.

## Configuración avanzada

### Variables de entorno

Crea un archivo `.env` para personalizar:

```env
# Zona horaria
TZ=Europe/Madrid

# Puertos personalizados (si los defaults están ocupados)
HTTP_PORT=80
HTTPS_PORT=443
ADMIN_PORT=81
```

Luego modifica `docker-compose.yaml`:

```yaml
environment:
  TZ: "${TZ:-Europe/Madrid}"
ports:
  - "${HTTP_PORT:-80}:80"
  - "${ADMIN_PORT:-81}:81"
  - "${HTTPS_PORT:-443}:443"
```

### Integración con redes Docker

Este stack usa la red `proxy` (externa) para compartir conectividad con otros servicios Docker.

#### Conectar otros servicios a la red proxy

**Opción 1: Desde Portainer**

Para cada stack/contenedor que quieras exponer via NPM:

1. **Containers** → Seleccionar contenedor → **Duplicate/Edit**
2. **Network** → **+ Connect to network** → `proxy`
3. **Deploy**

**Opción 2: En docker-compose.yaml**

```yaml
services:
  mi_servicio:
    # ... configuración existente ...
    networks:
      - mi_red_interna
      - proxy  # Añadir red proxy

networks:
  mi_red_interna:
    name: mi_red_interna
  proxy:
    external: true  # Red creada por NPM
```

#### Configurar Proxy Host en NPM

Una vez el servicio esté en la red `proxy`:

- **Forward Hostname**: Usar **nombre del contenedor** (ej: `portainer`, `jellyfin`, `nextcloud`)
- **Forward Port**: Puerto **interno** del contenedor (no necesitas publicarlo con `-p`)

**Ejemplo**: Si tienes Jellyfin corriendo:
- Container name: `jellyfin`
- Puerto interno: `8096`
- En NPM: Forward Hostname = `jellyfin`, Port = `8096`

💡 **Ventaja**: No expones puertos innecesarios al host, todo el tráfico pasa por NPM.

## Documentación adicional

Consulta la [**Wiki del proyecto**](https://git.ictiberia.com/groales/npm/wiki) para documentación detallada:

- [Guía de Configuración](https://git.ictiberia.com/groales/npm/wiki/Configuracion)
- [Certificados SSL/TLS](https://git.ictiberia.com/groales/npm/wiki/SSL)

## Solución de problemas

### Puerto ocupado

Si algún puerto está en uso:

```bash
# Ver qué proceso usa el puerto
Get-NetTCPConnection -LocalPort 80
# o en Linux
sudo netstat -tulpn | grep :80

# Cambiar puerto en docker-compose.yaml
ports:
  - "8080:80"  # HTTP en puerto 8080
```

### Contenedor no arranca

```bash
# Ver logs
docker logs nginx-proxy-manager

# Logs en tiempo real
docker logs -f nginx-proxy-manager
```

### Certificado SSL no se genera

Verifica que:
1. ✅ El dominio apunta correctamente al servidor (DNS)
2. ✅ Puertos 80 y 443 accesibles desde Internet
3. ✅ No hay firewall bloqueando el tráfico
4. ✅ El dominio no supera los límites de Let's Encrypt (5 certs/semana)

```bash
# Test de accesibilidad desde exterior
curl -I http://tudominio.com

# Verificar DNS
nslookup tudominio.com
dig tudominio.com
```

### Olvido de credenciales

Resetear contraseña del admin:

```bash
docker exec -it nginx-proxy-manager /bin/bash

# Dentro del contenedor
cd /app
npx knex migrate:latest --env production
npx knex seed:run --env production

# Credenciales vuelven a:
# Email: admin@example.com
# Password: changeme
```

## Personalización

### Cambiar base de datos a MySQL/PostgreSQL

Por defecto NPM usa SQLite. Para usar MySQL:

```yaml
services:
  app:
    environment:
      DB_MYSQL_HOST: "mysql"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm_password"
      DB_MYSQL_NAME: "npm"
    depends_on:
      - mysql

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: npm
      MYSQL_USER: npm
      MYSQL_PASSWORD: npm_password
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  npm_data:
  npm_letsencrypt:
  mysql_data:
```

### Deshabilitar IPv6

Si tienes problemas con IPv6:

```yaml
environment:
  DISABLE_IPV6: "true"
```

## Recursos oficiales

- 📘 [Documentación oficial](https://nginxproxymanager.com/)
- 🐛 [GitHub - Issues](https://github.com/NginxProxyManager/nginx-proxy-manager/issues)
- 💬 [Reddit Community](https://reddit.com/r/nginxproxymanager)
- 🎥 [Setup Guide (YouTube)](https://www.youtube.com/watch?v=P3imFC7GSr0)
- 🐳 [Docker Hub](https://hub.docker.com/r/jc21/nginx-proxy-manager)

## Seguridad

### Recomendaciones

1. ✅ **Cambiar credenciales** por defecto inmediatamente
2. ✅ **Restringir acceso** al puerto 81 (admin) con firewall
3. ✅ **Usar certificados SSL** para todos los hosts
4. ✅ **Habilitar listas de acceso** para hosts sensibles
5. ✅ **Actualizar regularmente** con `docker compose pull && docker compose up -d`
6. ✅ **Backup periódico** del volumen `npm_data`

### Proteger puerto de administración

Limitar acceso al puerto 81 solo desde IP local:

```yaml
ports:
  - "80:80"
  - "127.0.0.1:81:81"  # Solo accesible desde localhost
  - "443:443"
```

Luego accede via túnel SSH:

```bash
ssh -L 8081:localhost:81 usuario@servidor
# Abre http://localhost:8081 en tu navegador
```

O usa VPN (WireGuard, Tailscale) para acceso seguro remoto.

## Actualización

```bash
# Pull nueva imagen
docker compose pull

# Reiniciar con nueva versión
docker compose up -d

# Ver logs
docker logs -f nginx-proxy-manager
```

NPM mantiene compatibilidad de base de datos entre versiones, las actualizaciones suelen ser sin problemas.

---

**Versión**: Latest (imagen Docker rolling)  
**Última actualización**: Noviembre 2025
