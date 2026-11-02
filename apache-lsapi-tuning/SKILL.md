---
name: apache-lsapi-tuning
description: >
  Usar para diagnosticar y ajustar el rendimiento de PHP en Apache con mod_lsapi
  (lsphp) sobre CloudLinux/cPanel. Disparadores: "PHP lento bajo Apache", "errores 503
  / Connect to backend rejected", "508 Resource Limit Reached", "ajustar
  lsapi_backend_children", "activar Connection Pool / pool_mode", "TTFB alto",
  "tuning de OpCache de lsphp", "504/timeout de PHP", "subir workers de lsphp". No
  cubre PHP-FPM ni LiteSpeed Web Server. Diagnóstico primero; editar lsapi.conf y
  reiniciar Apache se confirma antes de ejecutar.
---

# Tuning de Apache + mod_lsapi (lsphp) en CloudLinux

mod_lsapi corre PHP vía el protocolo LSAPI: Apache delega a procesos `lsphp` por
VirtualHost. **No hay pools de PHP-FPM acá.** El rendimiento se gobierna con los
directivos `lsapi_*` y, sobre todo, con su interacción con los límites de LVE
(skill `cloudlinux-lve`).

## Reglas de seguridad

1. Confirmar host (`hostname -f`) antes de tocar nada.
2. Editar `lsapi.conf`, reiniciar/reconstruir Apache: CONFIRMAR y **respaldar el archivo
   antes** (`cp lsapi.conf lsapi.conf.bak-FECHA`).
3. Un directivo a la vez, recargar, y re-medir. No cambiar children + idle + pool a la
   vez: no sabrías qué movió la aguja.

## 1. ¿Está realmente usando mod_lsapi?

La forma rápida: en un `phpinfo()` de un sitio, el campo **Server API** debe decir
**LiteSpeed**. A nivel servidor el handler de PHP se asigna en WHM → MultiPHP Manager
(o EasyApache 4); no se cambia editando vhosts a mano en cPanel.

## 2. Dónde se configura

```
Config global de tuning : /etc/apache2/conf.d/lsapi.conf
Binarios lsphp          : /opt/alt/phpXX/usr/bin/lsphp  (alt-php) o ea-phpXX
```

OJO en cPanel: el handler por dominio lo administra MultiPHP/EA4; un rebuild puede
regenerar configs. Los directivos `lsapi_*` globales van en `lsapi.conf`, que se incluye
en Apache vía `Include`.

## 3. Directivos clave de tuning

| Directivo | Default | Qué hace |
|---|---|---|
| `lsapi_engine` | On | Activa el motor LSAPI. |
| `lsapi_backend_children` | 80 | Máx. procesos lsphp hijos simultáneos. Subir acelera la respuesta (a costa de RAM/NPROC). |
| `lsapi_backend_max_idle` | 300 | Seg. que un hijo espera nueva request antes de morir. Subir = mejor rendimiento sostenido. |
| `lsapi_backend_pgrp_max_idle` | 30 | Seg. que el proceso de control espera antes de salir. Subir = respuesta más rápida; bajar = menos memoria. |
| `lsapi_backend_max_reqs` | — | Cuántas requests atiende un hijo antes de auto-reciclarse (mitiga fugas de memoria). |
| `lsapi_backend_max_process_time` | 300 | Tiempo máx. de procesamiento por request (seg). **Debe coincidir con `max_execution_time` del php.ini.** |
| `lsapi_backend_connect_timeout` | 100000 | Timeout de conexión al backend. |
| `lsapi_debug` | Off | Activar solo para depurar; genera mucho log. |

Subir `lsapi_backend_children` es la primera palanca ante "PHP lento por saturación",
pero ver la sección 5 antes: el techo real puede ser LVE, no este directivo.

## 4. Connection Pool (pool_mode)

Por defecto cada request crea/usa un lsphp que muere al terminar. Con pool mode, los
lsphp quedan vivos (hasta `lsapi_backend_max_idle` o `lsapi_backend_max_reqs`), mantienen
conexión persistente con Apache y un **OpCache centralizado por dominio**: gran mejora de
TTFB y throughput. El costo: **más procesos por LVE (NPROC) y más memoria.**

Activar (CONFIRMAR — respaldar antes):
```bash
# en /etc/apache2/conf.d/lsapi.conf, descomentar y poner On:
#   lsapi_with_connection_pool On
```

Alternativa: **CRIU**, más rápido y con menos memoria que el pool (CL7/CL8). Si la RAM
es el cuello, evaluar CRIU en lugar de pool.

## 5. La relación crítica con LVE

Hay **un proceso PHP padre por VirtualHost, no por cuenta**, y no se puede limitar
directamente. Para contener a un usuario hay que **combinar `lsapi_backend_children` con
los límites NPROC/EP de LVE** (skill `cloudlinux-lve`). Activar Connection Pool sube el
NPROC consumido: si los packages LVE quedan cortos, los sitios empezarán a pegar contra
el límite aunque "sobre" RAM en el server.

Regla práctica: cualquier cambio de pool o de children se valida mirando `lvetop` después.

## 6. OpCache para lsphp

OpCache se configura en el `php.ini` de la versión correspondiente (alt-php o ea-php).
Con Connection Pool, el OpCache se comparte por dominio entre las requests del worker
vivo, así que el beneficio de OpCache se nota mucho más con pool activado.

## 7. Diagnóstico por síntoma

```bash
tail -200 /usr/local/apache/logs/error_log     # errores de Apache/lsapi
```

| Síntoma | Causa probable | Acción |
|---|---|---|
| `503` + "Connect to backend rejected" en error_log | Backend saturado (REJECT mode) | Subir `lsapi_backend_children`; revisar NPROC de LVE (sección 5). |
| `508 Resource Limit Is Reached` | Límite de LVE alcanzado | Ir a `cloudlinux-lve`: optimizar el sitio o subir el package. |
| `500`/`504` tras X segundos | Tope de tiempo | Alinear `lsapi_backend_max_process_time` con `max_execution_time`. |
| TTFB alto pero sin errores | Workers fríos | Evaluar Connection Pool o CRIU. |

## 8. Aplicar cambios

```bash
httpd -t                              # validar sintaxis de Apache ANTES
/scripts/restartsrv_httpd             # reiniciar Apache (CONFIRMAR; el hook lo pone en ask)
```

> Toda edición de `lsapi.conf` y todo reinicio de Apache se proponen, se respaldan y se
> ejecutan solo con OK del operador, de a un cambio por vez.
