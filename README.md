# NGINX Proxy Manager

Servicio de proxy inverso con interfaz web para publicar aplicaciones internas con HTTPS.

Referencia oficial de instalación: https://nginxproxymanager.com/guide/

## Características

- Gestión web de hosts, redirecciones y reglas.
- Emisión de certificados SSL con Let's Encrypt.
- Panel administrativo en puerto dedicado.
- Persistencia de configuración y certificados.

## Requisitos Previos

- Docker Engine instalado.
- Docker Compose instalado.
- Puertos disponibles en el host: `80`, `81` y `443`.
- Red Docker externa `proxy` creada.

## Archivos de este Repositorio

- `compose.yaml` - Definición del servicio.
- `README.md` - Esta documentación.
- `data/` - Persistencia de configuración (se crea al arrancar).
- `letsencrypt/` - Certificados TLS (se crea al arrancar).

---

## Despliegue con Docker Compose

### 1. Clonar el repositorio

```bash
git clone https://github.com/groales/npm.git
cd npm
```

### 2. Revisar `compose.yaml`

```yaml
services:
	app:
		container_name: nginx-proxy-manager
		image: jc21/nginx-proxy-manager:latest
		restart: unless-stopped
		environment:
			TZ: "Europe/Madrid"
		ports:
			- "80:80"
			- "81:81"
			- "443:443"
		volumes:
			- ./data:/data
			- ./letsencrypt:/etc/letsencrypt

networks:
	default:
		external: true
		name: proxy
```

### 3. Levantar el servicio

```bash
docker network create proxy
docker compose up -d
```

## Método Alternativo: Crear Manualmente

Puedes copiar el `compose.yaml` en una carpeta nueva y ejecutar el mismo despliegue.

---

## Acceso Inicial

- Panel administrativo: `http://IP_DEL_SERVIDOR:81`
- Entrada HTTP de proxys: puerto `80`
- Entrada HTTPS de proxys: puerto `443`

Credenciales por defecto:

- Email: `admin@example.com`
- Password: `changeme`

Cámbialas en el primer inicio.

## Comandos Útiles

```bash
docker compose logs -f app
docker compose restart app
docker compose pull
docker compose up -d
docker compose down
```

## Estructura de Volúmenes

```text
Bind mounts:
├── ./data -> /data
└── ./letsencrypt -> /etc/letsencrypt
```

## Configuración Avanzada

- Ajusta zona horaria con `TZ`.
- Configura Proxy Hosts con certificados por dominio.
- Puedes añadir Access Lists para restringir acceso por IP.

## Solución de Problemas

Si no emite certificados:

- Verifica que el dominio resuelve al servidor.
- Asegura exposición de puertos `80` y `443`.
- Revisa logs con `docker compose logs -f app`.

## Seguridad

- Restringe acceso al panel (`81`) por firewall.
- Publica aplicaciones por HTTPS.
- Usa contraseñas robustas y MFA si aplica.

## Backup y Restauración

```bash
# Backup
tar -czf npm-backup-$(date +%Y%m%d).tar.gz ./data ./letsencrypt

# Restauración
docker compose down
rm -rf ./data ./letsencrypt
tar -xzf npm-backup-YYYYMMDD.tar.gz
docker compose up -d
```

## Actualización

```bash
docker compose pull
docker compose up -d
docker compose logs -f app
```

## Recursos

- Sitio oficial: https://nginxproxymanager.com/
- Repositorio oficial: https://github.com/NginxProxyManager/nginx-proxy-manager
- Imagen Docker: https://hub.docker.com/r/jc21/nginx-proxy-manager

## Licencia

Este repositorio de configuración es de uso libre. Revisa la licencia del proyecto original en su repositorio oficial.
