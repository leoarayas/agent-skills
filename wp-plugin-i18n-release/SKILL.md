---
name: wp-plugin-i18n-release
description: Prepara un plugin de WordPress hospedado en plugins.svn.wordpress.org para internacionalización completa y publica el release al SVN. Cubre: reescribir strings hardcoded en español a inglés (PHP y JS), generar .pot con wp-cli, crear traducción inicial .po/.mo (ej: español neutro), bumpear versión sincronizada, commit al trunk y crear tag. Usar cuando el plugin fue recién aprobado en WP.org, cuando los strings traducibles están en español y bloquean a otros locales, o cuando se quiere añadir/actualizar traducciones y subir un nuevo release.
---

# WordPress Plugin: i18n completo + Release al SVN de .org

## Cuándo usarla

- Plugin recién aprobado en WP.org que necesita estar listo para translate.wordpress.org.
- Strings traducibles (`__()`, `_e()`, `wp.i18n.__()`, etc.) están hardcoded en español y bloquean a otros locales (fr, de, pt, ja, etc.) en GlotPress.
- Hay que añadir una traducción inicial o un `.pot` por primera vez.
- Cualquier release que bumpea versión y se publica al SVN de plugins.svn.wordpress.org.

## Pre-requisitos

| Componente | Cómo verificar | Si falta |
|---|---|---|
| TortoiseSVN (svn.exe) | `Test-Path "C:\Program Files\TortoiseSVN\bin\svn.exe"` | Instalar desde https://tortoisesvn.net/downloads.html (marcar "command line tools") |
| PHP 8.x en PATH | `php --version` desde un dir no-UNC | Herd legacy está bien; o PHP standalone |
| wp-cli.phar | Descargar de https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar | `Invoke-WebRequest -Uri ... -OutFile wp-cli.phar` |
| Acceso SVN al plugin | Email de aprobación de plugins@wordpress.org con SVN URL, username y password | Solicitar al equipo si falta |

## Workflow completo

1. **Verificar entorno y plugin**
2. **Checkout del SVN** del plugin
3. **Localizar strings traducibles** en código (PHP + JS)
4. **Reescribir strings al inglés** manteniendo semántica técnica
5. **Regenerar `.pot`** con `wp i18n make-pot`
6. **Generar `.po` + `.mo`** de la traducción inicial (ej: español neutro)
7. **Bumpear versión sincronizada** (header `Version`, constante, readme stable tag, CHANGELOG)
8. **Regenerar build limpio** excluyendo archivos de desarrollo (`.distignore`)
9. **Commit al trunk** + **crear tag** en SVN
10. **(Opcional) Sincronizar el directorio local** preservando archivos de Git

---

## 1. Verificar entorno y plugin

```powershell
# SVN en Windows (no suele estar en PATH; usar ruta completa o agregar)
$env:Path = "$env:ProgramFiles\TortoiseSVN\bin;$env:Path"
svn --version

# PHP
php --version
```

Antes de tocar código, **leer el plugin**:
- Plugin header (Name, Version, Text Domain, Domain Path)
- `readme.txt` (Stable tag, Changelog)
- `CHANGELOG.md` (si existe)
- Lista de archivos PHP/JS en `includes/` y `assets/`

## 2. Checkout del SVN

```powershell
$svnUser = '<wp-org-user>'       # username WP.org (case-sensitive)
$svnPass = 'svn_XXXXX...'     # desde "Account & Security" en profiles.wordpress.org
$slug    = 'mi-plugin'        # el slug del plugin
$checkout = "C:\Temp\opencode\$slug-svn"

svn checkout --non-interactive --trust-server-cert `
  --username $svnUser --password $svnPass `
  "https://plugins.svn.wordpress.org/$slug" $checkout
```

SVN ya contiene `trunk/`, `tags/`, `assets/`. **Si acabas de subir el primer release, `tags/` estará vacío y `trunk/` también** (recién creado con el trunk inicial).

## 3. Localizar strings traducibles

Identifica TODAS las llamadas a funciones i18n. Usa `grep` con estos patrones:

```powershell
# PHP
Select-String -Path 'includes/*.php' -Pattern '__\(|_e\(|_x\(|esc_html__\(|esc_attr__\(|esc_html_e\(|esc_attr_e\(|_n\(|_nx\('

# JS (wp.i18n o strings hardcoded sin función)
Select-String -Path 'assets/js/*.js' -Pattern 'wp\.i18n|__\('
Select-String -Path 'assets/js/*.js' -Pattern '"[^"]{6,80}"'   # detecta strings hardcoded
```

**Anti-patrón común**: strings en español dentro de HTML concatenado en JS (ej: `html += '<option>Todas</option>';`). Reescribir a inglés directamente es válido; usar `wp.i18n.__()` es mejor si quieres que sean traducibles en runtime.

## 4. Reescribir strings al inglés

**Reglas estrictas**:

1. SOLO traduce el **primer argumento** de las funciones i18n. NO toques el text-domain (segundo argumento).
2. NO toques nada fuera de una llamada i18n.
3. NO modifiques el formato PHP (espacios, comillas, paréntesis).
4. Términos técnicos como `ExampleStore`, `WooCommerce`, `API`, `Correlation ID`, `Premium`, `JSON`, `HTTP`, `URL`, `CLP` se mantienen en inglés (ya son nombres propios / siglas).
5. PRESERVA placeholders de sprintf/printf (`%s`, `%d`, `%1$s`, etc.) exactamente igual.
6. Si una string ya está en inglés, NO la toques.

**Tabla de traducción recomendada** (ajusta al contexto):

| Español | Inglés |
|---|---|
| "Activo" | "Enabled" |
| "Título" | "Title" |
| "Configurar" | "Configure" |
| "Guardar" | "Save" |
| "Eliminar" | "Delete" |
| "Comuna" | "Commune" |
| "Región" | "Region" |
| "Selecciona una región" | "Select a region" |
| "Sin reglas guardadas" | "No saved rules" |
| "Configura la URL de API y el token" | "Configure API URL and token" |
| "Diagnóstico ejecutado correctamente" | "Diagnostic executed successfully" |
| "No se pudo {verbo}" | "Could not {verbo}" |
| "Debes ingresar X" | "You must enter X" |
| "Cotización API recibida" | "API quote received" |
| "Tarifa por defecto (CLP)" | "Default rate (CLP)" |

Para archivos grandes, **delega a un agente** con instrucciones precisas y la tabla de traducciones.

## 5. Regenerar `.pot` con wp-cli

```powershell
# Desde el directorio del plugin (espejo local)
$wpCli = 'C:\Temp\opencode\wp-cli.phar'
php $wpCli i18n make-pot . "languages/mi-plugin.pot" --domain="mi-plugin"
```

El `.pot` debe estar en `/languages/` con el nombre `<text-domain>.pot`.

## 6. Crear traducción `.po` + `.mo`

**Decisión clave**: ¿qué locale? Para que el plugin sea traducido automáticamente a **todos los hispanohablantes** sin variantes regionales, usa `<text-domain>-es.po` (raíz `es`). WordPress hace fallback en cascada: `es_CL` → `es` → inglés.

Estrategia recomendada:
- **`es.po/.mo`** (raíz) → sirve a es_CL, es_ES, es_MX, es_AR, es_PE, etc.
- (Opcional después) `es_CL.po/.mo` específico para Chile con modismos locales.

**Generar el .po**: usa un script PHP que lea el `.pot` y aplique un array de traducciones EN→ES. Usa `mb_chr(0x00f3)` para caracteres acentuados (PHP en Windows tiene problemas de codepage con literales UTF-8).

```php
// build-es.php
declare(encoding='UTF-8');
$pot = file_get_contents('languages/mi-plugin.pot');
$po_es = 'languages/mi-plugin-es.po';

// Pre-cargar `mb_chr` para caracteres acentuados
function A() { return mb_chr(0x00e1); } // á
function E() { return mb_chr(0x00e9); } // é
function O() { return mb_chr(0x00f3); } // ó
// etc.

$t = [
    "Configure API URL and token" => "Configura la URL de API y el token",
    // ...
];

$header = "msgid \"\"\nmsgstr \"\"\n"
        . "\"Project-Id-Version: Mi Plugin 1.0.0\\n\"\n"
        . "\"Language-Team: Español (neutro) <>\n\"\n"
        . "\"Language: es\\n\"\n"
        . "\"Plural-Forms: nplurals=2; plural=(n != 1);\\n\"\n";

// Parsear .pot, poblar msgstr con $t, escribir .po
// (lógica completa: ver wp-plugin-i18n-release/build-es.php)
```

**Compilar .mo**:

```powershell
php $wpCli i18n make-mo languages/
```

Esto crea automáticamente `languages/<text-domain>-es.mo`.

## 7. Bumpear versión sincronizada

Actualiza **cuatro archivos**:

```php
// 1. my-plugin.php (header)
 * Version: 1.1.0

// 2. my-plugin.php (constante)
define('PLUGIN_VERSION', '1.1.0');

// 3. readme.txt
Stable tag: 1.1.0
/* + entrada nueva en == Changelog == */

// 4. CHANGELOG.md
## [1.1.0] - 2026-09-14
### Changed
- ...
```

## 8. Regenerar build limpio (excluyendo dev files)

El `.distignore` del plugin define qué excluir. Si no existe, usa esta lista estándar:

```powershell
$exclude = @('.git','.github','.gitignore','.gitattributes','.distignore',
             '.idea','.DS_Store','.release-build','bin','node_modules',
             'tests','vendor/bin','phpcs.xml*','phpunit.xml*',
             '.phpunit.result.cache','*.lock','*.log','*.zip',
             'README.md','RELEASING.md','Makefile','.pr-body-*.md')

# Limpiar destino, copiar todo, eliminar excluidos
$src  = 'ruta/espejo/plugin'
$dist = 'ruta/build/plugin'
Copy-Item -Path "$src\*" -Destination $dist -Recurse -Force
foreach ($pat in $exclude) {
  Get-ChildItem -Path $dist -Recurse -Force -Filter $pat -ErrorAction SilentlyContinue | Remove-Item -Recurse -Force
}
```

## 9. Commit al trunk + crear tag

```powershell
$env:Path = "$env:ProgramFiles\TortoiseSVN\bin;$env:Path"

# Reemplazar trunk con build limpio
$trunk = "C:\Temp\opencode\$slug-svn\trunk"
Get-ChildItem -Path $trunk -Force | Where-Object { $_.Name -ne '.svn' } | Remove-Item -Recurse -Force
Copy-Item -Path "$dist\*" -Destination $trunk -Recurse -Force

# svn add + commit
Set-Location $trunk
svn add --force --depth infinity .

svn commit --non-interactive --trust-server-cert `
  --username $svnUser --password $svnPass `
  -m "Release 1.1.0: English source strings, full i18n rewrite, initial Spanish (es) translation"

# Crear tag (svn copy remoto)
svn copy --non-interactive --trust-server-cert `
  --username $svnUser --password $svnPass `
  -m "Tagging 1.1.0" `
  "https://plugins.svn.wordpress.org/$slug/trunk" `
  "https://plugins.svn.wordpress.org/$slug/tags/1.1.0"
```

Tras tag remoto, **forzar actualización local** (los tags nuevos no se materializan completamente en el working copy):

```powershell
svn cleanup
svn update --depth infinity --force --non-interactive --trust-server-cert `
  --username $svnUser --password $svnPass "$trunk/../"
```

## 10. (Opcional) Sincronizar directorio local

Preservar archivos que NO están en el SVN export (`.git/`, `.gitignore`, `tests/`, `bin/`, etc.):

```powershell
$local    = 'C:\Users\...\wp-content\plugins\mi-plugin'
$exportDir = 'C:\Users\...\export-1.1.0'

# Export del tag recién creado
svn export "https://plugins.svn.wordpress.org/$slug/tags/1.1.0" $exportDir

# Eliminar SOLO los archivos del release del local, mantener lo demás
$releaseNames = (Get-ChildItem $exportDir -Force).Name
foreach ($name in $releaseNames) {
  $localPath = Join-Path $local $name
  if (Test-Path $localPath) { Remove-Item $localPath -Recurse -Force }
}

# Copiar el export sobre el local
Copy-Item -Path "$exportDir\*" -Destination $local -Recurse -Force
```

## 11. Verificación final

```powershell
# Versiones en local
Write-Host "header: " (Get-Content "$local\mi-plugin.php" | Select-String -Pattern 'Version:' | Select-Object -First 1).ToString().Trim()
Write-Host "readme: " (Get-Content "$local\readme.txt" | Select-String -Pattern 'Stable tag:' | Select-Object -First 1).ToString().Trim()

# Verificar SVN remoto
svn ls --recursive "https://plugins.svn.wordpress.org/$slug/tags/1.1.0/"
```

---

## Convenciones críticas de WordPress.org

| Concepto | Regla |
|---|---|
| **Text Domain** | Debe ser **idéntico al slug** del plugin (ej: `mi-plugin`). Nunca cambies el slug sin re-aprobar el plugin. |
| **Domain Path** | `/languages/` para que WP.org encuentre el `.pot` |
| **Plugin Name header** | No incluir marcas registradas de terceros al inicio (revisión WP.org rechaza) |
| **readme.txt** | WP.org parsea este archivo (no `README.md`). El `Stable tag` debe coincidir con la versión. |
| **SVN layout** | `/trunk/` (desarrollo), `/tags/X.Y.Z/` (releases), `/assets/` (banners, icon, screenshots) |
| **SVN como release system** | "Solo push ready-to-use versions". No commits de desarrollo频繁es al trunk. |
| **Fallback de traducciones** | WP busca en cascada: `text-domain-locale.mo` → `text-domain.mo` → `text-domain-en_US.mo` (núcleo). El locale raíz `es` sirve a TODAS las variantes `es_XX`. |

## Gotchas y errores comunes

### PowerShell y UNC paths

PHP y muchos binarios fallan si el CWD es un UNC path (`\\server\share`). Antes de correr comandos:

```powershell
Set-Location C:\Users\<usuario>   # o cualquier dir local
php $wpCli ...                # ahora funciona
```

### Codificación de caracteres acentuados en PHP (Windows)

PHP en Windows tiene problemas con literales UTF-8 en scripts. **Solución**: usar `mb_chr(0x00f3)` para cada carácter acentuado:

```php
"Obtener pa" . mb_chr(0x00ed) . "s"  // "Obtener país"
```

PowerShell también puede mostrar `?` en lugar de caracteres acentuados al hacer `Get-Content`. Verifica los **bytes** del archivo, no el render:

```powershell
$bytes = [System.IO.File]::ReadAllBytes('archivo.po')
# Buscar 0xC3 0xB3 (UTF-8 de 'ó')
```

### SVN working copy parece vacío después de un tag nuevo

SVN deduplica archivos entre trunk y tag (mismo "blob"). Tras `svn copy` remoto, los archivos aparecen en el servidor pero el working copy local tiene placeholders. **Solución**: `svn cleanup` + `svn update --depth infinity --force`.

### svn.exe no está en PATH

TortoiseSVN se instala en `C:\Program Files\TortoiseSVN\bin\` pero no agrega `svn.exe` al PATH por defecto. Solución temporal:

```powershell
$env:Path = "$env:ProgramFiles\TortoiseSVN\bin;$env:Path"
```

O agregar permanentemente a PATH del usuario.

### El plugin usa strings en español en el header

Los strings del header del plugin (Plugin Name, Description, etc.) NO son traducibles. Pero si wp i18n los incluye en el `.pot`, déjalos con `msgstr ""` (vacío). WP.org y GlotPress saben que no se traducen.

### No incluir `.po/.mo` para locales gestionados por GlotPress

Una vez que GlotPress tenga traducciones para tu plugin (a través de la comunidad), WP.org automáticamente provee los `.mo`. Tu `.po/.mo` iniciales son **solo el seed inicial**. No luches contra GlotPress.

---

## Archivos modificados por release (checklist)

- [ ] `my-plugin.php` — header Version + PLUGIN_VERSION
- [ ] `readme.txt` — Stable tag + entrada en Changelog
- [ ] `CHANGELOG.md` — nueva entrada
- [ ] `languages/<text-domain>.pot` — regenerado
- [ ] `languages/<text-domain>-es.po` — traducción inicial (opcional)
- [ ] `languages/<text-domain>-es.mo` — compilado (opcional)
- [ ] Todos los strings PHP/JS en inglés

---

## Recursos externos

- Plugin Handbook i18n: https://developer.wordpress.org/plugins/internationalization/
- wp-cli i18n package: https://github.com/wp-cli/i18n-command
- translate.wordpress.org / GlotPress: https://translate.wordpress.org/
- SVN guide WP.org: https://developer.wordpress.org/plugins/wordpress-org/how-to-use-subversion/
- readme.txt validator: https://wordpress.org/plugins/developers/readme-validator/

## Versión

- v1.0.0 — Inicial. Cubre flujo completo de i18n + release SVN en Windows (TortoiseSVN + PowerShell + wp-cli + PHP).