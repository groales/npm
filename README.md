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
- **Red**: `npm_network` dedicada para el stack

## Pasos de despliegue

### 1. Clonar el repositorio

```bash
git clone https://git.ictiberia.com/groales/npm
cd npm
```

### 2. Levantar el stack

```bash
docker compose up -d
```

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

Para proxy a contenedores en otras redes Docker:

```yaml
services:
  app:
    # ... configuración existente ...
    networks:
      - npm_network
      - otra_red_docker

networks:
  npm_network:
    name: npm_network
  otra_red_docker:
    external: true
```

Luego en NPM usa el **nombre del contenedor** como Forward Hostname (ej: `portainer`).

## Documentación adicional

Consulta la [**Wiki del proyecto**](https://git.ictiberia.com/groales/npm/wiki) para documentación detallada:

- [Guía de Configuración](https://git.ictiberia.com/groales/npm/wiki/Configuracion)
- [Certificados SSL/TLS](https://git.ictiberia.com/groales/npm/wiki/SSL)
- [Backup y Restauración](https://git.ictiberia.com/groales/npm/wiki/Backup)
- [Configuración Avanzada](https://git.ictiberia.com/groales/npm/wiki/Avanzado)

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
