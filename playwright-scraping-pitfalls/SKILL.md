---
name: playwright-scraping-pitfalls
description: Errores recurrentes al escribir scrapers con Playwright en pipelines TypeScript/Node: el helper `__name is not defined` al inyectar código en la página, el crecimiento de memoria de Chromium al navegar muchas veces en la misma pestaña, timeouts encadenados en hardware modesto, detección de CAPTCHA sin evasión y extracción de datos que cambian de DOM. Usar cuando se programa un scraper con Playwright, cuando `page.evaluate` falla con `__name is not defined`, cuando Chromium consume cada vez más RAM o cuando las navegaciones empiezan a dar timeout en un servidor.
---

# Scraping con Playwright en pipelines de Node/TypeScript

## Cuándo usarla

- Va a existir un scraper con Playwright (Chromium) dentro de una app Node.
- `page.evaluate` revienta con `ReferenceError: __name is not defined`.
- Chromium sube de memoria conforme avanza el trabajo y al final las navegaciones dan timeout.
- El scraping corre en un servidor compartido y hay que que no tumbe el resto.
- Hay que extraer datos de un sitio cuyo DOM cambia periódicamente.

---

## 1. `__name is not defined` en `page.evaluate`

**Síntoma**

```
page.evaluate: ReferenceError: __name is not defined
    at eval (eval at evaluate (:311:30), <anonymous>:1:113)
```

**Causa.** El código que se pasa a `page.evaluate` se serializa como función y se ejecuta
dentro del navegador. Pero si el archivo está transpilado (esbuild vía `tsx`, ts-node,
swc, babel), el transpiler reescribe las funciones con nombre usando un helper
`__name(...)` que **solo existe en el módulo original**, no en el contexto de la página.

Lo dispara cualquier `const f = () => {}`, un `function` anidada o una clase dentro del
callback.

**Arreglo: código de navegador en JS plano, inyectado como texto.**

```ts
// packages/discovery/src/browser-scripts/extract-detail.js  (JS plano, sin imports)
function extractDetail() {
  var el = document.querySelector('button[data-item-id="phone"]');
  return { phone: el ? el.getAttribute('aria-label') : null };
}

// packages/discovery/src/browser-scripts.ts
import { readFileSync } from 'node:fs';
function read(name: string) {
  return readFileSync(path.join(scriptsDir, name), 'utf8');
}
export const browserScripts = {
  extractDetail: `(() => { ${read('extract-detail.js')}\n return extractDetail(); })()`,
};

// uso
const detail = await page.evaluate<RawDetail>(browserScripts.extractDetail);
```

Reglas para esos archivos:

- `function` declaradas a nivel superior (para invocarlas al final de la cadena).
- Sin `import`/`export` ni sintaxis moderna que el navegador destino no soporte.
- Envolver en `(() => { ... return fn(); })()` para que quede una expresión.
- ESLint: en `**/*.js` de esa carpeta hay que desactivar `no-undef` y
  `no-unused-vars`, porque ahí sí existen `document` y `window`.

**Variante alternativa**: construir el contexto de una vez y leerlo en cada iteración, en
vez de reinyectar el script.

---

## 2. Chromium crece y luego todo da timeout

**Síntoma.** El job empieza bien y, tras N navegaciones, aparecen `TimeoutError: page.goto`
y la RAM del proceso sube hasta el tope.

**Causa.** Reutilizar la misma pestaña para muchas navegaciones: cada documento deja
estructuras y el proceso de render no libera igual de rápido. Con ~40 navegaciones de Google
Maps se midió **1.5 GB** y las navegaciones fallaron a los 45 s.

**Arreglo: una pestaña por navegación, cerrada al terminar.**

```ts
let detailPage: Page | null = null;
try {
  detailPage = await page.context().newPage();
  await detailPage.goto(url, { waitUntil: 'domcontentloaded' });
  const data = await detailPage.evaluate<Raw>(browserScripts.extract);
} finally {
  await detailPage?.close().catch(() => undefined);
}
```

Efecto medido en el mismo trabajo: de 1.56 GB (pegado al límite) a **~350 MB en reposo**.

El contexto del navegador (`browser.newContext()`) sí se reutiliza; solo la `Page` es
desechable.

**Otros topes útiles en Docker**

```yaml
# Chromium usa /dev/shm; el default de Docker (64 MB) provoca crashes.
volumes:
  - /dev/shm:/dev/shm:rw,size=256m
mem_limit: 1500m
tmpfs: ["/tmp:size=256m"]
```

---

## 3. Timeouts: por negocio, no por navegación

`PLAYWRIGHT_TIMEOUT` aplica a cada operación. En un job con muchas navegaciones el tiempo
total se multiplica. Opciones:

- Subir el timeout por navegación **y** acotar cuántas hay (`DETAIL_MAX`).
- Poner un timeout propio solo en la fase cara (las fichas individuales), y uno corto en la
  fase de listado.
- Registrar cada fallo con `url` + `error` y **seguir**: un registro roto no debe tumbar
  el job.

---

## 4. CAPTCHA y bloqueos: se registran, no se esquivan

Detectarlo y fallar explícitamente es la decisión correcta cuando el trabajo es de bajo
volumen y legítimo:

```ts
const CAPTCHA_SELECTORS = ['form#captcha-form', 'iframe[src*="recaptcha"]'].join(', ');
const CAPTCHA_TEXTS = ['unusual traffic', 'tráfico inusual', 'not a robot'];

// Ojo: el motor `text=` no se puede mezclar con CSS en locator.count.
const visible = await page.locator(CAPTCHA_SELECTORS).first().isVisible({ timeout: 1500 });
if (visible) throw new DiscoveryBlockedError('captcha', 'bloqueo de Google');

const body = (await page.locator('body').innerText()).toLowerCase();
if (CAPTCHA_TEXTS.some((t) => body.includes(t))) throw new DiscoveryBlockedError('captcha', '...');
```

Lanzar un error de dominio propio (`DiscoveryBlockedError`) permite que la capa de arriba
guarde el motivo real en la base, en vez de un `Error: undefined`.

No implementar: rotación de IP, proxies, resolución de CAPTCHA, evasión de fingerprint.

---

## 5. Selectores frágiles: ponerlos en un solo archivo

Un scraper serio tiene **todos** sus selectores en un módulo, con la URL de cada uno y
comentado cuándo se verificó por última vez. Todo lo demás consume funciones de ese módulo.

Cuando el sitio cambia, el flujo de reparación es:

1. Correr el extractor en modo `smoke` y ver qué campos salen vacíos.
2. Volcar el HTML real a un archivo (script `dump`).
3. Volcar la estructura legible: `aria-label`, `data-item-id`, textos hoja (`inspect`).
4. Ajustar **solo** el archivo de selectores.
5. Cubrir la lógica con tests que usen fixtures HTML locales, para que el suite no dependa
   del sitio real.

Preferencia de estabilidad, de más a menos durable:

| Tipo | Ejemplo |
| --- | --- |
| Atributo de datos | `[data-item-id="phone:tel:+569..."]` |
| Rol + aria-label | `[role="img"][aria-label*="estrella"]` |
| Clase con nombre legible | `.fontHeadlineSmall` |
| Clase ofuscada | `.UsdlK` (Google la cambia periódicamente) |

---

## 6. Si el valor no está en el DOM, dejarlo vacío

Es tentador "deducir" el dato con un regex sobre el texto plano. Si la heurística no es
fiable, produce números falsos con apariencia de dato real: peor que no tener el campo.

En vez de `(\d+)` sobre el `innerText` para contar reseñas (acaba devolviendo el primer grupo
de un teléfono), devolver `null` y documentar por qué en `docs/scraping-notes.md`.

---

## Checklist antes de dar por bueno un scraper

- [ ] El código de página va en JS plano inyectado como texto (sin `__name`).
- [ ] Una pestaña por navegación, cerrada en `finally`.
- [ ] `/dev/shm` montado si corre en Docker.
- [ ] CAPTCHA se detecta y se falla con un motivo legible.
- [ ] Un registro roto no detiene el job.
- [ ] Todos los selectores en un archivo, con fecha de verificación.
- [ ] Tests con fixtures HTML locales; el suite no toca el sitio real.
- [ ] Campos no disponibles: `null`, no inventados.
- [ ] Delays aleatorios entre navegaciones.
- [ ] Un modo `smoke` para ver qué devuelve hoy el scraper.
- [ ] `Límite` de navegaciones por job, para acotar RAM y tiempo.