# Auditoría de `sin-revisar`

**Fecha:** 2026-10-21  
**Carpeta auditada:** `/home/alumno/practicas/docker/recursos/notas/sin-revisar`  
**Motivo:** la copia de `sin-revisar` que había en el proyecto se borró; se audita la copia original de `recursos/notas`, que es idéntica. El código no se ha restaurado en `salida/`.  
**Imagen mala construida desde:** `/home/alumno/practicas/docker/recursos/notas` con `docker build -f sin-revisar/Dockerfile -t notas-malo .`.

## 1. Informe inicial del agente

1. `FROM python:latest` no está fijada, cambia con el tiempo y arrastra una imagen base muy grande; además, actualmente es Python 3.15.0.
2. `apt-get install -y python3-pip curl vim git` instala herramientas que la aplicación no necesita, una segunda toolchain de Python (3.13) y deja cachés de APT; la capa de APT pesa 131 MB.
3. `COPY . .` se ejecuta antes de `pip install`, así que cualquier cambio en el código invalida la capa de dependencias. Además no hay `.dockerignore` y se copia la propia carpeta `sin-revisar`.
4. No hay `USER`: la web se ejecuta como `root`.
5. `CMD python app.py` usa el servidor de desarrollo de Flask, con `debug=True`; es formato shell, no JSON, y deja `python app.py` detrás de `/bin/sh`.
6. No hay `EXPOSE 8000`.
7. No hay `HEALTHCHECK`.
8. `ENV REDIS_URL` y `ENV CLAVE_ADMIN` quedan dentro de la imagen; `CLAVE_ADMIN` es un secreto legible con `docker image inspect`.
9. En el compose, `build: .` apunta a `sin-revisar`, no a la raíz de la aplicación; el build falla porque no encuentra `requirements.txt`.
10. El servicio `cache` publica `6379:6379`, exponiendo Redis a la red exterior sin autenticación.
11. El servicio `cache` no tiene volumen con nombre ni configuración de persistencia; los datos quedan en la capa escribible del contenedor.
12. No hay healthcheck en el compose y `depends_on` solo espera a que el contenedor arranque, no a que Redis esté listo.
13. `redis:latest` no está fijada y es más grande que `redis:7-alpine`.
14. `restart: always` reinicia el servicio incluso después de una parada manual; es menor, pero `unless-stopped` es preferible.
15. `CLAVE_ADMIN` se repite en el compose en texto plano, pero la aplicación no la usa; da una falsa sensación de seguridad y no protege Redis, que no tiene contraseña.

## 2. Verificación

### 2.1. Tabla de problemas

| # | Problema señalado | ¿Es cierto? (evidencia) | Gravedad | Corrección |
|---:|---|---|---|---|
| 1 | `python:latest` sin fijar y base enorme | **Sí.** `docker image ls`: mala 1.84 GB / 475 MB; buena 191 MB / 46.1 MB. `python:latest`: 1.64 GB; `python:3.12-slim`: 179 MB. `python:latest` es 3.15.0. | Alta | Fijar `python:3.12-slim` (mejor con digest). |
| 2 | APT instala paquetes innecesarios y una Python duplicada | **Sí.** Capa APT de 131 MB; `/usr/share/vim` 44 MB; `/var/lib/apt/lists` 21 MB. Instala Python 3.13 y pip 25.1.1, pero la app usa Python 3.15 y pip 26.2.1 de `/usr/local/bin`. | Media | Quitar `curl`, `vim`, `git` y `python3-pip`; usar slim; limpiar listas si se instala algo. |
| 3 | `COPY . .` antes de instalar dependencias y sin `.dockerignore` | **Sí.** El orden del Dockerfile hace que un cambio de código invalide la capa de `pip`; `/app` contiene `sin-revisar`; no hay `.dockerignore`. | Media | Copiar primero `requirements.txt`, instalar, y después el código; añadir `.dockerignore`. |
| 4 | La web se ejecuta como root | **Sí.** `docker run --rm --entrypoint id notas-malo` devuelve `uid=0(root)`. `Config.User` está vacío y el PID 1 es `/bin/sh -c python app.py`, también root. | Alta | Crear `appuser` y usar `USER appuser`. |
| 5 | Servidor de desarrollo Flask, debug activo y `CMD` shell | **Sí.** `Config.Cmd` es `["/bin/sh","-c","python app.py"]`; `curl -I` devuelve `Server: Werkzeug/3.1.9 Python/3.15.0`; los logs muestran `Debug mode: on`, `WARNING: This is a development server` y `Debugger is active!`. Gunicorn 22.0.0 está instalado, pero no se usa. | Alta | `CMD` JSON con gunicorn, sin `debug=True`; el proceso principal debe ser gunicorn. |
| 6 | Falta `EXPOSE 8000` | **Sí, pero es irrelevante para el funcionamiento.** `Config.ExposedPorts` es `null`, pero `docker run -p 8001:8000` funciona y el compose publica 8000. `EXPOSE` es documentación, no publica el puerto. | Baja / informativa | Añadir `EXPOSE 8000` por claridad. |
| 7 | Falta `HEALTHCHECK` | **Sí.** La imagen no define healthcheck y el compose malo tampoco; no hay estado `healthy` ni comprobación de `/salud`. | Media | Añadir `HEALTHCHECK` contra `/salud`. |
| 8 | `REDIS_URL` y `CLAVE_ADMIN` están dentro de la imagen | **Sí.** `docker image inspect notas-malo --format '{{json .Config.Env}}'` incluye `REDIS_URL=redis://cache:6379` y `CLAVE_ADMIN=admin1234`; `docker history` muestra las instrucciones `ENV`. El compose malo también repite `CLAVE_ADMIN`. | Alta (secreto) / Media (config) | Sacar la configuración al compose; no meter secretos en la imagen. |
| 9 | El compose malo no construye por el contexto `build: .` | **Sí.** `docker compose config` muestra `context: .../sin-revisar`; `docker compose build` falla con `ERROR: Could not open requirements file: ... 'requirements.txt'`. | Alta: la pila no arranca | Usar `build: {context: .., dockerfile: sin-revisar/Dockerfile}` o poner el Dockerfile en la raíz del proyecto. |
| 10 | Redis se publica al exterior | **Sí.** `docker compose config` muestra `published: "6379"`; `ss -ltn` muestra `0.0.0.0:6379`; `redis-cli -h 127.0.0.1 ping` devuelve `PONG` desde el host. No hay contraseña. | Alta | Quitar `ports` de `cache`; dejar Redis solo en la red interna. |
| 11 | Redis no tiene volumen con nombre ni persistencia | **Sí.** `docker inspect sin-revisar-cache-1 --format ...` da `sin volumen`; `Mounts` es `[]`. La cache buena monta `salida_notas-datos -> /data`. | Media | Volumen con nombre `notas-datos:/data` y AOF (`--appendonly yes`). |
| 12 | No hay healthcheck de Redis y `depends_on` no espera a que esté sano | **Sí.** `docker compose config` muestra `condition: service_started`; no hay healthcheck en `cache`. La web puede arrancar antes de que Redis responda. | Media | Healthcheck `redis-cli ping` y `condition: service_healthy`. |
| 13 | `redis:latest` sin fijar | **Sí.** `redis:latest` ocupa 213 MB / 57.6 MB frente a `redis:7-alpine` 57.8 MB / 16.7 MB; el tag puede cambiar. | Media | Fijar `redis:7-alpine` o un digest. |
| 14 | `restart: always` | **Sí, pero es menor.** El compose lo declara así; reinicia incluso tras una parada manual. | Baja | Usar `unless-stopped`. |
| 15 | `CLAVE_ADMIN` en el compose en texto plano y sin uso real | **Sí.** `docker compose config` muestra `CLAVE_ADMIN: admin1234`; `grep -R CLAVE_ADMIN app.py templates` no encuentra nada; la app no tiene autenticación y Redis no tiene `requirepass`. | Media (falsa seguridad) | Quitar la variable; si hace falta autenticación, implementarla de verdad y proteger Redis. |

### 2.2. Puntos falsos, irrelevantes o exagerados

**Falso: “Redis corre como root”.**  
`docker compose exec cache id` devuelve `uid=0(root)`, pero eso es el usuario del `exec`, no el del proceso principal. El entrypoint de Redis baja privilegios y `/proc/1/status` muestra:

```text
Name:   redis-server
Uid:    999 999 999 999
Gid:    1000 1000 1000 1000
```

`docker top` muestra el UID 999 (en el host aparece como `dnsmasq`, que es el usuario con ese UID en el host). El proceso Redis corre como `redis` (UID 999), no como root. La web mala sí corre como root.

**Falso: “CLAVE_ADMIN protege el tablón o es una credencial de administrador”.**  
La aplicación nunca lee esa variable:

```text
$ grep -R 'CLAVE_ADMIN' app.py templates
app.py y templates no usan CLAVE_ADMIN
```

No hay login ni comprobación de permisos. Además, Redis no tiene contraseña y el compose malo publica el 6379. `CLAVE_ADMIN=admin1234` es un valor decorativo que no protege nada; lo grave es que esté en la imagen y en el compose, no que sea una autenticación real.

**Irrelevante/exagerado: “sin `EXPOSE 8000` la aplicación no es accesible”.**  
`docker image inspect notas-malo` muestra `ExposedPorts: null`, pero `docker run -p 8001:8000 notas-malo` arranca y `curl -I http://localhost:8001/salud` responde con `Werkzeug`. `EXPOSE` solo documenta el puerto; la publicación real la hace `ports` en Compose o `-p` en `docker run`. Es una mejora de claridad, no un fallo funcional.

**Falso: “la imagen mala no tiene gunicorn”.**  
Gunicorn 22.0.0 está instalado, porque aparece en `requirements.txt`. Lo que falla es que `CMD python app.py` no lo usa. La corrección es cambiar el `CMD`, no instalar otra cosa.

**Exagerado: “sin volumen Redis no arranca”.**  
Redis arranca y responde `PONG`; el volumen solo afecta a la persistencia. Sin volumen, al eliminar el contenedor se pierden las notas. Es un problema de persistencia, no de arranque.

## 3. Evidencia obligatoria

### a) Tamaño de las dos imágenes

Comando de construcción, ejecutado desde la raíz de `recursos/notas`:

```bash
docker build -f sin-revisar/Dockerfile -t notas-malo .
```

Resultado: `Successfully tagged notas-malo:latest`.

Salida de `docker image ls` (Docker 29 distingue uso en disco y tamaño de contenido):

```text
IMAGE               ID             DISK USAGE   CONTENT SIZE
notas-malo:latest   65e1d1c3cc79       1.84GB          475MB
notas-web:latest    d1302e753347        191MB         46.1MB
python:latest       4e324f579bdd       1.64GB          436MB
python:3.12-slim    2259debdc6af        179MB         45.4MB
redis:latest        5c02d7c0f8d7        213MB         57.6MB
redis:7-alpine      858f009f9709       57.8MB         16.7MB
```

Tamaño exacto de las dos imágenes:

```text
notas-web: 46.118.674 bytes  (44,0 MiB)
notas-malo: 475.239.901 bytes (453,2 MiB)
```

La imagen mala es **10,3 veces** más grande en contenido y **9,9 veces** más grande en uso de disco. Solo la base `python:latest` (1,64 GB) es 9,4 veces la base `python:3.12-slim` (179 MB). A eso se suman la capa APT de 131 MB, los 24,7 MB de `pip` y la caché de pip de 2,2 MB.

### b) Usuario que ejecuta cada contenedor

| Contenedor | Usuario real del proceso principal | Evidencia |
|---|---|---|
| Web buena (`salida-web-1`) | `appuser`, UID 10001 | `docker compose exec -T web id` → `uid=10001(appuser)` |
| Web mala (`notas-malo`) | `root`, UID 0 | `docker run --rm --entrypoint id notas-malo` → `uid=0(root)`; `docker top` muestra PID 1 y los workers como root |
| Cache buena (`salida-cache-1`) | `redis`, UID 999; GID 1000 | `docker compose exec -T cache sh -c 'grep ... /proc/1/status'` → `redis-server`, `Uid: 999`, `Gid: 1000` |
| Cache mala (`sin-revisar-cache-1`) | `redis`, UID 999; GID 999 | `/proc/1/status` → `redis-server`, `Uid: 999`, `Gid: 999` |

Matiz importante: `docker compose exec cache id` devuelve `uid=0(root)` en ambos Redis porque `docker exec` usa el usuario configurado de la imagen (vacío = root), no el usuario al que el entrypoint baja el proceso principal. El proceso que realmente atiende Redis es `redis` (UID 999). La web mala sí es root de verdad porque no tiene `USER` ni entrypoint que baje privilegios.

### c) Dónde acaba `CLAVE_ADMIN` y cómo se lee desde fuera

Está en tres sitios:

1. En la imagen, por el `ENV CLAVE_ADMIN=admin1234` del Dockerfile.
2. En el compose, por `environment: - CLAVE_ADMIN=admin1234`.
3. En el contenedor que se cree desde esa imagen, heredado de la imagen o del compose.

Se puede leer desde fuera con:

```bash
docker image inspect notas-malo --format '{{json .Config.Env}}'
docker history --no-trunc notas-malo
docker compose -f sin-revisar/docker-compose.yml config
docker inspect <contenedor>
```

Salida relevante:

```text
["PATH=...","PYTHON_VERSION=3.15.0","PYTHON_SHA256=...","REDIS_URL=redis://cache:6379","CLAVE_ADMIN=admin1234"]
```

`docker history` muestra las capas:

```text
/bin/sh -c #(nop)  ENV CLAVE_ADMIN=admin1234
/bin/sh -c #(nop)  ENV REDIS_URL=redis://cache:6379
```

`docker compose config` muestra `CLAVE_ADMIN: admin1234` en claro. Cualquiera con acceso al daemon de Docker, al fichero del compose o a la imagen publicada puede leerlo. **No se puede leer desde la aplicación por HTTP**: `app.py` y `templates/` no usan esa variable, así que no es una credencial real ni protege el tablón. Si la imagen se publica en un registro, cualquiera que la descargue puede inspeccionar su configuración.

Además, el compose malo publica Redis en `0.0.0.0:6379` sin contraseña, así que `CLAVE_ADMIN` tampoco protege la base de datos. La combinación real es: variable decorativa en la imagen + Redis abierto y sin autenticación.

### d) Servidor web que arranca cada versión

**Versión buena:**

```text
$ docker image inspect notas-web:latest --format 'CMD: {{json .Config.Cmd}}'
CMD: ["gunicorn","--bind","0.0.0.0:8000","--workers","2","--access-logfile","-","app:app"]

$ curl -sI http://localhost:8000/salud
HTTP/1.1 200 OK
Server: gunicorn
```

**Versión mala:**

```text
$ docker image inspect notas-malo --format 'CMD: {{json .Config.Cmd}}'
CMD: ["/bin/sh","-c","python app.py"]

$ curl -sI http://localhost:8001/salud
HTTP/1.1 503 SERVICE UNAVAILABLE
Server: Werkzeug/3.1.9 Python/3.15.0

$ docker logs notas-malo-test
 * Serving Flask app 'app'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment.
 * Running on all addresses (0.0.0.0)
 * Debugger is active!
```

El 503 de `/salud` es porque el contenedor de prueba no tenía Redis; el servidor web sí estaba arrancado. Lo importante es que es el servidor de desarrollo de Flask, con `debug=True` y el depurador activo, y que el `CMD` es formato shell. Gunicorn 22.0.0 está instalado en la imagen, pero no se usa.

### e) Punto falso, irrelevante o exagerado

Se han documentado cinco en la sección 2.2. El más relevante es el de **“Redis corre como root”**: es falso si se mira el proceso principal. `docker exec id` muestra root porque `exec` usa el usuario configurado de la imagen, pero `/proc/1/status` demuestra que `redis-server` corre como `redis` (UID 999). El informe inicial debía corregirse con esa evidencia.

## 4. Correcciones recomendadas

### Dockerfile malo

- Cambiar `FROM python:latest` por `FROM python:3.12-slim`.
- Eliminar `curl`, `vim`, `git` y `python3-pip`; no hacen falta para ejecutar la app.
- Copiar solo `requirements.txt`, instalar con `pip install --no-cache-dir -r requirements.txt`, y después copiar `app.py` y `templates/`.
- Añadir `.dockerignore` con `.git`, `.env`, `sin-revisar`, `__pycache__`, etc.
- Crear un usuario sin privilegios y usar `USER appuser`.
- Cambiar `CMD` a formato JSON con gunicorn:
  `CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]`.
- Añadir `EXPOSE 8000` y `HEALTHCHECK` contra `/salud`.
- Sacar `REDIS_URL` y `CLAVE_ADMIN` de la imagen; no meter secretos.

### Compose malo

- Corregir el contexto: `build: {context: .., dockerfile: sin-revisar/Dockerfile}` o mover el Dockerfile a la raíz del proyecto.
- Quitar `ports: "6379:6379"` de `cache`; Redis solo debe ser accesible por la red interna.
- Añadir un volumen con nombre, por ejemplo `notas-datos:/data`, y `--appendonly yes`.
- Añadir healthcheck a Redis y usar `depends_on: cache: condition: service_healthy`.
- Fijar `redis:7-alpine` en lugar de `redis:latest`.
- Poner `REDIS_URL` y `TITULO_APP` en `environment` del compose; quitar `CLAVE_ADMIN`.
- Cambiar `restart: always` por `restart: unless-stopped`.

## 5. Capturas de la auditoría

Las imágenes están en `../capturas/`:

- `16_tamano_imagenes.png`: `docker image ls`.
- `17_tamano_bytes.png`: tamaño exacto con `docker image inspect`.
- `18_usuario_web_buena.png`: `id` de la web buena.
- `19_usuario_web_mala.png`: `id` de la web mala.
- `20_usuario_cache_buena.png`: usuario real de Redis bueno.
- `21_usuario_cache_mala.png`: usuario real de Redis malo.
- `22_clave_admin_imagen.png`: `CLAVE_ADMIN` en la imagen.
- `23_compose_config_malo.png`: configuración completa del compose malo.
- `24_servidor_bueno.png`: gunicorn y cabecera HTTP.
- `25_servidor_malo_cmd.png`: `CMD` shell de la versión mala.
- `26_servidor_malo_http.png`: Werkzeug, debug y logs.
- `27_compose_build_falla.png`: error de build del compose malo.
- `28_redis_expuesto.png`: puerto 6379 y `PONG` desde fuera.
- `29_volumen_malo.png`: volumen bueno frente a “sin volumen” del malo.
- `30_expose_falso.png`: `ExposedPorts: null` pero puerto accesible.
- `31_clave_admin_no_usada.png`: la app no lee `CLAVE_ADMIN`.
- `32_compose_config_bueno.png`: configuración del compose bueno.
- `33_python_duplicado.png`: Python 3.13 de APT frente a Python 3.15 de la imagen.
- `34_gunicorn_instalado.png`: gunicorn instalado pero no usado.

## 6. Conclusión

La versión mala no es desplegable tal cual: el compose no construye por el contexto, la web corre como root, usa el servidor de desarrollo de Flask con depurador activo, Redis está publicado sin autenticación y no hay volumen de persistencia. La imagen pesa unas 10 veces más que la buena. La versión buena corrige estos puntos con `python:3.12-slim`, usuario `appuser`, gunicorn, healthchecks, Redis privado con volumen y variables en el compose.
