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

### ⚠️ Este repo estaba MUY atrasado frente a Máquinas (MQ V1/V2) — se portó por fases
A 22-sep-2026, el usuario pidió portar 3 fases: **(1) Historial + Dashboard, (2) Clientes/Gerente de Cuenta, (3) Posible Cierre + Seguimiento** (dejó fuera, por ahora, la firma digital del PDF). **Las 3 fases quedaron portadas y probadas el mismo día** (con datos simulados en el Browser pane — sin login real a SharePoint en este entorno, ver detalle de cada fase abajo). Progreso:

**✅ Ya portado (22-sep-2026):**
- `asesorLabel()` — y con él, el fix del bug de "vendedores con mismo primer nombre mezclados" en 5 sitios: `editarProbabilidadEstrellas()`, `borrarOferta()`, el chequeo de Editar/Borrar del Historial, y los 2 gráficos "por asesor" del Dashboard (antes usaban `.split(' ')[0]`/`.startsWith()`).
- Historial: Vista Previa de ítems (`previewItems()`), Estado como `<select>` inline de 3 opciones (reemplaza el modal `cambiarEstado`), Probabilidad como 5 estrellas clicables (reemplaza el `prompt()` de `editarProbabilidad`), filtro de N° Oferta (`hFiltNum`), y la herramienta "🧹 Duplicados" completa (`abrirLimpiezaDuplicados()` + 2 pasadas de detección).
- Dashboard: cross-filter interactivo (`dashFiltro`, clic en cualquiera de los 4 gráficos existentes filtra los demás + los KPIs; clic de nuevo quita el filtro) sobre los mismos 4 gráficos que ya tenía SW (Pipeline por Estado, Ventas por Asesor, Tasa de Cierre por Asesor, Top Tipologías) — no se agregaron gráficos nuevos (Origen, Top Clientes, Scatter, Ofertas x Mes, etc. de Máquinas siguen sin portar, quedan para una fase futura si se necesitan). De paso, 2 bugs reales corregidos en el Dashboard: **"Top Tipologías" duplicaba el valor de ítems con cantidad>1** (`item.tarifa` ya es el total de la línea, se volvía a multiplicar por `cantidad`; mismo bug ya corregido en Máquinas 12-sep-2026) y el mismo bug de asesores mezclados por primer nombre.

**⏳ Pendiente — resto de fase 1 (Dashboard):**
- Filtros de barra superior Mes/Asesor + toggle "Ocultar datos incompletos" (SW solo tiene el filtro de Año) — opcional, no pedido explícitamente, se puede agregar después si se necesita.

**✅ Ya portado (22-sep-2026) — fase 2 (Clientes):**
- Gerente de Cuenta completo — checkbox "Mis Clientes" en el toolbar (`renderTablaClientes()` filtra por `asesorLabel(c.asesor)===asesorLabel(currentUserName)`), header de la ficha con "⚠ Sin Gerente de Cuenta"/Asignarme (`asignarmeCliente()`) o Gerente actual/Reasignar (`mostrarReasignar()`+`confirmarReasignar()`, ambos vía `patchAsesorCliente()` compartido), y tarjeta "📊 Ofertas a este cliente" (KPIs Ofertas/Total Cotizado/Ganadas/Tasa de Cierre + lista, calculado desde `historialData` con carga perezosa si aún no se ha visitado Historial) — todo portado casi literal de Máquinas. **No se portó** el gráfico de dona `chartClienteEstado` de esa tarjeta (solo se dejaron los KPIs+lista), por alcance/tiempo — se puede agregar después si se pide. No se necesitó ninguna columna nueva en SharePoint: `Asesor` ya existe en la lista Clientes/Contactos (es compartida entre todos los cotizadores).
- De paso, 1 bug real grave corregido: `editarCliente()` (y el botón "+ Agregar contacto" del modal) llamaban a `mcAgregarContacto()`/`mcContactosCount`, que **nunca estuvieron definidas en este archivo** — SW nunca tuvo ese bloque completo, a diferencia de Máquinas. Esto significaba que **editar cualquier cliente existente rompía con un ReferenceError** (el modal ni siquiera abría). Se portó el bloque "MODAL CLIENTE: contactos dinámicos" completo de Máquinas.

**✅ Ya portado (22-sep-2026) — fase 3 (Posible Cierre + Seguimiento):**
- Selector "Posible cierre" (30/60/90/fecha personalizada) en la barra fija de Nueva Oferta, junto a Entrega — `calcularFechaCierreForm()`/`obtenerPosibleCierreLabel()`/`actualizarCierreLabel()`. Al guardar (`guardarCotizacion()`), si se definió, se manda `FechaCierre` (ISO) + `PosibleCierre` (etiqueta histórica de qué opción se usó) con el mismo patrón de reintento resiliente que ya usa `generarNumOferta()`: si Graph rechaza el POST porque una columna todavía no existe, reintenta sin ese campo (hasta 15 veces) en vez de perder la cotización completa, y avisa qué columnas faltan.
- `cargarHistorial()` ahora calcula `fechaCierre`/`fechaSeguimiento` (= FechaCierre − 5 días, vía `calcularFechaSeguimiento()`) /`posibleCierre` por cada oferta, usando `parsearFechaSharePoint()` para tolerar que Graph devuelva el campo "Fecha y hora" como ISO completo en vez de solo la fecha.
- Historial: nueva columna "Seguimiento" — clic en la fecha abre `abrirModalCierre()` (PATCH directo, sin crear revisión -R, pensado para llenar retroactivamente ofertas viejas).
- `editarOferta()` restaura el Posible Cierre existente como fecha personalizada (nunca intenta adivinar si originalmente fue 30/60/90, para no correr la fecha real al reabrir).
- Pantalla nueva "📅 Seguimiento" (botón de nav con badge rojo de vencidos+próximos 3 días) — vistas Día/Semana/Mes, sección "Vencidos" siempre visible arriba, filtra solo ofertas propias (`asesorLabel`) y activas (no Cerrada-Ganada/Perdida). Moneda EUR (no USD, a diferencia del texto que trae Máquinas de donde se portó).
- **Necesita 2 columnas nuevas en SharePoint** — el usuario debe crearlas en la lista `cotizaciones_sw`: `FechaCierre` (Fecha y hora) y `PosibleCierre` (Una línea de texto), mismos nombres que en Máquinas. Hasta que existan, el retry-loop de `guardarCotizacion()` guarda la oferta igual pero sin esos 2 campos (avisa por toast) y todo lo demás (Historial, Seguimiento) simplemente no tendrá fechas que mostrar.
- **No se portó** la agrupación "Solo Vigentes" (`soloVigentes()` de Máquinas, que oculta revisiones `-Rn` viejas). Se confirmó leyendo `editarOferta()` que SW **sí genera revisiones `-Rn` igual que Máquinas** (la nota anterior de este archivo, que decía que no, era especulación sin confirmar — quedó corregida). Sin "Solo Vigentes", si una oferta se revisa varias veces y las revisiones viejas quedan con Posible Cierre definido, la pantalla de Seguimiento podría mostrar tarjetas de revisiones ya superadas — no se implementó por no ser parte de lo pedido; si se vuelve molesto en la práctica, portar `soloVigentes()` de Máquinas resuelve esto.

**Fuera de alcance por ahora:** firma digital del cliente en el PDF, link de firma externa, agrupar revisiones -Rn ("Solo vigentes", ver nota arriba).

Ver `cotizador-ayj/CLAUDE.md` para el detalle de cómo funciona cada feature en Máquinas (la fuente de la que se está portando). **No portar todo de una sola vez** — se está avanzando por fases, cada una con su propio commit y prueba.

### Fixes recientes
- **22-sep-2026 — Historial: ordenar por columnas + bug real de moneda (mostraba Pesos etiquetados como USD).** El usuario reportó ambas cosas juntas:
  - **Ordenar por columnas**: se portó `ordenarHistorial(col)`/`_histOrden` de Máquinas — clic en N° Oferta/Fecha/Cliente/Vendedor/Total ordena (con flecha ↑/↓ en el header), segundo clic invierte.
  - **Bug de moneda (real, no solo de etiqueta)**: `GranTotal` en SharePoint se guarda en **COP**, no en EUR (`guardarCotizacion()` calcula `totalCOP=totalEUR*trm` y guarda ESE valor) — pero `cargarHistorial()` leía ese número directo y lo mostraba con el prefijo "USD" sin convertir. Es decir, el Historial mostraba el monto en pesos colombianos como si fuera dólares. El mismo dato mal etiquetado como "EUR" se había propagado hoy mismo al Dashboard, la pantalla de Seguimiento y la tarjeta "Ofertas a este cliente" (los 3 los porté yo en esta sesión, heredando el bug sin darme cuenta). Fix: se agregó el campo `TasaEur` (TRM COP/EUR usada al guardar) a `guardarCotizacion()`, y `cargarHistorial()`/`cargarDashboard()` ahora calculan `total` (EUR real = GranTotal COP ÷ TasaEur) y `totalCOP` (el valor guardado, sin convertir) por separado. El Historial ahora muestra **ambas monedas** en cada fila y en el resumen ("Total: EUR X · COP Y"); Dashboard/Seguimiento/Clientes quedan corregidos automáticamente porque todos leen el mismo `total` ya bien calculado. Las ofertas guardadas ANTES de este fix no tienen `TasaEur` guardado — se usa la TRM actual como aproximación (no es exacta para el histórico viejo, pero es lo mejor disponible sin esa columna). **Necesita 1 columna nueva en SharePoint**: `TasaEur` (Número) en la lista `cotizaciones_sw` — si no existe, el retry-loop de `guardarCotizacion()` guarda la oferta igual pero sin ese campo.
- **22-sep-2026 — 3 bugs reportados por el usuario: consecutivo pegado, duplicar y probabilidad "no funcionan".** Diagnóstico:
  - `editarProbabilidad()` y el chequeo de "es mi oferta" en el Historial usaban el mismo patrón fragil que causó el bug de Jorge Pulido/Jorge Salamanca en Máquinas (`currentUserName.includes(vendedor.split(' ')[0])`) — con 2 vendedores de nombre "Jorge" esto los mezcla, incluyendo en el permiso de editar. Se portó `asesorLabel()` de Máquinas y se usa en ambos lugares. Confirmado con prueba simulada: antes Jorge Pulido podía editar la probabilidad de la oferta de Jorge Salamanca; ahora se bloquea correctamente.
  - `duplicarOferta()` no tenía un mensaje claro cuando la oferta no tenía ítems guardados (`Items` vacío o mal parseado) — ahora avisa explícitamente en vez de fallar en silencio.
  - **El consecutivo pegado NO se pudo reproducir/confirmar desde este entorno** (no hay login real a SharePoint) — `generarNumOferta()` y `graphGetAll()` en sí mismos están bien (paginan correctamente, igual que en Máquinas). La sospecha más probable es que `resolverSite()` no está encontrando `listIdCotizaciones` (usa comparación EXACTA `n==='cotizaciones_sw'` contra el nombre de la lista en SharePoint — si el nombre real difiere aunque sea un espacio, esto nunca hace match y el consecutivo se queda pegado en 001 para siempre). Se agregó diagnóstico en consola (`console.log`/`console.warn`) en `resolverSite()` y `generarNumOferta()` — la próxima vez que pase, revisar la consola del navegador (F12) y confirmar si `listIdCotizaciones` sale "NO ENCONTRADA" y qué nombres de lista aparecen realmente.
- **31-ago-2026 — Consecutivo de cotización pegado.** `generarNumOferta()` usaba `graphGet()` sin paginar; se cambió a `graphGetAll()`. Ver regla en la sección de Graph API arriba.
