# CLAUDE.md — cotizador-software (SW, imos)

## AYJ Maquinaria — Ecosistema de Cotizadores

Este repo es parte de un ecosistema de 5 cotizadores para AYJ Maquinaria S.A.S. (35+ años, Bogotá y Medellín). Cada uno es un **single-file HTML** (~5.000-6.000 líneas, HTML+CSS+JS vanilla, sin frameworks) que se despliega directo con GitHub Pages.

| Cotizador | Repo | Estado |
|---|---|---|
| MQ V1 — Máquinas (estable) | aristi01AYJ/cotizador-ayj | producción |
| MQ V2 — Máquinas (desarrollo) | aristi01AYJ/cotizador-ayj-v2 | desarrollo activo |
| SW — Software imos | aristi01AYJ/cotizador-software | producción |
| ADH — Adhesivos | aristi01AYJ/cotizador-adhesivos | producción |
| SVC — Servicios | aristi01AYJ/cotizador-service | producción |

**Regla de sincronización:** en Máquinas, V2 es donde se desarrolla y prueba; cuando un fix o feature está estable, se replica en V1 (mismo cambio, mismo archivo `index.html`).

### Stack técnico
- Frontend: HTML + CSS + JS vanilla, un solo archivo `index.html` por cotizador
- Auth: MSAL.js + Azure AD (Entra ID)
- Backend de datos: Microsoft Graph API v1.0 → listas de SharePoint Online
- Hosting: GitHub Pages
- TRM: BanRep (datos.gov.co), con fallback a open.er-api.com (ADH usa BanRep × 1.15)

### ⚠️ Credenciales — NUNCA hardcodear en este repo (es público)
Este repositorio es **público** (GitHub Pages gratis). El token de GitHub, el Azure AD App ID, los `siteId` de SharePoint y la clave del módulo MG viven en el documento de handoff privado del Proyecto de Claude **"APP PARA AYJ - COTIZADOR"** — pídeselos a Adolfo o consulta ese documento, nunca los pegues en código ni los subas a este repo. Si necesitas guardarlos localmente para trabajar, usa un `.env` o similar que esté en `.gitignore`.

### Workflow de edición de código
1. Editar `index.html` localmente (o vía Claude Code con el repo clonado).
2. **Validar el JS antes de cualquier commit**: extraer el bloque `<script>...</script>` principal y correr `node --check`. Nunca subir sin este paso.
3. Commit descriptivo, push a `main` (GitHub Pages se actualiza solo).
4. Si el fix aplica también a la otra versión de Máquinas (V1 ↔ V2), replicarlo ahí igual.
5. Actualizar el handoff del Proyecto de Claude con el fix aplicado (fecha, archivo, línea, qué cambió) para que la próxima sesión tenga contexto.

### Reglas críticas Graph API / SharePoint (aplican a todos)
- NUNCA usar `$select=fields` en queries → error 400. Usar `expand=fields&$top=500`.
- Campo NIT interno: `NIT_x002f_RUT` — leer con `.toString().trim()`.
- Campos tipo Hyperlink (LinkPDF, Ficha Técnica): `{Url:"...", Description:"..."}`.
- Clientes/Contactos: SIEMPRE en el `siteId` de Comercial (todos los cotizadores, incluso SVC).
- **SIEMPRE usar `graphGetAll()` (pagina con `@odata.nextLink`) para cualquier query que pueda superar 500 ítems** — especialmente `generarNumOferta()`. `graphGet()` con `$top=500` solo trae la primera página; una vez la lista de Cotizaciones supera 500 registros, el consecutivo se queda pegado para siempre porque deja de ver los ítems más nuevos (bug real, 31-ago-2026, afectó los 5 cotizadores).


## Este repo específico: Software imos

- Moneda: **EUR**, no USD (a diferencia del resto del ecosistema).
- `editarOferta` restaura condPago, validez, descuento, TCE e items — soporta formato compacto `{n,t,c,tl}` y formato expandido en `historialData`.
- Liquidador imos con botón 🖨 PDF propio.
- `actualizarNumOferta` está definida localmente en este archivo (no la busques en un módulo compartido).
- `guardarCliente` usa el `siteId` de Comercial — si llega vacío, primero llama `await cargarListaContactos()`.

### ⚠️ Este repo está MUY atrasado frente a Máquinas (MQ V1/V2) — portando por fases
A 22-sep-2026, el usuario pidió portar 3 fases: **(1) Historial + Dashboard, (2) Clientes/Gerente de Cuenta, (3) Posible Cierre + Seguimiento** (dejó fuera, por ahora, la firma digital del PDF). Progreso:

**✅ Ya portado (22-sep-2026):**
- `asesorLabel()` — y con él, el fix del bug de "vendedores con mismo primer nombre mezclados" en 3 sitios: `editarProbabilidadEstrellas()`, `borrarOferta()`, y el chequeo de Editar/Borrar del Historial.
- Historial: Vista Previa de ítems (`previewItems()`), Estado como `<select>` inline de 3 opciones (reemplaza el modal `cambiarEstado`), Probabilidad como 5 estrellas clicables (reemplaza el `prompt()` de `editarProbabilidad`), filtro de N° Oferta (`hFiltNum`), y la herramienta "🧹 Duplicados" completa (`abrirLimpiezaDuplicados()` + 2 pasadas de detección).

**⏳ Pendiente — resto de fase 1 (Dashboard):**
- Cross-filter interactivo (`dashFiltro`, clic en gráfico filtra el resto) — SW sigue con 4 gráficos estáticos, sin interacción.

**⏳ Pendiente — fase 2 (Clientes):**
- Gerente de Cuenta ("Mis Clientes" / asignar / reasignar) + "Ofertas a este cliente" en la ficha.

**⏳ Pendiente — fase 3 (Posible Cierre + Seguimiento):**
- Posible Cierre (30/60/90) + Fecha de Seguimiento + pantalla de Seguimiento + edición desde Historial. Necesita 2 columnas nuevas en la lista `cotizaciones_sw` de SharePoint: `FechaCierre` (Fecha y hora) y `PosibleCierre` (Una línea de texto) — mismos nombres que en Máquinas.

**Fuera de alcance por ahora:** firma digital del cliente en el PDF, link de firma externa, agrupar revisiones -Rn ("Solo vigentes" — SW no tiene el concepto de revisiones; `editarOferta` no genera `-Rn`, hay que confirmar primero si sobreescribe o duplica antes de portar esto).

Ver `cotizador-ayj/CLAUDE.md` para el detalle de cómo funciona cada feature en Máquinas (la fuente de la que se está portando). **No portar todo de una sola vez** — se está avanzando por fases, cada una con su propio commit y prueba.

### Fixes recientes
- **22-sep-2026 — 3 bugs reportados por el usuario: consecutivo pegado, duplicar y probabilidad "no funcionan".** Diagnóstico:
  - `editarProbabilidad()` y el chequeo de "es mi oferta" en el Historial usaban el mismo patrón fragil que causó el bug de Jorge Pulido/Jorge Salamanca en Máquinas (`currentUserName.includes(vendedor.split(' ')[0])`) — con 2 vendedores de nombre "Jorge" esto los mezcla, incluyendo en el permiso de editar. Se portó `asesorLabel()` de Máquinas y se usa en ambos lugares. Confirmado con prueba simulada: antes Jorge Pulido podía editar la probabilidad de la oferta de Jorge Salamanca; ahora se bloquea correctamente.
  - `duplicarOferta()` no tenía un mensaje claro cuando la oferta no tenía ítems guardados (`Items` vacío o mal parseado) — ahora avisa explícitamente en vez de fallar en silencio.
  - **El consecutivo pegado NO se pudo reproducir/confirmar desde este entorno** (no hay login real a SharePoint) — `generarNumOferta()` y `graphGetAll()` en sí mismos están bien (paginan correctamente, igual que en Máquinas). La sospecha más probable es que `resolverSite()` no está encontrando `listIdCotizaciones` (usa comparación EXACTA `n==='cotizaciones_sw'` contra el nombre de la lista en SharePoint — si el nombre real difiere aunque sea un espacio, esto nunca hace match y el consecutivo se queda pegado en 001 para siempre). Se agregó diagnóstico en consola (`console.log`/`console.warn`) en `resolverSite()` y `generarNumOferta()` — la próxima vez que pase, revisar la consola del navegador (F12) y confirmar si `listIdCotizaciones` sale "NO ENCONTRADA" y qué nombres de lista aparecen realmente.
- **31-ago-2026 — Consecutivo de cotización pegado.** `generarNumOferta()` usaba `graphGet()` sin paginar; se cambió a `graphGetAll()`. Ver regla en la sección de Graph API arriba.
