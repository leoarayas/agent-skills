---
name: cpanel-account-ops
description: >
  Usar para operaciones sobre cuentas en cPanel/WHM: crear, suspender/reactivar,
  cambiar de plan/paquete, cambiar contraseña, cambiar versión de PHP de un dominio,
  revisar/ajustar quota, inspeccionar recursos (correo, bases, cron) y respaldar o
  restaurar una cuenta. Disparadores: "crear una cuenta nueva", "suspender al cliente
  X", "subir/bajar el plan", "resetear la clave de cPanel", "cambiar el PHP del
  dominio", "la cuenta está al límite de disco", "hacer backup de la cuenta", "migrar
  una cuenta". Las acciones que cambian estado se confirman antes de ejecutar.
---

# cPanel/WHM — operaciones de cuenta

Se opera con `whmapi1` (a nivel servidor) y `uapi --user=USUARIO` (en el contexto de
una cuenta). Convención: **identificar la cuenta y mostrar su estado antes de cambiar
nada.** El comando crudo es del lado servidor; el envío por SSH lo define CLAUDE.md.

## Reglas de seguridad

1. Confirmar host (`hostname -f`) y la cuenta exacta antes de cualquier cambio.
2. Crear, suspender, reactivar, cambiar plan/clave, ajustar quota, restaurar:
   CONFIRMAR antes. Mostrar qué cuenta y qué cambio.
3. **Eliminar una cuenta es irreversible**: ver la sección final.

## 1. Identificar la cuenta

```bash
whmapi1 listaccts search=DOMINIO searchtype=domain
whmapi1 listaccts search=USUARIO searchtype=user
whmapi1 accountsummary user=USUARIO          # plan, estado, suspensión, disco
whmapi1 listpkgs                             # paquetes/planes disponibles
```

## 2. Crear cuenta (CONFIRMAR)

```bash
whmapi1 createacct username=USUARIO domain=DOMINIO.com plan=PLAN \
  contactemail=correo@cliente.com password='<CLAVE_FUERTE>'
```

Verificar después con `whmapi1 accountsummary user=USUARIO`. Si no se pasa `plan`, usa
el default; mejor especificarlo.

## 3. Suspender / reactivar (CONFIRMAR — destructivo de servicio)

```bash
whmapi1 suspendacct user=USUARIO reason="Falta de pago - ticket #123"
whmapi1 unsuspendacct user=USUARIO
# Equivalentes por script:
/scripts/suspendacct USUARIO "motivo"
/scripts/unsuspendacct USUARIO
```

## 4. Cambiar de plan / paquete (CONFIRMAR)

```bash
whmapi1 changepackage user=USUARIO pkg=NUEVO_PLAN
```

## 5. Cambiar contraseña de la cuenta (CONFIRMAR)

```bash
whmapi1 passwd user=USUARIO password='<CLAVE_NUEVA_FUERTE>'
```

Avisar al cliente por canal seguro; no dejar la clave en logs ni chats.

## 6. Versión de PHP por dominio

```bash
whmapi1 php_get_vhost_versions                                   # mapa dominio→versión
whmapi1 php_set_vhost_versions vhost=DOMINIO.com version=ea-php82   # CONFIRMAR
```

(En servidores con CloudLinux, las versiones alt-php se ven además con
`cloudlinux-selector list --json --interpreter=php`.)

## 7. Quota / disco

```bash
uapi --user=USUARIO Quota get_quota_info        # uso real de la cuenta
/scripts/fixquotas                              # recalcular quotas (CONFIRMAR)
```

Si el disco "no baja" tras borrar, revisar inodos (`df -i`) y maildirs grandes.

## 8. Inspeccionar recursos de la cuenta

```bash
uapi --user=USUARIO Email list_pops             # cuentas de correo
uapi --user=USUARIO Mysql list_databases        # bases de datos
uapi --user=USUARIO Cron list_cron              # tareas cron
uapi --user=USUARIO DomainInfo list_domains     # dominios/addon/sub
```

## 9. Respaldar / restaurar una cuenta (migración)

```bash
# Empaquetar la cuenta (queda en /home como cpmove-USUARIO.tar.gz):
/scripts/pkgacct USUARIO

# Restaurar un paquete en el servidor destino (CONFIRMAR):
/scripts/restorepkg USUARIO                     # busca el cpmove en /home
```

Para migración: pkgacct en origen → copiar el tar al destino → restorepkg en destino →
verificar DNS/PHP/SSL antes de cortar el origen.

## 10. Eliminar una cuenta — IRREVERSIBLE

```
whmapi1 removeacct user=USUARIO     # y /scripts/killacct
```

El hook de seguridad BLOQUEA estos comandos a propósito. Para borrar de verdad:
1. Respaldo con `/scripts/pkgacct USUARIO` y verificar el tar.
2. Doble confirmación del operador, por fuera del flujo automático.
3. Recién entonces ejecutarlo manualmente.

> Toda acción que cree, modifique o elimine estado de una cuenta se propone con la
> cuenta y el cambio explícitos, y se ejecuta solo con OK del operador.
