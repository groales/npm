# Nginx Proxy Manager

Nginx Proxy Manager (NPM) ofrece una interfaz web para administrar hosts de proxy inverso, redirecciones y certificados TLS/SSL. Este proyecto ejecuta el contenedor mantenido por el propio proyecto: `jc21/nginx-proxy-manager`.

## Enlaces oficiales

- Proyecto y código fuente: <https://github.com/NginxProxyManager/nginx-proxy-manager>
- Guía y configuración: <https://nginxproxymanager.com/guide/>
- Instalación completa: <https://nginxproxymanager.com/setup/>
- Configuración avanzada: <https://nginxproxymanager.com/advanced-config/>
- Actualización: <https://nginxproxymanager.com/upgrading/>
- Imagen en Docker Hub: <https://hub.docker.com/r/jc21/nginx-proxy-manager>

## Requisitos y puertos

- Docker Engine y Docker Compose v2 (`docker compose`).
- Los puertos 80, 81 y 443 del anfitrión deben estar libres.
- HTTP público: `80` → contenedor `80`.
- Interfaz administrativa: `81` → contenedor `81`, publicada en todas las interfaces del anfitrión.
- HTTPS público: `443` → contenedor `443`.
- La red Docker externa `proxy` debe existir antes de iniciar el servicio. Conecta también a ella los servicios de destino.
- Para recibir tráfico externo de Internet, configura el DNS de los dominios y reenvía 80/443 en el router o firewall al anfitrión Docker.

La configuración mínima usa SQLite integrado y persiste los archivos bajo `./data` y `./letsencrypt`. La imagen admite `amd64` y `arm64`; ARMv7 requiere una etiqueta antigua indicada por el mantenedor.

## Despliegue clonando un repositorio Git

Este directorio no configura ni publica un remoto. Después de publicar el repositorio en el servicio Git que elijas, sustituye la URL de ejemplo por la URL real:

```bash
git clone <URL-del-repositorio> npm
cd npm
cp .env.example .env
# Edita .env si necesitas cambiar la zona horaria
docker network create proxy
docker compose config --quiet
docker compose up -d --pull always
```

Si vas a usar la copia local ya creada, no hace falta clonar: entra en este directorio y continúa con los comandos de operación indicados abajo.

## Despliegue manual sin Git

1. Instala Docker Engine con Compose v2.
2. Copia `compose.yaml` y `.env.example` a una carpeta del anfitrión.
3. Crea `.env` copiando `.env.example` y ajusta la zona horaria si es necesario.
4. Crea la red externa una sola vez: `docker network create proxy` (si aún no existe).
5. Desde esa carpeta, valida e inicia el servicio:

```bash
docker compose config --quiet
docker compose up -d --pull always
```

Compose crea automáticamente `data/` y `letsencrypt/` al iniciar, si no existen. En Windows PowerShell, el equivalente a `cp .env.example .env` es `Copy-Item .env.example .env`.

## Variables de entorno

Compose lee `.env` automáticamente para interpolar los valores de `compose.yaml`. La plantilla `.env.example` se puede compartir; `.env` es local y está excluido de Git.

| Variable | Valor predeterminado | Uso |
| --- | --- | --- |
| `TZ` | `Europe/Madrid` | Zona horaria del contenedor. Imagen y puertos se definen directamente en `compose.yaml`. |

No añadas contraseñas ni otros secretos a la imagen, a `compose.yaml` o al repositorio. La primera ejecución solicita crear el usuario administrador desde la interfaz.

## Persistencia

- `./data:/data`: base de datos SQLite, claves JWT, configuración y datos de NPM.
- `./letsencrypt:/etc/letsencrypt`: certificados y datos relacionados con Let's Encrypt.

Estas carpetas contienen estado local y están excluidas de Git. Haz copias de seguridad coherentes de ambas antes de actualizar o migrar el anfitrión.

## Operación

Ejecuta los comandos desde la carpeta del proyecto:

```bash
# Estado
docker compose ps

# Registros
docker compose logs -f app

# Parada conservando datos
docker compose down

# Actualizar a la imagen configurada
docker compose up -d --pull always
```

Para acceder a la consola de administración abre <http://localhost:81>. El puerto 81 se publica en todas las interfaces del anfitrión; restríngelo con firewall y no lo expongas directamente a Internet. La inicialización puede tardar un poco. Desde la interfaz, crea el usuario administrador y configura los proxy hosts y certificados.

## Seguridad

- La interfaz administrativa se publica en todas las interfaces mediante el puerto 81; limítala con firewall a redes de confianza y no la expongas directamente a Internet.
- Usa contraseñas únicas y robustas y mantén Docker y la imagen actualizados.
- Publica 80/443 solo cuando quieras aceptar conexiones públicas; configura también DNS y reglas de firewall apropiadas.
- `.env`, `data/` y `letsencrypt/` no deben añadirse al repositorio. Protege sus copias de seguridad, especialmente los certificados y la base de datos.
- `latest` recibe cambios en la imagen; revisa las notas de versión y haz copias de seguridad antes de actualizar.
