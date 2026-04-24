# NGINX Proxy Manager

Servicio de proxy inverso con interfaz web para publicar aplicaciones internas con HTTPS.

Referencia oficial: https://nginxproxymanager.com/

## Caracteristicas

- Gestion web de hosts, redirecciones y reglas.
- Emision de certificados SSL con Let's Encrypt.
- Panel administrativo en puerto dedicado.
- Persistencia de configuracion y certificados en volumenes Docker.

## Requisitos Previos

- Docker Engine instalado.
- Docker Compose instalado.
- Puertos disponibles en el host: `80`, `81` y `443`.
- DNS apuntando al servidor para emision de certificados.
- Red Docker externa `proxy` creada en el host.

## Archivos del Repositorio

- `compose.yaml`
- `README.md`
- `data/` (se crea automaticamente al arrancar)
- `letsencrypt/` (se crea automaticamente al arrancar)

---

## Despliegue

### 1. Levantar el servicio

Si no existe la red externa, creala primero:

```bash
docker network create proxy
```

Luego levanta el servicio:

```bash
docker compose up -d
```

### 2. Verificar arranque

```bash
docker compose ps
docker compose logs -f app
```

---

## Acceso Inicial

- Panel de administracion: `http://IP_DEL_SERVIDOR:81`
- Entrada HTTP de proxys: puerto `80`
- Entrada HTTPS de proxys: puerto `443`

Credenciales iniciales:

- Email: `admin@example.com`
- Password: `changeme`

Cambia estas credenciales en el primer inicio.

---

## Uso Basico

1. Inicia sesion en el panel.
2. Crea un Proxy Host con tu dominio.
3. Define el host y puerto interno del servicio destino.
4. Solicita certificado SSL y activa redireccion a HTTPS.
5. Guarda y valida acceso.

---

## Comandos Utiles

```bash
docker compose logs -f app
docker compose restart app
docker compose pull
docker compose up -d
```

---

## Seguridad

1. Restringe el acceso al puerto administrativo `81`.
2. Publica servicios solo por HTTPS.
3. Habilita controles de acceso para aplicaciones sensibles.
4. Realiza backup periodico de los volumenes de datos y certificados.

En esta configuracion, los datos persisten en carpetas locales:

- `./data`
- `./letsencrypt`

---

## Recursos

- Sitio oficial: https://nginxproxymanager.com/
- Repositorio: https://github.com/NginxProxyManager/nginx-proxy-manager
- Imagen Docker: https://hub.docker.com/r/jc21/nginx-proxy-manager

---

## Persistencia y Red

- Persistencia mediante bind mounts locales:
	- `./data:/data`
	- `./letsencrypt:/etc/letsencrypt`
- Red usada por el servicio: red externa `proxy`.
