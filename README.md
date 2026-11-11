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
2. Crea una carpeta para el proyecto y guarda estos dos archivos en ella.

Contenido de `compose.yaml`:

```yaml
services:
  app:
    container_name: nginx-proxy-manager
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    environment:
      - TZ=${TZ:-Europe/Madrid}
    ports:
      - "80:80"      # HTTP
      - "81:81"      # Admin UI
      - "443:443"    # HTTPS
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt

networks:
  default:
    external: true
    name: proxy
```

Contenido de `.env`:

```dotenv
# Zona horaria del contenedor, por ejemplo Europe/Madrid
TZ=Europe/Madrid
```

3. Crea la red externa una sola vez: `docker network create proxy` (si aún no existe).
4. Desde la carpeta del proyecto, valida e inicia el servicio:

```bash
docker compose config --quiet
docker compose up -d --pull always
```

Compose crea automáticamente `data/` y `letsencrypt/` al iniciar, si no existen. Ajusta `TZ` en `.env` si necesitas otra zona horaria.

## Publicar servicios Docker mediante la red `proxy`

Conecta Nginx Proxy Manager y cada servicio de destino a la red externa `proxy`. Así NPM puede resolver el nombre del servicio por DNS interno de Docker; no es necesario publicar en el anfitrión el puerto del servicio de destino.

En el `compose.yaml` de cada aplicación, declara la red externa y asígnala al servicio:

```yaml
services:
  web:
    image: nginx:alpine
    restart: unless-stopped
    expose:
      - "80"
    networks:
      - proxy

networks:
  proxy:
    external: true
    name: proxy
```

Usa el nombre real del servicio y el puerto donde escucha dentro del contenedor. `expose` es opcional para la conectividad entre contenedores; no publica el puerto en el anfitrión. No uses `localhost` como destino en NPM, porque dentro del contenedor apunta al propio NPM.

Después de iniciar ambos proyectos, abre <http://localhost:81> y crea un **Proxy Host**:

1. En **Domain Names**, escribe el dominio público y configura sus registros DNS para que apunten a la dirección del anfitrión.
2. En **Forward Hostname / IP**, indica el nombre del servicio Compose conectado a `proxy` (en el ejemplo, `web`).
3. En **Forward Port**, indica el puerto interno de la aplicación (en el ejemplo, `80`).
4. Selecciona el esquema HTTP o HTTPS que realmente use el servicio de destino y guarda.
5. Para TLS, abre la pestaña **SSL**, solicita o selecciona un certificado y activa **Force SSL** si corresponde. Para emitir certificados de Let's Encrypt, el dominio debe resolver públicamente al anfitrión y los puertos 80/443 deben ser accesibles desde Internet.

Si NPM y la aplicación ya se ejecutan en proyectos Compose distintos, pueden compartir esta red siempre que ambos se conecten a la red externa `proxy`. Comprueba con `docker network inspect proxy` que los dos contenedores aparecen conectados.

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
