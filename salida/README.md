# Despliegue de la aplicación de notas

Aplicación web Flask que guarda notas en Redis. La web se ejecuta con Gunicorn en un contenedor sin privilegios y Redis usa persistencia AOF. Todo el despliegue se orquesta con Docker Compose.

Todos los comandos de este README se ejecutan desde la carpeta `salida/`, que es la raíz del despliegue.

## 1. Requisitos

- Docker Engine 24 o superior.
- Docker Compose v2 (`docker compose version`).
- Puerto `8000` libre en el host.
- `curl` para las pruebas de comprobación; también se puede usar el navegador.
- Conexión a Internet la primera vez, para descargar `python:3.12-slim` y `redis:7-alpine`.
- No hace falta instalar Python, Redis ni Gunicorn en el host: todo se ejecuta en contenedores.

El contexto de build debe contener estos archivos:

```text
salida/
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── app.py
├── requirements.txt
├── templates/
│   └── index.html
└── README.md
```

## 2. Variables de entorno

| Variable | Valor en este despliegue | Servicio | Obligatoria | Descripción |
|---|---|---|---|---|
| `REDIS_URL` | `redis://cache:6379` | `web` | Sí | URL de Redis. `cache` es el nombre del servicio dentro de la red de Compose. |
| `TITULO_APP` | `Tablón de notas` | `web` | No | Título que aparece en la portada y en el HTML. |

No se necesita ningún archivo `.env`. La configuración está en `docker-compose.yml` y no se guarda dentro de la imagen.

`CLAVE_ADMIN` no se usa en esta aplicación y no debe añadirse. Tampoco hay contraseña de Redis: por eso el servicio `cache` no publica el puerto `6379` y solo es accesible desde la red interna de Compose.

## 3. Puesta en marcha

```bash
cd salida
docker compose up -d --build
docker compose ps
```

La primera vez se descargan las imágenes base y se construye `notas-web:latest`. `--build` fuerza la reconstrucción de la imagen de la aplicación.

Hay que esperar a que los dos servicios aparezcan como `healthy`:

```text
NAME             IMAGE              STATUS
salida-cache-1   redis:7-alpine     Up (healthy)
salida-web-1     notas-web:latest   Up (healthy)
```

Comprobación rápida:

```bash
curl -sS http://localhost:8000/salud
```

Resultado esperado en un volumen nuevo:

```json
{"estado":"ok","notas":0}
```

Abrir en el navegador: <http://localhost:8000/>

Si el puerto `8000` está ocupado, cambiar en `docker-compose.yml` la línea `"8000:8000"` por, por ejemplo, `"8080:8000"` y usar `http://localhost:8080`.

## 4. Pruebas de comprobación

### 4.1. Salud y portada

```bash
curl -sS http://localhost:8000/salud
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8000/
```

Esperado:

```text
{"estado":"ok","notas":0}
200
```

### 4.2. Crear una nota y comprobarla

```bash
curl -sS -X POST -d 'texto=Nota de prueba' -o /dev/null -w '%{http_code}\n' http://localhost:8000/notas
curl -sS http://localhost:8000/ | grep -F 'Nota de prueba'
curl -sS http://localhost:8000/salud
docker compose exec -T cache redis-cli LLEN notas
```

Esperado:

```text
302
<li>Nota de prueba</li>
{"estado":"ok","notas":1}
1
```

### 4.3. Persistencia de Redis

Reiniciar solo la web no debe perder la nota:

```bash
docker compose restart web
curl -sS http://localhost:8000/salud
```

Esperado: `{"estado":"ok","notas":1}`.

También se puede probar a recrear los contenedores sin borrar el volumen:

```bash
docker compose down
docker compose up -d
curl -sS http://localhost:8000/salud
```

`docker compose down` conserva el volumen. **No usar `-v`** en esta prueba, porque borra los datos.

### 4.4. Usuario, salud y seguridad de la imagen

```bash
docker compose exec -T web id
docker compose exec -T web ls -a /app
docker compose exec -T web sh -c 'find /app -maxdepth 2 \( -name .env -o -name .git \) -print'
docker compose ps
```

Esperado:

```text
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
.
..
app.py
requirements.txt
templates
(sin salida para .env/.git)
salida-web-1     ...     Up (healthy)
salida-cache-1   ...     Up (healthy)
```

La imagen no contiene `.env` ni `.git`: `.dockerignore` los excluye del contexto de build. La web corre como `appuser` (UID 10001), no como root.

### 4.5. Tamaño y capas de la imagen

```bash
docker image ls
docker history notas-web:latest | head
```

En esta máquina, `notas-web:latest` ocupa aproximadamente **191 MB de uso en disco** y **46,1 MB de contenido**. Las capas más pesadas son las de la base Debian/Python; las capas propias de la aplicación (código, plantillas y usuario) son de pocos kB. `docker history | head` muestra las capas más recientes; para ordenarlas por tamaño:

```bash
docker history notas-web:latest --format '{{.Size}}\t{{.CreatedBy}}' | sort -hr | head
```

## 5. Dónde quedan los datos

Las notas se guardan en Redis, dentro del volumen con nombre `notas-datos`. Compose le añade el prefijo del proyecto, así que en este despliegue el volumen real es:

```text
salida_notas-datos
```

Está montado en `/data` dentro del contenedor `cache` y Redis arranca con `--appendonly yes`, por lo que usa el archivo AOF de Redis.

Comprobaciones:

```bash
docker volume ls | grep notas-datos
docker volume inspect salida_notas-datos
docker compose exec -T cache ls -la /data
```

El contenedor `web` no guarda datos: es stateless. Toda la información persistente está en Redis.

- `docker compose down` conserva el volumen y los datos.
- `docker compose down -v` elimina el volumen y, por tanto, las notas.

Si se cambia el nombre de la carpeta o el nombre del proyecto Compose, cambia el prefijo del volumen. En este README se usa `salida_notas-datos` porque el proyecto se llama `salida`.

## 6. Cómo ver los registros

```bash
docker compose logs -f web
docker compose logs -f cache
docker compose logs -f
docker compose logs --tail=100
```

- `web` muestra los logs de Gunicorn y los accesos HTTP. El `Dockerfile` arranca Gunicorn con `--access-logfile -`, así que los accesos salen por la salida estándar.
- `cache` muestra los logs de Redis.
- `docker compose logs -f` sigue los logs de todos los servicios.
- Para ver el estado de salud:

```bash
docker compose ps
docker inspect --format '{{json .State.Health}}' salida-web-1
docker inspect --format '{{json .State.Health}}' salida-cache-1
```

## 7. Parar y limpiar

### Parar sin borrar nada

```bash
docker compose stop
```

Detiene los contenedores, pero los conserva. Se vuelven a arrancar con `docker compose start`.

### Quitar contenedores y red, conservando los datos

```bash
docker compose down
```

Elimina los contenedores y la red, pero mantiene el volumen `salida_notas-datos`. Es la parada normal si se quiere conservar las notas.

### Limpieza total del despliegue

```bash
docker compose down -v
docker image rm notas-web:latest
```

`down -v` elimina contenedores, red y volumen; `docker image rm` elimina la imagen construida. Los datos se pierden.

### Limpieza completa para probar desde cero

Si también se quieren borrar las imágenes base y los restos de la auditoría que haya en la máquina:

```bash
docker compose down -v
docker image rm notas-web:latest
docker image rm python:3.12-slim redis:7-alpine 2>/dev/null || true
docker image rm notas-malo:latest redis:latest python:latest 2>/dev/null || true
```

Las imágenes `notas-malo`, `redis:latest` y `python:latest` son de la auditoría, no de este despliegue; si no existen, la línea no hace nada. La siguiente puesta en marcha descargará de nuevo las imágenes base que falten.

## 8. Reproducibilidad desde cero

Secuencia completa para comprobar que el despliegue se puede repetir:

```bash
cd salida
docker compose down -v
docker image rm notas-web:latest
docker compose up -d --build
docker compose ps
curl -sS http://localhost:8000/salud
```

Resultado esperado tras una limpieza con `-v`:

```json
{"estado":"ok","notas":0}
```

El contador vuelve a `0` porque el volumen se ha borrado. Si se han eliminado también las imágenes base, `docker compose up -d --build` las descarga automáticamente.

## 9. Problemas frecuentes

- **`permission denied` al usar Docker:** el usuario debe pertenecer al grupo `docker` o ejecutar los comandos con `sudo`. Después de añadirse al grupo, hay que cerrar y volver a abrir la sesión.
- **El puerto 8000 está ocupado:** cambiar el puerto del host en `docker-compose.yml`, por ejemplo `"8080:8000"`.
- **La web no pasa de `starting`:** revisar `docker compose logs web` y `docker compose logs cache`. La web espera a que Redis esté `healthy`.
- **`/salud` devuelve 503:** Redis no está disponible. Comprobar `docker compose ps` y los logs de `cache`.
- **Se han perdido las notas:** se ejecutó `docker compose down -v`, que borra el volumen. Para conservarlas, usar `docker compose down` sin `-v`.

## 10. Notas de seguridad

- La web se ejecuta como `appuser` (UID 10001), no como root.
- `.env`, `.git` y otros archivos locales se excluyen con `.dockerignore` y no se copian a la imagen.
- Redis no publica el puerto `6379`; solo es accesible desde la red interna de Compose.
- La configuración y el título están en `docker-compose.yml`, no dentro de la imagen.
- La imagen incluye un `HEALTHCHECK` contra `/salud`.
- No hay secretos en la imagen ni en el compose.

## 11. Validación de este README

Validado el 2026-10-21 siguiendo esta misma guía:

1. `docker compose down -v`.
2. `docker image rm` de `notas-web`, las imágenes de auditoría y las imágenes base; `docker image ls` quedó vacío.
3. `docker compose up -d --build` reconstruyó la imagen y descargó de nuevo las imágenes base.
4. `docker compose ps`: `salida-web-1` y `salida-cache-1` en `healthy`.
5. `/salud` devolvió `{"estado":"ok","notas":0}`.
6. Se creó `Nota de prueba README` (HTTP 302), se comprobó en la portada y en Redis (`LLEN notas` = 1), y persistió tras `docker compose restart web` y tras `docker compose down` + `docker compose up -d`.
7. Se borró la nota de prueba. Estado final: `/salud` con `notas:0`, ambos servicios `healthy` y volumen `salida_notas-datos` presente.
