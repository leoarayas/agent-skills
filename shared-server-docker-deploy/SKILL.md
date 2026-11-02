---
name: shared-server-docker-deploy
description: Desplegar una app Node (con dependencias nativas o Chromium) en un servidor Docker que ya aloja otras aplicaciones, sin tumbar nada ajeno. Cubre: liberar disco antes de construir, límites de memoria por servicio, healthcheck solo en servicios HTTP, versión de módulos nativos según el Node, publicar detrás de un Caddy compartido con basic auth, y validar/reload del proxy sin downtime. Usar cuando se despliega sobre un servidor existente con otros stacks corriendo, cuando el build falla por espacio en disco, cuando un módulo nativo aborta el proceso, o cuando hay que exponer un servicio nuevo por el proxy que ya existe.
---

# Deploy en servidor Docker compartido

## Cuándo usarla

- El servidor **no es dedicated**: ya corre el despliegue de otra gente o de otros proyectos.
- La app tiene dependencia nativa (`better-sqlite3`, `sharp`, `canvas`) o navegador (Chromium).
- Ya existe un proxy inverso (Caddy o nginx) que publica varios dominios.
- Cualquier `EADDRINUSE`, OOM o disco lleno afectaría a los demás stacks.

Regla de oro: **la app nueva no puede ser la causa de que caiga otra.** Por eso todo va
acotado y medido antes.

---

## 0. Antes de construir: medir y liberar disco

Un build con Chromium baja ~2 GB de imagen. Si el disco está al 78%, el build falla o deja
el servidor al borde y el próximo prune borra imágenes de producción.

```bash
ssh SERVIDOR 'docker system df; df -h /'
```

Lo que suele estar tirado y es seguro liberar:

```bash
docker builder prune -af      # caché de build (suele ser lo más gordo)
docker image prune -af        # imágenes sin usar
docker container prune -f     # contenedores detenidos
```

Antes y después, contar contenedores para confirmar que no se cayó nada:

```bash
docker ps -q | wc -l   # guardar el número, comparar después
```

Medido en un caso real: 78% → 32% recuperados, con los 41 contenedores de producción
intactos.

---

## 1. Módulos nativos y versión de Node

Un `.node` compilado para otra major de Node **no** falla con error claro: aborta el proceso
con un assert interno y un stack nativo inútil.

Síntoma típico:

```
# node.exe[61084]: void __cdecl node::RemoveEnvironmentCleanupHook(Isolate *, CleanupHook, void *)
# Assertion failed: (env) != nullptr
----- Native stack trace -----
```

Causa: `better-sqlite3@11` con Node 24. La soporte llega en `better-sqlite3@12`.

Reglas:

- Verificar la versión del módulo nativo contra la major de Node **antes** de integrar.
- En el `Dockerfile` de build, instalar el toolchain (`python3 make g++`) para que un binario
  precompilado que no exista pueda compilarse en el momento.
- En runtime, **no** asumir que el toolchain hace falta: mejor imagen final más chica.
- Si el módulo se importa desde varios paquetes del workspace, probarlo en un proceso que
  no haga otra cosa: un crash por GC postergado puede aparecer horas después.

---

## 2. Separar el proceso pesado del que responde

Chromium y un módulo nativo en el mismo proceso que el servidor HTTP produce crash y, además,
bloquea el event loop durante los jobs largos.

```text
api     : HTTP + SSE, memoria baja, healthcheck HTTP
worker  : cola + navegador, memoria alta, sin puertos, sin healthcheck HTTP
```

Se comunican por un almacén compartido (SQLite, Redis, Postgres). La API no necesita saber
si el worker está vivo.

**El healthcheck se define solo en el que sirve HTTP.** Heredarlo desde el `Dockerfile` en el
worker lo deja `unhealthy` para siempre, porque nunca abre un puerto:

```yaml
worker:
  healthcheck:
    disable: true
```

---

## 3. Acotar memoria por servicio

```yaml
api:
  mem_limit: 400m
worker:
  mem_limit: 1600m
  mem_reservation: 400m
  volumes:
    - /dev/shm:/dev/shm:rw,size=256m   # Chromium usa /dev/shm; 64 MB lo rompe
  tmpfs:
    - /tmp:size=256m
  security_opt:
    - no-new-privileges:true
```

Además, en el código, reducir el trabajo: el scraper debería tener un `DETAIL_MAX` /
`MAX_PAGES` más bajo en servidor que en local. **Si algún día falta RAM en el servidor, el
culpable es casi siempre ese número, no Docker.**

Medir antes y después de una búsqueda real:

```bash
docker stats --no-stream --format '{{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}' <api> <worker>
free -m | head -2
```

---

## 4. Build: stages y cachés que sí sirven

```dockerfile
FROM mcr.microsoft.com/playwright:v1.63.0-noble AS build   # versión clavada al paquete npm
...
FROM mcr.microsoft.com/playwright:v1.63.0-noble AS runtime
```

- **Clavar la versión** de la imagen a la del paquete npm instalado. Si no coinciden, el
  navegador no está donde Playwright lo busca.
- Copiar primero los `package.json` y después el código, para que `COPY . .` no invalide la
  capa de `install`.
- En runtime, `pnpm`/`npm` hacen falta si los scripts de arranque los usan → `corepack enable`.
- Si `node_modules` se copia desde build y los permisos no coinciden, pnpm intenta
  "repararlo" en cada arranque y falla. Se desactiva con `verify-deps-before-run=false` en un
  `.npmrc` de la imagen.
- Usuario no root en runtime. Las imágenes de Playwright ya traen `pwuser`.

`pnpm-lock.yaml` debe commitearse y el install del build con `--frozen-lockfile`.

---

## 5. Publicar por un proxy que ya existe

Si ya hay un Caddy publicando varios dominios, **no** levantar otro proxy ni abrir otro puerto
público: se agrega un bloque al Caddyfile existente y se une la red Docker.

```yaml
services:
  api:
    networks: [edge_proxy]        # red externa, ya creada
networks:
  edge_proxy:
    external: true
    name: edge_proxy
```

Descubrir la red y dónde vive el Caddyfile:

```bash
docker inspect <proxy> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} rw={{.RW}}{{println}}{{end}}'
```

Editar el archivo **en el host**: si el bind mount es `rw=false`, escribir dentro del
contenedor falla con `Resource busy` o se pierde al reiniciar.

Procedimiento seguro (nunca editar a ciegas):

```bash
cp "$CADDYFILE" "$CADDYFILE.bak-$(date +%Y%m%d-%H%M%S)"   # 1. backup
# 2. agregar el bloque
docker exec <proxy> caddy validate --config /etc/caddy/Caddyfile  # 3. validar
docker exec <proxy> caddy reload  --config /etc/caddy/Caddyfile   # 4. recargar (sin downtime)
```

Si `validate` falla, restaurar el backup. El reload no corta las conexiones existentes.

### Basic auth

```caddyfile
prospector.ejemplo.dev {
	basic_auth {
		leo $2a$14$...        # bcrypt, NUNCA en claro
	}
	reverse_proxy api:3000 {
		flush_interval -1     # imprescindible para SSE
	}
	encode zstd gzip
	header {
		X-Frame-Options DENY
		-Server
	}
}
```

- Generar el hash **en el servidor**: `docker exec <proxy> caddy hash-password --plaintext '...'`.
- Guardar la contraseña en claro solo en un archivo root (`/root/<app>-password.txt`, 600) para
  poder recuperarla o rotarla, y **nunca** en el repositorio.
- No usar el proxy de Cloudflare por delante si la app usa SSE: el timeout de 100 s y el
  buffering cortan el progreso. DNS-only (nube gris) deja que el origen emita el certificado real.
- `basic_auth` protege solo lo que pasa por el proxy. Si la app además se expone por otro
  puerto, queda sin protección: documentarlo.

---

## 6. Desplegar desde Windows

`rsync` no existe en Windows. Lo que sí funciona y es multiplataforma es `tar` por tubería
SSH:

```js
const tar = spawn('tar', ['-cf', '-', '--exclude', './node_modules', ...excludes, '.'], { cwd: ROOT });
const ssh = spawn('ssh', [HOST, `mkdir -p ${DIR} && tar -xf - -C ${DIR}`], { stdio: ['pipe', 'inherit', 'inherit'] });
tar.stdout.pipe(ssh.stdin);
```

`spawn` con `shell: true` es obligatorio para resolver `pnpm` en Windows (es un `.cmd`), pero
entonces **no** se le pueden pasar handles de archivo para redirigir la salida: usar la
variable de entorno `LOG_FILE` y que la propia aplicación escriba el log.

---

## 7. Antes de tocar el servidor compartido

- [ ] `docker ps -q | wc -l` guardado, para comparar al final
- [ ] Disco libre suficiente **después** de liberar
- [ ] `mem_limit` en todos los servicios nuevos
- [ ] El proceso pesado en su propio contenedor, sin puertos
- [ ] Healthcheck solo en quien sirve HTTP
- [ ] Red externa del proxy declarada, no una red nueva
- [ ] Backup del Caddyfile y `caddy validate` antes de reload
- [ ] Log en archivo (el proceso queda sin consola)
- [ ] Script de deploy idempotente: `check / push / build / up / logs / stats / backup`
- [ ] Volumen con backup probado **y restaurado**, no solo creado

## Diagnóstico rápido

```bash
docker stats --no-stream --format '{{.Name}} {{.MemUsage}} {{.CPUPerc}}' <api> <worker>
docker logs --tail 80 <worker>
docker exec <api> node -e "fetch('http://127.0.0.1:3000/api/health').then(r=>r.text()).then(console.log)"
free -m && df -h /
```