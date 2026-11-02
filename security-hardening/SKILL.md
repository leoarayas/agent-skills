---
name: security-hardening
description: >
  Usar para endurecer y auditar la seguridad de un servidor cPanel/WHM con CloudLinux:
  SSH, firewall CSF/LFD, cPHulk (fuerza bruta), WHM Security Advisor, Tweak Settings,
  2FA, hardening de PHP, puertos de gestión y actualizaciones. Disparadores: "hardening
  del servidor", "asegurar cPanel/WHM", "configurar CSF", "activar cPHulk", "auditar la
  seguridad", "cerrar puertos", "deshabilitar root por SSH", "checklist de seguridad".
  El WAF/antimalware lo cubre imunify-response y el aislamiento cloudlinux-lve. Propone
  cambios; aplicar firewall/SSH/cPHulk se confirma, y SIEMPRE se whitelistea la IP propia
  antes de tocar nada que pueda dejar afuera al operador.
---

# Hardening de servidor cPanel/WHM + CloudLinux

Checklist de postura de seguridad. La regla de oro operativa: **whitelistear tu propia
IP antes de tocar firewall, cPHulk o SSH**, para no dejarte afuera del servidor.

## Reglas de seguridad

1. Confirmar host (`hostname -f`).
2. Antes de habilitar/reiniciar firewall, cPHulk o sshd: confirmar que tu IP está en la
   whitelist y mantener una segunda sesión abierta como red de seguridad.
3. Cambios de firewall, SSH, cPHulk y Tweak Settings: CONFIRMAR. PHP hardening puede
   romper apps: probar en un sitio antes de aplicar global.

## Capas (modelo mental)

- Perímetro → **CSF/LFD** (firewall + detección de logins)
- Fuerza bruta → **cPHulk** (logins de cPanel/WHM/SSH/FTP/correo)
- WAF + malware → **Imunify360** (skill `imunify-response`)
- Aislamiento de cuentas → **CageFS/LVE** (skill `cloudlinux-lve`)
- Recuperación → **backups offsite probados**

## 1. Auditoría inicial (solo lectura)

```bash
ss -tlnp                                   # qué está escuchando y en qué puerto
grep -E '^(PermitRootLogin|PasswordAuthentication|Port)' /etc/ssh/sshd_config
awk -F: '$7 ~ /(bash|sh)$/ {print $1}' /etc/passwd   # usuarios con shell
```

Correr **WHM → Security Center → Security Advisor** y resolver sus advertencias
(parches, permisos, configs inseguras) es el punto de partida más rápido.

## 2. SSH

Objetivo: sin login de root, sin password, y acceso restringido por IP.

```
/etc/ssh/sshd_config:
  PermitRootLogin no
  PasswordAuthentication no        # solo claves
  # (opcional) Port distinto del 22, y AllowUsers acotado
```

CONFIRMAR antes de recargar sshd, con tu clave ya probada y una sesión de respaldo
abierta:

```bash
sshd -t && systemctl reload sshd
```

## 3. Firewall CSF/LFD

```bash
# PRIMERO: asegurar tu acceso
csf -a TU.IP.AQUI "admin - acceso permanente"      # o editar /etc/csf/csf.allow
csf -g TU.IP.AQUI                                  # verificar que estás permitido

csf -l                                             # listar reglas activas
```

Cerrar al exterior los puertos de gestión y permitirlos solo desde IPs de confianza:
cPanel 2082/2083, WHM 2086/2087, Webmail 2095/2096, WebDisk 2077/2078. Config en
`/etc/csf/csf.conf` (respaldar antes). Aplicar (CONFIRMAR — el hook pide confirmación):

```bash
csf -ra                                            # reinicia csf + lfd
```

`csf -x` deshabilita el firewall por completo: úsalo solo a conciencia.

## 4. cPHulk (fuerza bruta)

WHM → Security Center → cPHulk Brute Force Protection. **Whitelistear tu IP antes de
activarlo.** Umbrales razonables: ~5 fallos por IP y ~10 por cuenta en una ventana corta,
bloqueo de ~3600s. Protege SSH, WHM, cPanel, FTP y correo. No mezclar con Fail2Ban:
con cPHulk + CSF/LFD bien configurados, Fail2Ban suele generar conflictos.

## 5. WHM Tweak Settings (endurecer defaults)

- /tmp asegurado (noexec), firma de servidor Apache off, listado de directorios off.
- Shell Fork Bomb Protection on.
- Compiladores deshabilitados para todas las cuentas salvo root.
- mod_userdir tweak on; `open_basedir` protection on.
- Shell deshabilitada para usuarios que no la necesiten (o jailed shell).

## 6. 2FA y PHP

- 2FA obligatorio para WHM y cuentas cPanel de administración.
- PHP (probar antes de aplicar global): `expose_php = Off` y, según necesidad,
  `disable_functions` con funciones peligrosas que tus apps no usen. Verificar que no
  rompa sitios legítimos antes de extenderlo.

## 7. Actualizaciones y backups

- Mantener al día OS, cPanel/WHM, EasyApache 4 y CloudLinux; aplicar parches de
  seguridad pronto (las vulnerabilidades de cPanel se explotan rápido).
- Backups offsite, cifrados y con **restauración probada** periódicamente. Un backup sin
  test de restore no es un backup.

> Este skill propone y audita. Todo cambio que afecte acceso (SSH, firewall, cPHulk) se
> aplica con la IP propia ya whitelisteada, confirmación explícita y una vía de rescate
> abierta.
