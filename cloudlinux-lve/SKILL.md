---
name: cloudlinux-lve
description: >
  Usar para gestionar límites de recursos y aislamiento en CloudLinux: LVE (CPU/EP/IO
  por usuario), MySQL Governor (límites de base de datos), CageFS y PHP Selector
  (alt-php vs EA-PHP). Disparadores: "un cliente consume mucha CPU/IO", "el sitio está
  lento y choca contra sus límites", "subir/bajar el package LVE", "restringir MySQL de
  un usuario", "falta un binario/módulo dentro de CageFS", "cambiar la versión de PHP de
  un usuario", "el cliente pega contra Entry Processes". Diagnóstico primero; cambiar
  límites o remontar CageFS se confirma antes de ejecutar.
---

# CloudLinux — LVE, MySQL Governor, CageFS y PHP Selector

En hosting compartido la lentitud de "un sitio" casi siempre es **un usuario chocando
contra sus límites de LVE o de MySQL Governor**, no el servidor entero caído. El reflejo
correcto: medir qué recurso topa, y recién entonces decidir si optimizar el sitio o
subir el package. Va de la mano con el skill `server-triage`.

## Reglas de seguridad

1. Confirmar host (`hostname -f`) y el usuario exacto antes de cambiar límites.
2. Cambiar límites LVE/Governor, restringir usuarios, remontar/force-update de CageFS
   y cambiar versión de PHP: CONFIRMAR antes.
3. Un cambio a la vez y re-medir. Subir límites a la ligera traslada el problema a todo
   el servidor.

## 1. Medir el uso (qué recurso topa)

```bash
lvetop                              # uso LVE en tiempo real por usuario
lveinfo --period=1d --by-usage=cpu --limit=15   # quién pegó contra sus límites hoy
lveps -p                            # procesos/threads dentro de cada LVE
dbtop                               # uso de MySQL por usuario en tiempo real (dbtop -r12 = refresco 12s)
```

Lectura: en `lvetop`/`lveinfo`, columnas clave: CPU (speed), EP (entry processes), PNO
(procesos), IO, IOPS. En `dbtop`, fila roja = usuario restringido, azul = en enfriamiento;
la columna CAUSE dice por qué y cuántos segundos faltan para liberarlo.

## 2. Límites LVE por usuario

```bash
lvectl list                         # todos los LVE y sus límites
lvectl limits USUARIO               # límites de un usuario (o por LVE id)
```

Ajustar (CONFIRMAR). `--speed` reemplazó al viejo `--cpu` porcentual:

```bash
lvectl set USUARIO --speed=100% --pmem=1G --ep=20 --nproc=100 --io=2048 --iops=1024
lvectl set USUARIO --speed=0      # 0 = ilimitado para ese recurso
```

## 3. Límites por paquete (package)

Los packages LVE suelen mapear a los paquetes de cPanel. Mejor ajustar el package que
usuario por usuario:

```bash
lvectl package-set NOMBRE_PKG --speed=200% --pmem=2G --ep=30 --io=4096
lvectl apply all                    # aplica los límites de package a todos (CONFIRMAR)
```

## 4. MySQL Governor (dbctl)

Cada límite lleva 4 valores por ventana de tiempo: 1 min, 15 min, 1 h, 1 día.

```bash
dbctl list                          # límites actuales por usuario
dbctl list-restricted               # quién está restringido ahora

# Ajustar (CONFIRMAR):
dbctl set USUARIO --cpu=150,100,70,50 --read=2048,1500,1000,800 --write=2048,1500,1000,800

# Forzar/quitar restricción (CONFIRMAR):
dbctl restrict USUARIO
dbctl unrestrict USUARIO
```

Config en `/etc/container/mysql-governor.xml` (editar con `dbctl`, no a mano); si se
toca el XML, reiniciar con `service db_governor restart`.

## 5. CageFS (aislamiento del filesystem)

```bash
cagefsctl --list-enabled
cagefsctl --check-usrlocal
```

Acciones (CONFIRMAR — el hook pedirá confirmación en remount/force-update):

```bash
cagefsctl --enable USUARIO          # habilitar CageFS para un usuario
cagefsctl --update                  # tras instalar software nuevo en el host
cagefsctl --force-update            # regenerar skeleton (más pesado)
cagefsctl --remount-all             # remontar todos (último recurso)
```

Síntoma típico: "falta un comando/módulo dentro de la cuenta" → suele resolverse
agregando el binario al perfil de CageFS y un `--update`.

## 6. PHP Selector (alt-php) vs EA-PHP

Dos formas de manejar versiones de PHP conviven:
- **PHP Selector (alt-php)**: el usuario elige versión + extensiones + php.ini propio.
- **MultiPHP / EA-PHP**: gestionado por cPanel/EasyApache (ver skill `cpanel-account-ops`).

```bash
cloudlinux-selector list --json --interpreter=php                 # versiones disponibles
cloudlinux-selector get  --json --interpreter=php --user=USUARIO  # versión actual del usuario
cloudlinux-selector set  --json --interpreter=php --user=USUARIO --version=8.2   # CONFIRMAR
```

Si el usuario está en alt-php, su versión NO se cambia con `php_set_vhost_versions` de
cPanel; se cambia acá. No mezclar ambos métodos para el mismo usuario.

## 7. Inodos y disco

```bash
df -i                               # inodos por partición (causa oculta de "disco lleno")
/usr/bin/cl-quota -u USUARIO        # inodos usados/límite del usuario
```

## Cruce con el triage

Si en `server-triage` un usuario aparece arriba en `lvetop`/`dbtop`, este es el skill
para decidir la acción: optimizar el sitio (cache, queries) vs. subir el package. Nunca
subir límites como reflejo sin mirar el costo para el resto del servidor.

> Todo cambio de límites, restricción o versión de PHP se propone con usuario y valores
> explícitos, y se ejecuta solo con OK del operador.
