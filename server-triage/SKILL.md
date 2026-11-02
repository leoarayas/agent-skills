---
name: server-triage
description: >
  Usar cuando se diagnostica o resuelve incidentes en servidores de hosting cPanel/WHM
  con CloudLinux, Imunify360 y Apache/LiteSpeed/Nginx. Disparadores típicos: "el sitio
  está caído", "error 500/503", "el servidor está lento", "load muy alto", "se llenó el
  disco", "una cuenta consume mucha CPU/RAM", "falso positivo de ModSecurity/Imunify",
  "malware en una cuenta", "MySQL no responde", "correo no sale (exim)". Es una guía de
  diagnóstico de solo-lectura primero; toda acción destructiva o reinicio de servicio se
  confirma con el usuario antes de ejecutar.
---

# Triage de servidores de hosting (cPanel/WHM + CloudLinux + Imunify360)

Playbook de diagnóstico para servidores de hosting compartido. El objetivo es **aislar la
causa antes de actuar**: primero leer, luego confirmar, luego cambiar.

## Reglas de seguridad (leer siempre primero)

1. **Confirmar el servidor.** Antes de cualquier comando, ejecutar `hostname -f` y mostrar
   el resultado. Si hay duda de en qué host estamos, parar.
2. **Solo-lectura primero.** Diagnosticar con comandos que no modifican estado (los de las
   secciones de abajo lo son, salvo donde se indique).
3. **Acciones destructivas requieren confirmación explícita** del usuario: reiniciar
   servicios, `kill`, suspender cuentas, editar configs, borrar archivos, `cagefsctl
   --remount`, rebuilds. Nunca asumir luz verde.
4. **Una cosa a la vez.** Cambiar una variable, re-medir, y recién entonces el siguiente
   cambio. No encadenar reinicios.
5. **Producción.** Asumir que hay clientes activos. Preferir la acción menos disruptiva
   (reload > restart > rebuild).

## Orden de diagnóstico

```
¿Todo el server o una sola cuenta/sitio?
  └─ Todo el server → Sección 1 (sistema) → 2 (web) → 4 (DB)
  └─ Una cuenta      → Sección 6 (cuenta) → 3 (LVE) → 5 (Imunify)
```

---

## 1. Salud del sistema

```bash
hostname -f                       # confirmar host
uptime                            # load average (comparar con nº de cores: nproc)
nproc                             # cores disponibles
free -m                           # RAM y swap
df -h                             # espacio en disco por partición
df -i                             # INODOS (causa común y olvidada de "disco lleno")
top -bn1 | head -20               # top procesos por CPU
ps aux --sort=-%mem | head -10    # top procesos por RAM
dmesg -T | grep -i -A2 'killed process'   # ¿el OOM killer mató algo?
iostat -xz 1 3                    # I/O wait (paquete sysstat)
ss -tlnp                          # qué escucha en qué puerto
journalctl -p err -n 50 --no-pager
```

Pistas:
- Load alto + `%wa` alto en `top` → cuello de botella de disco/DB, no de CPU.
- `df -h` con espacio libre pero errores de "no space" → revisar `df -i` (inodos).
- Procesos `php-fpm`/`lsphp` disparados → ir a Sección 3 (LVE) para ubicar la cuenta.

## 2. Servicios web (Apache / LiteSpeed / Nginx)

```bash
# Apache (EA4)
httpd -S                          # volcado de vhosts (detecta vhost roto / SSL mal)
httpd -t                          # test de sintaxis ANTES de reiniciar
tail -100 /usr/local/apache/logs/error_log

# LiteSpeed (si es el web server en cPanel)
/usr/local/lsws/bin/lswsctrl status
tail -100 /usr/local/lsws/logs/error.log

# Nginx (standalone o como proxy/Engintron)
nginx -t
tail -100 /var/log/nginx/error.log

# Estado de servicios cPanel
whmapi1 servicestatus | head -40
```

Acción (CONFIRMAR antes):
```bash
/scripts/restartsrv_httpd            # Apache
/scripts/restartsrv_apache_php_fpm   # PHP-FPM
/scripts/restartsrv_lsws             # LiteSpeed
/scripts/rebuildhttpdconf            # reconstruye httpd.conf (solo si está corrupto)
```

## 3. CloudLinux LVE (límites por usuario)

```bash
lvetop                            # uso de LVE en tiempo real, ordena por CPU/IO/EP
lveinfo --period=1d --by-usage=cpu --limit=15   # quién pegó contra sus límites hoy
lveps -p                          # procesos agrupados por LVE/usuario
cloudlinux-selector list --json --interpreter=php   # versiones PHP por selector
cagefsctl --check-usrlocal        # estado CageFS
```

Si una cuenta "no abre" o falta un binario en CageFS (CONFIRMAR antes):
```bash
cagefsctl --enable USUARIO
cagefsctl --update                # tras instalar software nuevo en el host
```

Pistas: si `lveinfo` muestra un usuario chocando con `aCPU`/`aEP`/`aIO`, el sitio se
siente lento porque CloudLinux lo está conteniendo. Decidir entre optimizar el sitio o
subir el package LVE — no reiniciar nada.

## 4. Base de datos (MySQL / MariaDB)

```bash
mysqladmin status
mysqladmin processlist            # consultas activas / bloqueadas
mysql -e "SHOW PROCESSLIST;"
mysql -e "SHOW ENGINE INNODB STATUS\G" | head -60   # deadlocks, esperas
tail -100 /var/lib/mysql/$(hostname).err            # log de error MySQL
du -sh /var/lib/mysql                               # tamaño total
```

Acción (CONFIRMAR antes) — preferir matar una query colgada antes que reiniciar el motor:
```bash
mysql -e "KILL <id>;"             # query específica del PROCESSLIST
/scripts/restartsrv_mysql         # último recurso
```
Nunca editar `/etc/my.cnf` directo en prod sin respaldo; usar un archivo include y
re-medir un cambio a la vez.

## 5. Imunify360 (seguridad / malware / falsos positivos)

```bash
imunify360-agent version
imunify360-agent get-module-status
imunify360-agent doctor                              # diagnóstico del propio agente

# Malware
imunify360-agent malware on-demand start --path /home/USUARIO
imunify360-agent malware malicious list --limit 50

# Falsos positivos / WAF (ModSecurity)
# Revisar incidentes antes de tocar reglas; whitelistear IP legítima si corresponde:
imunify360-agent whitelist ip add 1.2.3.4 --comment "IP oficina cliente X"
imunify360-agent blacklist ip list | head
```

Pistas: si un cliente reporta "no puedo entrar a mi WordPress / me bloquea", primero
revisar si Imunify lo metió en grey/blacklist o si un evento de WAF lo está cortando —
antes de desactivar protecciones. Documentar toda excepción.

## 6. Cuenta / dominio específico (cPanel/WHM)

```bash
whmapi1 accountsummary user=USUARIO
whmapi1 listaccts search=DOMINIO searchtype=domain
whmapi1 php_get_vhost_versions | grep -A2 DOMINIO    # versión PHP del vhost
uapi --user=USUARIO Quota get_quota_info             # uso de disco de la cuenta

# Logs del dominio
tail -100 /etc/apache2/logs/domlogs/DOMINIO          # accesos
tail -100 /usr/local/apache/logs/error_log | grep DOMINIO
```

Acción (CONFIRMAR antes): suspender/reactivar, cambiar versión PHP, ajustar quota.

---

## Rutas de logs de referencia

| Servicio        | Ruta                                              |
|-----------------|---------------------------------------------------|
| Apache (EA4)    | `/usr/local/apache/logs/error_log`                |
| Domlogs         | `/etc/apache2/logs/domlogs/<dominio>`             |
| LiteSpeed       | `/usr/local/lsws/logs/error.log`                  |
| Nginx           | `/var/log/nginx/error.log`                        |
| MySQL/MariaDB   | `/var/lib/mysql/<hostname>.err`                   |
| Exim (correo)   | `/var/log/exim_mainlog`, `/var/log/exim_rejectlog`|
| cPanel          | `/usr/local/cpanel/logs/error_log`                |
| Imunify360      | `/var/log/imunify360/console.log`                 |
| Sistema         | `journalctl -xe`, `/var/log/messages`             |

## Cuándo escalar

- Hardware/I/O degradado a nivel de disco físico o RAID → escalar al proveedor del datacenter.
- Kernel panic / corrupción de filesystem → no improvisar, abrir ticket.
- Ataque activo en curso (DDoS, breach) → contención vía Imunify/firewall + escalamiento.

> Nota: este skill es de diagnóstico. Cualquier comando que cambie estado se confirma
> primero con el operador humano y se ejecuta de a uno.
