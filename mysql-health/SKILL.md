---
name: mysql-health
description: >
  Usar para diagnosticar y ajustar MySQL/MariaDB en servidores cPanel/WHM con
  CloudLinux. Disparadores: "MySQL lento/no responde", "muchas conexiones",
  "consultas colgadas / locks", "slow queries", "subir innodb_buffer_pool",
  "MySQL consume toda la RAM/CPU", "la base no arranca / tabla corrupta", "de qué
  tamaño es cada base". Diagnóstico primero; matar consultas, editar my.cnf o
  reiniciar el motor se confirma antes de ejecutar. En CloudLinux, recordar que
  MySQL Governor también restringe por usuario (ver skill cloudlinux-lve).
---

# Salud y tuning de MySQL/MariaDB (cPanel/CloudLinux)

En este stack hay dos capas que limitan MySQL: la **config del motor** (`my.cnf`) y, por
usuario, **MySQL Governor** de CloudLinux. Si "un cliente" satura la base, suele ser
Governor (skill `cloudlinux-lve`); si el motor entero sufre, es tuning/recursos.

## Reglas de seguridad

1. Confirmar host (`hostname -f`) antes de actuar.
2. Matar consultas, editar `my.cnf`, reiniciar el motor o reparar tablas: CONFIRMAR.
3. No editar `/etc/my.cnf` directo en prod sin respaldo; usar include y cambiar **una
   variable a la vez**, recargar y re-medir.

## 1. Salud rápida

```bash
mysqladmin status
mysqladmin extended-status | grep -Ei 'Threads_connected|Threads_running|Slow_queries|Aborted|Max_used_connections'
mysql -e "SELECT VERSION();"
mysql -e "SHOW GLOBAL STATUS LIKE 'Uptime';"
tail -100 /var/lib/mysql/$(hostname).err          # log de error del motor
```

Pistas: `Threads_running` alto y sostenido = saturación real; `Threads_connected`
pegado a `max_connections` = se rechazan conexiones nuevas.

## 2. Consultas en vivo y bloqueos

```bash
mysql -e "SHOW FULL PROCESSLIST;"
mysql -e "SHOW ENGINE INNODB STATUS\G" | sed -n '1,80p'   # deadlocks, esperas de lock
```

Matar una consulta colgada (CONFIRMAR) — preferir esto a reiniciar el motor:

```bash
mysql -e "KILL <id>;"
```

## 3. Slow query log (encontrar las consultas caras)

```bash
mysql -e "SHOW VARIABLES LIKE 'slow_query%';"
mysql -e "SHOW VARIABLES LIKE 'long_query_time';"
# Habilitar en caliente para muestrear (CONFIRMAR):
mysql -e "SET GLOBAL slow_query_log = 'ON'; SET GLOBAL long_query_time = 2;"
# Analizar:
mysqldumpslow -s t /var/lib/mysql/<hostname>-slow.log | head -30
pt-query-digest /var/lib/mysql/<hostname>-slow.log | head -60   # Percona Toolkit, más detallado
```

## 4. Tuning del motor

```bash
mysqltuner                                  # diagnóstico con recomendaciones (instalar si falta)
mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
mysql -e "SHOW VARIABLES LIKE 'max_connections';"
```

Palancas principales (ajustar de a una, CONFIRMAR):
- `innodb_buffer_pool_size`: la más impactante. Apuntar a contener el working set sin
  invadir la RAM que necesitan Apache/lsphp y el resto del server.
- `max_connections`: subir solo si se rechazan conexiones; subirlo "por las dudas"
  puede agotar RAM.
- `tmp_table_size` / `max_heap_table_size`: si hay muchas tablas temporales en disco.
- `thread_cache_size`: si `Threads_created` crece rápido.
- `query_cache`: depende de la versión (removido en MySQL 8; en MariaDB suele convenir
  off). No asumir; verificar versión antes de tocarlo.

Config en `/etc/my.cnf`. Respaldar antes de editar. Aplicar con reinicio (sección 6).

## 5. Tamaño y espacio

```bash
du -sh /var/lib/mysql
mysql -e "SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024,1) AS MB \
  FROM information_schema.tables GROUP BY table_schema ORDER BY MB DESC LIMIT 15;"
```

## 6. No arranca / corrupción (último recurso)

```bash
tail -200 /var/lib/mysql/$(hostname).err        # leer el motivo exacto primero
mysqlcheck --all-databases --check               # diagnóstico (CONFIRMAR; puede lockear)
```

Si InnoDB no levanta por corrupción, `innodb_force_recovery` (1→6) en `my.cnf` permite
arrancar en modo degradado **solo para volcar datos**; es delicado y se hace con
respaldo y confirmación explícita, nunca como rutina.

Reiniciar el motor (CONFIRMAR — el hook lo pone en ask):

```bash
/scripts/restartsrv_mysql
```

## 7. Cruce con MySQL Governor

Si la presión viene de un usuario puntual, no es tuning del motor: ir a `cloudlinux-lve`
y usar `dbtop`/`dbctl` para ver y, si corresponde, restringir a ese usuario.

> Matar consultas, editar config, reparar tablas o reiniciar el motor se proponen con el
> detalle exacto y se ejecutan solo con OK del operador, de a un cambio por vez.
