---
name: imunify-response
description: >
  Usar cuando hay incidentes de seguridad o falsos positivos de Imunify360 en un
  servidor cPanel/WHM con CloudLinux. Disparadores: "Imunify bloqueó a un cliente",
  "falso positivo de ModSecurity/WAF", "no puedo entrar a mi WordPress / me pide
  captcha", "detectó malware en una cuenta", "IP legítima bloqueada", "quiero
  whitelistear una IP", "un archivo legítimo lo marca como malicioso", "ataque de
  fuerza bruta". Guía de diagnóstico primero; desactivar protecciones o limpiar
  archivos se confirma con el operador antes de ejecutar.
---

# Imunify360 — respuesta a incidentes y falsos positivos

Servidor cPanel/WHM + CloudLinux. El reflejo correcto ante "Imunify me bloquea" es
**entender QUÉ disparó el bloqueo antes de desactivar nada**. Casi nunca la solución
es apagar la protección; es whitelistear lo legítimo y documentar la excepción.

## Reglas de seguridad

1. Confirmar host (`hostname -f`) antes de actuar.
2. No desactivar módulos de seguridad globales para resolver el caso de un cliente.
   Resolver con la excepción más acotada posible (IP, archivo, usuario).
3. Limpieza de malware, restauración y desactivar protecciones: CONFIRMAR antes.
4. Toda excepción (whitelist, regla desactivada) se documenta con motivo y fecha.

## 0. Estado del agente

```bash
imunify360-agent version
systemctl status imunify360            # service imunify360 status en CL6
imunify360-agent doctor                # diagnóstico interno del agente
tail -100 /var/log/imunify360/console.log
```

## 1. Leer el incidente (¿qué bloqueó al cliente?)

El detalle vive en la pestaña **Incidents** del UI; el campo **Sensor** dice el origen.
Bloqueos de Active Response y PAM aparecen como **OSSEC**. Antes de tocar reglas,
identificar: ¿fue WAF (ModSecurity), fuerza bruta (OSSEC/PAM), graylist, o malware?

```bash
# ¿La IP del cliente está en alguna lista?
imunify360-agent ip-list local list --purpose white --by-ip 1.2.3.4
imunify360-agent ip-list local list --purpose black --by-ip 1.2.3.4
```

## 2. IP legítima bloqueada → whitelist

`ip-list` es el comando vigente; `whitelist`/`blacklist` siguen funcionando como alias
en muchas versiones pero están deprecados.

```bash
# Agregar/quitar (forma práctica que sigue operando):
imunify360-agent whitelist ip add 1.2.3.4 --comment "IP oficina cliente X - 2026-06"
imunify360-agent whitelist ip delete 1.2.3.4
imunify360-agent blacklist ip add 9.9.9.9 --comment "scanner abusivo"

# Consultar listas (forma vigente):
imunify360-agent ip-list local list --purpose white
```

## 3. Falso positivo de WAF / ModSecurity

Primero ubicar la regla que disparó (campo del incidente). Si la IP es legítima,
whitelistearla (sección 2) suele bastar. Para ajustar umbrales o reglas:

```
Config principal : /etc/sysconfig/imunify360/imunify360.config
Umbral PAM (BFM) : /etc/pam_imunify/i360.ini
```

No desactivar una ruleset completa por un caso puntual; whitelistear IP o crear la
excepción mínima. Cualquier cambio en estos archivos: CONFIRMAR y respaldar antes.

## 4. Malware en una cuenta

```bash
# Escanear una cuenta (excluir maildir acelera y evita ruido):
imunify360-agent malware on-demand start --path=/home/USUARIO --ignore-mask=/home*/*/mail

# Escanear todas:
imunify360-agent malware on-demand start --path=/home*/*/ --ignore-mask=/home*/*/mail

# Listar archivos maliciosos (límite por defecto 50):
imunify360-agent malware malicious list --limit 100
```

Limpieza (CONFIRMAR antes): revisar la lista de maliciosos y limpiar desde la acción
de Imunify (UI/cleanup) o, si la cuenta está muy comprometida, **restaurar desde un
backup limpio** previo a la infección. Verificar la sintaxis de cleanup vigente en el
servidor antes de aplicarla a ciegas. Tras limpiar: cambiar contraseñas, actualizar
CMS/plugins y re-escanear.

## 5. Archivo legítimo marcado como malicioso → falso positivo

```bash
imunify360-agent submit false-positive --reason "tema propio, código interno" /home/USUARIO/public_html/archivo.php
# Caso inverso (algo malicioso que pasó como limpio):
imunify360-agent submit false-negative /ruta/completa/al/archivo
```

Para que el scanner ignore un archivo/carpeta de forma estable se usa la whitelist del
scanner (JSON con `path`/`reason`/`md5`/`recursive`). Documentar el motivo.

## 6. "Me pide captcha / no puedo entrar"

Suele ser graylist (la IP del cliente quedó en gris por comportamiento sospechoso) o
Active Response. Confirmar con la sección 1 y, si la IP es del cliente, whitelistear.
El captcha invisible y los timeouts de OSSEC (default 600s) se gestionan por config.

## Cuándo NO apagar protecciones

- Si la duda es "apago el WAF para que funcione" → casi siempre es whitelist de IP o
  excepción de archivo, no apagar el módulo.
- Desactivar HardenedPHP o desinstalar Imunify puede romper updates de Apache/PHP:
  nunca como atajo de soporte.

> Acciones que cambian estado (limpiar, restaurar, editar config, desactivar módulos)
> se proponen y se ejecutan solo con OK explícito del operador.
