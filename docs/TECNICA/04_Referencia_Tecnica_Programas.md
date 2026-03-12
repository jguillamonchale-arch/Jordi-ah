# Referencia Técnica de Programas — Sistema AMS
## Fluidra S.A. — Fichas Técnicas de Programas Principales

**Versión:** 2.0 — Marzo 2026
**Audiencia:** Equipo AMS técnico, desarrolladores, mantenimiento
**Clasificación:** Interno — Técnico

---

## Índice

- [Guía de Lectura](#guía-de-lectura)
- [MÓDULO: Pedidos de Venta (CO/CM)](#módulo-pedidos-de-venta-cocm)
- [MÓDULO: Compras (PU)](#módulo-compras-pu)
- [MÓDULO: Motor DOM y Distribución (ED/DO)](#módulo-motor-dom-y-distribución-eddo)
- [MÓDULO: Maestros y Parametrización (MS/PM/MM)](#módulo-maestros-y-parametrización-mspmmm)
- [MÓDULO: EDI y Trazabilidad (TR/WR/EDTR)](#módulo-edi-y-trazabilidad-trwredtr)
- [MÓDULO: Configuración y Auxiliares](#módulo-configuración-y-auxiliares)
- [Tabla Resumen de Todos los Programas Principales](#tabla-resumen-de-todos-los-programas-principales)

---

## Guía de Lectura

Cada ficha técnica incluye:

| Sección | Contenido |
|---------|-----------|
| **Módulo** | Dominio de negocio al que pertenece |
| **Tipo** | SFL interactivo / Panel / Servicio / Proceso batch / DDS-PF / DDS-LF / DDS-Display |
| **Función** | Descripción breve del propósito del programa |
| **Fecha creación** | Fecha del primer código documentado |
| **Líneas aprox.** | Tamaño orientativo del programa |
| **Ficheros que lee** | Tabla de ficheros con modo de acceso |
| **Ficheros que escribe** | Ficheros actualizados/creados por el programa |
| **Llama a** | Subprogramas invocados mediante CALL |
| **Llamado desde** | Programas que invocan a este |
| **Parámetros** | Interfaz de entrada/salida (@ENTRY PLIST) |
| **Notas técnicas** | Patrones, peculiaridades, modificaciones relevantes |

**Modos de acceso a ficheros:**
- `IF` — Input (solo lectura)
- `UF` — Update (lectura + modificación)
- `O` — Output (solo escritura)
- `A` — Append (solo añadir registros)
- `K` — Acceso keyed (por clave)
- `USROPN` — El programa abre/cierra el fichero manualmente

---

## MÓDULO: Pedidos de Venta (CO/CM)

---

### CM0085 — Lista de pedidos de cliente

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta |
| **Tipo** | RPG Interactivo — Subfile (SFL) |
| **Pantalla** | CM0085FM |
| **Función** | Lista paginada de pedidos de cliente activos. Punto de entrada del módulo de compras. |
| **Fecha creación** | 23/07/1997 |
| **Líneas aprox.** | ~2.887 |

**Ficheros que lee:**

| Fichero | Modo | Descripción |
|---------|------|-------------|
| `CM0085FM` | CF WORKSTN | Pantalla con subfile SUFD |
| `COCLIN` | IF K | Maestro de clientes |
| `COCAPE` | IF K | Cabecera de pedido |
| `COCAPE23` | IF K | Lógico: acceso por cliente |
| `COCAPE25` | IF K | Lógico: pedidos status '10' *(FQ365)* |
| `COCAPE38` | IF K | Lógico: orden descendente *(09022)* |
| `CMA040` | IF K | Datos adicionales de pedido |
| `CMA04024` | IF K | Lógico de CMA040 |
| `CMA04034` | IF K | Lógico de CMA040 *(09022)* |
| `PMSETP` | UF A K | Setup del sistema |
| `PMTABD` | IF K | Tablas de parámetros |
| `CODEPE08` | IF K | Detalle pedido para importes *(FQ665)* |
| `MSIVAS` | IF K | IVA artículos *(FQ665)* |
| `PMUNAL` | IF K | Unidades de medida *(FQ665)* |
| `MSDIV0` | IF K | Divisas *(FQ665)* |
| `ZQGSRK00` | IF K | Riesgos de cliente *(2310B)* |
| `ZQGCSC00` | IF K | Gestor asignado al cliente *(FH002)* |

**Llama a:** `CO0120` (al seleccionar un pedido)

**Llamado desde:** `CM0085CL` (CL wrapper), acceso directo de usuario

**Notas técnicas:**
- ZXOPT1: `S01` (vista 1/2), `S02` (vista 3 — descendente), `END`
- El parámetro de LDA `PURLDADS` proporciona CIAS y DLGA del contexto
- Modificaciones clave: FQ365 (riesgo), FQ665 (importe), PG615/E7394 (bloqueo albarán Italia)

---

### CO0120 — Mantenimiento de cabecera de pedido

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta |
| **Tipo** | RPG Interactivo — Subfile + Panel |
| **Función** | Programa principal del módulo. Crea, modifica y confirma la cabecera del pedido. Orquestador del ciclo de vida del pedido. |
| **Fecha creación** | 23/07/1997 |
| **Líneas aprox.** | ~12.565 |

**Ficheros que lee y escribe:**

| Fichero | Modo | Descripción |
|---------|------|-------------|
| `PMTABL` | IF K | Tablas de numeración |
| `PMTABD` | UF K | Tablas de parámetros (actualizable) |
| `COCAPE` | UF K | Cabecera de pedido |
| `CODIPE` | UF K | Detalle de pedido |
| `COOBPE` | UF K | Observaciones |
| `COPECO` | UF K | Condiciones |
| `CODEPE` | UF K | Líneas del pedido |
| `COGAPE` | UF K | Agrupaciones |
| `COCAIM` | UF K | Cabecera de importes |
| `CODEP2` | UF K | Detalle vista 2 |
| `COCLIN` | IF K (PREFIX CL_) | Maestro de clientes |
| `MSLOAD` | IF K | Almacenes logísticos |
| `MSALMA08` | IF K USROPN | Almacenes Fluidra Direct *(F-2OL)* |
| `PMUNAL` | IF K | Unidades de medida *(28099)* |
| `PMSETP` | IF K | Parámetros de setup *(3003A)* |

**Llama a:**

| Programa | Condición | Función |
|----------|-----------|---------|
| `CO0140` | Al entrar en líneas | Edición de líneas del pedido |
| `CO2519` | Al confirmar/lanzar | Lanzamiento de orden |
| `CO2593` | Al imprimir | Parámetros de impresión |
| `CO2696` | Al expedir | Validación coenvío |
| `CO2755` | Por artículo | Categoría de rotación |
| `CO2611` | Al calcular plazo | Días laborables |
| `UT2725` | Al confirmar *(C1997)* | Email de confirmación |
| `MM2559` | Al imprimir | Verificación de impresora |
| `MMPROM` | En F4 | Búsqueda en diccionario |

**Llamado desde:** `CM0085` (al seleccionar pedido de la lista)

**Parámetros (estructura CO0120DS):**

| Parámetro | Dir | Descripción |
|-----------|-----|-------------|
| `@PCIAS` | E | Empresa |
| `@PDLGA` | E | Delegación |
| `@PNPCO` | E/S | Número de pedido |
| `@PTPED` | E/S | Tipo de pedido |
| `@PDATP` | E/S | Fecha del pedido |
| `@PNOPL` | E/S | Línea (al volver de CO0140) |
| `@PERRO` | S | Código de error/resultado |
| `ZXOPT` | E/S | Control de flujo |

**Notas técnicas:**
- El programa más grande del módulo CO (~12.500 líneas)
- La estructura de parámetros `CO0120DS` es una DS externa compartida con los programas invocados
- ZXOPT1: `S01` (lista), `Pnn` (paneles), `Wnn` (ventanas), `END`
- Modificaciones críticas: C0031 (coenvíos), FQ365 (riesgo), FQ284 (Fluidra Direct), F-2OL (multi-OL)

---

### CO0140 — Entrada de líneas de pedido

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta |
| **Tipo** | RPG Interactivo — Subfile |
| **Función** | Captura y modificación del detalle de líneas (artículos, cantidades, precios, fechas, ecotasa, coenvíos). |
| **Fecha creación** | 23/02/1998 |
| **Líneas aprox.** | ~21.369 (el más grande del sistema) |

**Parámetros (@ENTRY PLIST):**

| # | Parámetro | Dir | Descripción |
|---|-----------|-----|-------------|
| — | `@PERRO` | E | Si='1' → confirmar directamente sin mostrar pantalla |
| — | `@PCIAS` | E | Empresa |
| — | `@PDLGA` | E | Delegación |
| — | `@PNPCO` | E | Número de pedido |
| — | `@PTPED` | E | Tipo de pedido |
| — | `@PDATP` | E | Fecha del pedido |
| — | `@PERRO` | S | `' '`=OK, `'A'`=cancelado |

**Notas técnicas:**
- El programa más extenso del sistema: ~21.000 líneas
- Usa ZXOPT1: `Snn` (subfile), `Pnn` (panel), `Wnn` (ventana), `END`
- Modificación ECOP (2007): ecotasa automática
- Modificación C0074 (2006): compatibilidades ADR/Astralpool
- Modificaciones 31712/26712: manejo de kits y servicios en coenvíos

---

### CO2519 — Lanzamiento de órdenes de venta

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta |
| **Tipo** | RPG Proceso (interactivo o batch) |
| **Función** | Genera el número de orden de venta y lanza la orden al sistema de ejecución. |
| **Fecha creación** | 17/10/1997 |
| **Líneas aprox.** | ~1.921 |

**Ficheros que lee y escribe:**

| Fichero | Modo | Descripción |
|---------|------|-------------|
| `PMTABL` | IF K | Tablas de numeración (maestro) |
| `PMTABD` | UF K | Detalle tablas / numeración |
| `COCAPE` | UF K | Cabecera de pedido |
| `CODIPE` | UF K | Detalle de pedido |
| `COOBPE` | UF K | Observaciones |
| `COPECO` | UF K | Condiciones |
| `CODEPE` | UF K | Líneas del pedido |
| `COGAPE` | UF K | Agrupaciones |
| `COCAIM` | UF K | Cabecera importes |
| `CODEP2` | UF K | Detalle vista 2 |
| `COCLIN` | IF K (PREFIX CL_) | Maestro de clientes |
| `MSLOAD` | IF K | Almacenes logísticos |
| `MSALMA08` | IF K USROPN | Almacenes Fluidra Direct *(F-2OL)* |
| `PMUNAL` | IF K | Unidades de medida *(28099)* |
| `PMSETP` | IF K | Parámetros de setup *(3003A)* |

**Parámetros (@ENTRY PLIST — selección):**

| Parámetro | Dir | Descripción |
|-----------|-----|-------------|
| `@PECIAS` | E | Empresa |
| `@PEDLGA` | E | Delegación |
| `@PENPCO` | E | Número de pedido |
| `@PETPED` | E | Tipo de pedido |
| `@PEDATP` | E | Fecha del pedido |
| `@PETIPO` | E | Modo: `' '`=solo crear, `'1'`=solo lanzar, `'2'`=crear+lanzar |
| `@PERRO` | E/S | Modo/resultado |

**Códigos de resultado @PERRO:**

| Valor | Significado |
|-------|-------------|
| `' '` | OK — operación completada |
| `'1'` | Error — no se pudo localizar número de orden |

**Llama a:** `CO2755` (rotación), `UT2725` (email)

**Llamado desde:** `CO0120` (al lanzar pedido)

**Notas técnicas:**
- Numeración de OC: búsqueda en cascada CIAS+DLGA+TPED → CIAS+DLGA → CIAS+TPED → CIAS
- Modificación PGE21 (dic-2025): auditoría digital en CODEPE al lanzar
- Modificación F-2OL (sep-2021): soporte multi-OL

---

### CO2593 — Parámetros de impresión (PURLIBC8)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta |
| **Tipo** | RPG Panel (con soporte batch) |
| **Función** | Pantalla de configuración de parámetros de impresión. Soporta modo silencioso para llamadas desde SalesForce API. |
| **Líneas aprox.** | ~380 |

**Parámetros (@ENTRY PLIST):**

| # | Parámetro | Dir | Descripción |
|---|-----------|-----|-------------|
| 1 | `@PCIAS` | E | Empresa |
| 2 | `@PDLGA` | E | Delegación |
| 3 | `@PPROG` | E | Programa invocador |
| 4 | `@PCCIT` | E | Título de la pantalla |
| 5 | `@PGEXP` | E/S | Formato de exportación |
| 6 | `@PNPIC` | E/S | Número de copias |
| 7 | `@PPEPE` | E/S | Impresora |
| 8 | `@PPAR1` | E | `'1'`=no mostrar pantalla |
| 9 | `@PPAR2` | E | Libre |
| 10 | `@PJOBT` | E | Nombre del trabajo |
| 11 | `@PERRO` | S | Resultado: `' '`=OK, `'A'`=cancelado, `'3'`=error |

**Llama a:** `RTVTIJ` (detecta batch/interactivo), `MMPROM` (valida valores), `MM2559` (verifica impresora)

**Notas técnicas:**
- Si RTVTIJ retorna `'0'` (batch/QUSER) → no abre pantalla → usa parámetros recibidos
- Modificación B0251 (jun-2020, Lluís Albert Grau): soporte SalesForce API

---

### CO2611 — Suma días laborables a una fecha

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta — Utilitario |
| **Tipo** | RPG Subprograma de servicio (sin pantalla) |
| **Función** | Dado una fecha y N días laborables, calcula la fecha resultante excluyendo sábados y domingos. |
| **Fecha creación** | 06/10/2010 |
| **Autor** | KIKE |
| **Líneas aprox.** | ~200 |

**Parámetros (@ENTRY PLIST):**

| # | Parámetro | Tipo | Dir | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PEDAT1` | 8,0 | E | Fecha de inicio (YYYYMMDD) |
| 2 | `@PEDAYS` | 5,0 | E | Número de días laborables |
| 3 | `@PEDAT2` | 8,0 | S | Fecha resultado (YYYYMMDD) |
| 4 | `@PEERRO` | 1A | S | `' '`=OK, `'1'`=Error (fecha inválida) |

**Llama a:** `@GETWEEK` (obtiene día de la semana — 1=Lun, 0=Dom)

**Limitación conocida:** Solo excluye fines de semana. No contempla festivos.

---

### CO2696 — Validación de coenvíos

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta — Utilitario |
| **Tipo** | RPG Subprograma de servicio (sin pantalla) |
| **Función** | Verifica si un pedido puede expedirse según las reglas de integridad de coenvío (100% o nada). |
| **Fecha creación** | Feb-2006 |
| **Autor** | Carlos del Valle |
| **Líneas aprox.** | ~679 |

**Parámetros (@ENTRY PLIST):**

| # | Parámetro | Dir | Descripción |
|---|-----------|-----|-------------|
| 1 | `@PCIAS` | E | Empresa |
| 2 | `@PARTI` | E | Artículo |
| 3 | `@PVARI` | E | Variante |
| 4 | `@PMODE` | E | Modelo |
| 5 | `@PERRO` | S | Resultado (ver tabla) |

**Ficheros:** COCLIN (IF K), CODEPE (IF K)

**Códigos de retorno:** `' '`=OK, `'0'`=no es coenvío, `'1'`=no reservado, `'2'`=faltan líneas, `'3'`=reserva parcial, `'4'`=dependiente no OK

---

### CO2733 — Situación distribución conjuntos de venta

| Atributo | Valor |
|----------|-------|
| **Módulo** | Pedidos de Venta — Utilitario |
| **Tipo** | RPG Subprograma de servicio |
| **Función** | Determina si un artículo conjunto tiene todos sus componentes en Trace, ninguno, o mezcla. |
| **Fecha creación** | 03/01/2013 |
| **Autor** | Carlos del Valle |
| **Líneas aprox.** | ~173 |

**Parámetros:** CIAS, ARTI, VARI, MODE → `@PERRO`: `'1'`=todos Trace, `'2'`=ninguno, `'3'`=mixto, `'4'`=no es conjunto

**Ficheros:** MSARTI00 (IF K), MSCOJC (IF K), MSCOJD (IF K)

---

### CO2755 — Categoría de rotación de artículo

| Atributo | Valor |
|----------|-------|
| **Módulo** | Clasificación / MTS-MTO |
| **Tipo** | RPG Subprograma de servicio |
| **Función** | Determina la categoría de rotación MTS/MTO y subcategorías A/B/C de un artículo. |
| **Fecha creación** | 14/05/2010 |
| **Autor** | X.BOU |
| **Última modificación** | 26/11/2025 (08835 — corrección MTS/MTO Fluidra Direct, SCE-104) |
| **Líneas aprox.** | ~523 |

**Parámetros:**

| # | Parámetro | Dir | Descripción |
|---|-----------|-----|-------------|
| 1-7 | `@PCIAS, @PDLGA, @PLOGE, @PARTI, @PVARI, @PMODE, @PTCOJ` | E | Contexto del artículo |
| 8-11 | `@PCROT, @PCROTA, @PCROTB, @PCROTC` | S | Categorías resultado |
| 12 | `@PLOFD` | E | Flag Fluidra Direct (opcional — verificar `%PARMS=12`) |

**Ficheros:** PMLIBL, PMEMPR, MSLOAD, MSSTOK, MSSTST *(C1957)*, MSCOJD, MSARTI00, MSALMA05 *(CR964)*

---

### CO2610 / CO2770 — Pedidos pendientes de servir (KORE)

| Atributo | Valor |
|----------|-------|
| **Módulo** | KORE — Disponibilidad |
| **Tipo** | RPG Subprograma de servicio |
| **Función** | Para un artículo y rango de fechas, calcula la cantidad total pendiente de servir en pedidos activos. Invocado por motor KORE externo. |
| **Nombre objeto** | CO2610 (referenciado como CO2770) |
| **Fecha creación** | 07/03/2013 |
| **Líneas aprox.** | ~218 |

**Parámetros:**

| # | Parámetro | Dir | Descripción |
|---|-----------|-----|-------------|
| 1-6 | `@PCIAS, @PDLGA, @PARTI, @PVARI, @PMODE, @PTCOJ` | E | Artículo y contexto |
| 7-8 | `@PDATD, @PDATH` | E | Rango de fechas (YYYYMMDD) |
| 9 | `@PCANT` | E/S | Cantidad acumulada (entrada inicial + resultado) |
| 10 | `@PCAN1` | S | Cantidad resultado |
| 11 | `@PLOGE` | E | Almacén lógico |
| 12 | `@POMFD` | E | `'1'`=excluir pedidos modificados (S5TPE4='2') |
| 13 | `@PERRO` | S | Código error (reservado) |

**Ficheros:** CODEPE42 (IF K — por artículo), COCAPEA8 (IF K — pedidos activos), MSLOAD, MSARTI00, PMUNAL

**Subrutina interna:** `CS_CNVUBS` — convierte cantidades a unidad base del artículo usando PMUNAL

---

## MÓDULO: Compras (PU)

---

### PU2519 — Consulta de órdenes de compra

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Tipo** | RPG Interactivo — Subfile |
| **Función** | Lista de órdenes de compra activas con filtros y opciones de gestión. |

**Ficheros:** PUCAPE (IF K), PUCAPE21 (IF K), PUPROV (IF K), PMTABD (IF K)

**Llama a:** PU2600 (al seleccionar una OC)

---

### PU2600 — Gestión de cabecera de OC

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Tipo** | RPG Interactivo — SFL + Panel |
| **Función** | Creación, modificación y gestión completa de la cabecera de una Orden de Compra. |

**Ficheros principales:**

| Fichero | Modo | Descripción |
|---------|------|-------------|
| `PUCAPE` | UF K | Cabecera de OC |
| `PUDEPE` | UF K | Líneas de OC |
| `PUPROV` | IF K | Maestro de proveedores |
| `PUOBPE` | UF K | Observaciones de OC |
| `PUPECO` | UF K | Condiciones de OC |
| `PUCAIM` | UF K | Cabecera de importes OC |
| `PUCEDI` | IF K | Config EDI proveedor |
| `PUPRMD` | IF K | Parámetros de modificación |

---

### PUPRPR — Proceso de Orden de Compra

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Tipo** | RPG Proceso |
| **Función** | Ejecuta el proceso de OC: cálculos de importe, validaciones, actualización de estados. |

**Ficheros:** PUCAPE, PUDEPE, PUPROV, PUCAIM, PUPRMD, MSSTOK, MSARTI00

---

### PUCEDI — Configuración EDI de Proveedor

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras / EDI |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Tabla maestra de configuración EDI por proveedor. Define qué mensajes EDIFACT se habilitan para cada proveedor. |

**Campos clave:**

| Campo | Descripción |
|-------|-------------|
| `C9CIAS + C9PRVR` | Clave: empresa + proveedor |
| `C9EPED` | `'1'`=enviar ORDERS por EDI |
| `C9RRSP` | `'1'`=recibir ORDRSP |
| `C9RINV` | `'1'`=recibir INVRPT |
| `C9RFAC` | `'1'`=enviar INVOIC |
| `C9RALB` | `'1'`=enviar DESADV |
| `C9COMP/C9RECE/C9CFAC/C9PAGA` | Roles EDIFACT |

---

### PUHALD — Histórico de albaranes de compra

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Tipo** | DDS Fichero Físico + proceso RPG |
| **Función** | Registro histórico de albaranes de compra recibidos. |

**Fichero lógico:** `PUHALD17` (acceso por fecha)

---

### PUPROV — Maestro de proveedores

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras / Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Datos maestros de cada proveedor: código, nombre, condiciones, país, divisa, estado. |

---

## MÓDULO: Motor DOM y Distribución (ED/DO)

---

### ED3025CL — CL Wrapper DOM

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM |
| **Tipo** | CL Program (Command Language) |
| **Función** | Punto de entrada batch del Motor DOM. Copia ficheros a QTEMP para aislamiento. Llama a ED3025. |
| **Líneas aprox.** | ~30 |

**Secuencia:**
1. `CPYF COCAPE → QTEMP/COCAPE`
2. `CPYF CODEPE → QTEMP/CODEPE`
3. `CPYF CODEPE20/26 → QTEMP/...`
4. `CPYF COHFEN → QTEMP/COHFEN`
5. `CALL ED3025`

**Llama a:** `ED3025`

---

### ED3025 — Orquestador del Motor DOM

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM |
| **Tipo** | RPG Orquestador |
| **Función** | Gestiona el flujo completo del proceso DOM: decide cuándo activar DOOM, presenta confirmación, llama EDI y auditoría. |
| **Líneas aprox.** | ~237 |

**Parámetros:**

| Parámetro | Dir | Descripción |
|-----------|-----|-------------|
| `@PCIAS` | E | Empresa |
| `@PDLGA` | E | Delegación |
| `@PNPCO` | E | Número de pedido |
| `@PTPED` | E | Tipo de pedido |
| `@PDATP` | E | Fecha del pedido |
| `@PHORA` | E | Hora del proceso |
| `@PERRO` | S | Resultado |

**Ficheros:** Trabaja sobre copias en QTEMP (COCAPE, CODEPE, CODEPE20, CODEPE26, ED0050W, COHFEN)

**Llama a:** `ED3050` (DOOM), `ED3061` (confirmación), `ED0060` (EDI), `ED0100` (auditoría)

**Condiciones de bypass:**
- `S0MOST='1'` → no llama ED3050
- `S0SPLF flag` → no llama ED3050
- `ZXERRO='R'` → no llama ED3061

---

### ED3050 / DOOM01 — Algoritmo DOOM

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM |
| **Tipo** | RPG Proceso Batch |
| **Función** | Core del Motor DOM. Implementa el cubo de decisión 3D: para cada artículo × almacén × validación, selecciona el almacén óptimo. |
| **Líneas aprox.** | ~3.390 (ED3050) / ~5.187 (DOOM01) |

**Ficheros que lee:**

| Fichero | Descripción |
|---------|-------------|
| `COCAPE` (QTEMP) | Cabecera del pedido |
| `CODEPE` (QTEMP) | Líneas del pedido |
| `MSALMA` | Almacenes físicos |
| `MSLOAD` | Almacenes lógicos |
| `MSARTI00` | Maestro de artículos |
| `MSSTOK` | Stock por almacén |
| `EDPAR2` | Parámetros por almacén |
| `EDPAR3` | Restricciones capacidad |
| `EDPAR4` | Prioridades de distribución |
| `EDPARC` | Parámetros por cliente |
| `EDPARM` | Parámetros por delegación |
| `MSDDOM` / `MSDDOM11-14` | Matriz delegaciones + prioridades especiales |
| `PUPROV` | Maestro de proveedores |
| `MSTRTA` | Tipos de transporte |
| `MSCOJC/MSCOJD` | Kits y componentes |

**Ficheros que escribe:**

| Fichero | Campo | Descripción |
|---------|-------|-------------|
| `CODEPE` (QTEMP) | `S5LOGE` | Almacén lógico asignado |
| `CODEPE` (QTEMP) | `S5PROV` | Proveedor asignado |
| `DOOMHT` | todos | Histórico de la decisión |
| `DO0050W-053W` | todos | Ficheros de trabajo DOM |

**Notas técnicas:**
- El algoritmo es el programa más complejo del sistema (5.187 líneas en DOOM01)
- Cubo de decisión: loop artículos → loop almacenes → evaluación de tiers
- Los ficheros de trabajo `DO0050W-053W` residen en QTEMP para evitar conflictos
- Ver Documento 3 para descripción completa del algoritmo

---

### ED0060 — Generación de mensajes EDI

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM / EDI |
| **Tipo** | RPG Proceso Batch |
| **Función** | Genera los mensajes EDIFACT (ORDERS, DESADV, INVOIC, etc.) tras la confirmación del DOM. |

**Ficheros que lee:** EDPARM, EDPARC, EDPAR4, PUCEDI, COCEDI, COCAPE, CODEPE

**Mensajes generados:**
- `ORDERS` → proveedor (si PUCEDI.C9EPED='1')
- `INVOIC` → cliente (si COCEDI.S0EFAC='1')
- `DESADV` → cliente (si COCEDI.S0EALB='1')
- `ORDRSP` → registra respuesta recibida
- `INVRPT` → si PUCEDI.C9RINV='1'

---

### DO0001 — Lista de prioridades DOM (v1)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM — Configuración |
| **Tipo** | RPG Interactivo — Subfile |
| **Función** | Lista paginada de reglas de prioridad DOM nivel empresa/almacén con vista de delegación. |
| **Fecha creación** | 16/01/2014 |
| **Líneas aprox.** | ~1.405 |

**Ficheros:** EDPARM, EDPAR2, EDPAR4 (UF K), PMLIBL, PMEMPR, MSALMA, MSSGGA, PMSETP (UF A K), MSARTI00, PUPROV, MSFAMI, PMTABD, COCLIN

**Llama a:** `DO0005` / `DO0025` (al seleccionar una regla)

**Notas técnicas:** Soporta TREG='H' (excepción por procedimiento de pedido) que DO0020 no soporta. Modificación IT244 (may-2022): 2ª fase multi-OL.

---

### DO0005 — Panel de prioridad DOM (v1)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM — Configuración |
| **Tipo** | RPG Interactivo — Panel (formulario) |
| **Función** | Alta, baja, copia y modificación de reglas de prioridad DOM a nivel delegación. |
| **Fecha creación** | 18/03/2010 |
| **Líneas aprox.** | ~1.392 |

**Operaciones:** DSP / CRT / DLT / CPY / CHG

**Ficheros:** EDPAR2 (IF K), EDPAR4 (UF A K), EDHPA4 (O K — histórico), PMEMPR, PMTAB2, MSARTI00, PMTABD, MSALMA, PUPROV, MSFAMI, COCLIN

**Notas:** Al modificar EDPAR4 → escribe en EDHPA4 (auditoría de cambios). Modificación C3758 (nov-2017): excepciones por artículo.

---

## MÓDULO: Maestros y Parametrización (MS/PM/MM)

---

### MSARTI00 — Maestro de artículos

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Tabla central del catálogo de productos. Contiene toda la información maestra de cada artículo. |

**Clave:** `M5CIAS + M5ARTI + M5VARI + M5MODE`

**Campos principales:**

| Campo | Descripción |
|-------|-------------|
| `M5UNIM` | Unidad de medida base |
| `M5SIOL` | Indicador Trace (`'2'`=en Trace) |
| `M5TCOJ` | Tipo conjunto (`'4'`=venta, `'1'`=compra) |
| `M5FAMI` | Familia |
| `M5CROP` | Categoría rotación propuesta |
| `M5ECOP` | Flag ecotasa |
| `M5OMP` | Order Management Parameter |
| `M5OBSO` | Flag obsolescencia |
| `M5AMPL` | Área de datos ampliados (500 bytes) |

**Lógicos:** MSARTI03 (por familia), MSARTI07 (por almacén), MSARTI10 (por código alternativo)

---

### MSSTOK — Stock por almacén

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Stock disponible de cada artículo en cada almacén físico. Consulta principal del DOOM para disponibilidad. |

**Clave:** `M1CIAS + M1MAGA + M1ARTI + M1VARI + M1MODE`

**Campos clave:** `M1DISP` (disponible), `M1RESD` (reservado), `M1PEND` (pendiente recibir)

**Lógico:** MSSTOK01

---

### MSSTST — Stock estadístico

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Datos históricos y estadísticos de stock. Usado por CO2755 para clasificación MTS/MTO. |

**Clave:** `M6CIAS + M6DLGA + M6ARTI + M6VARI + M6MODE + M6NETL`

---

### MSALMA — Almacenes físicos

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) + programa RPG |
| **Función** | Maestro de almacenes físicos (edificios, ubicaciones reales). |

**Lógicos:** MSALMA00, MSALMA02, MSALMA03, MSALMA05 (para CO2755), MSALMA08 (Fluidra Direct)

---

### MSLOAD — Almacenes logísticos

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Vista de negocio de almacenes: empresa + delegación + almacén lógico (LOGE). Múltiples vistas lógicas de un mismo almacén físico. |

**Clave:** `MICIAS + MIDLGA + MILGCA`

**Campo clave:** `MIMAGA` (almacén físico asociado), `MINDS` (nivel de servicio)

**Lógico:** MSLOAD02

---

### PMEMPR — Parámetros de empresa

| Atributo | Valor |
|----------|-------|
| **Módulo** | Parametrización |
| **Tipo** | DDS Fichero Físico (PF) + programa RPG |
| **Función** | Hub central de configuración del sistema. Cada empresa tiene su propio registro con decenas de parámetros que controlan el comportamiento de todos los módulos. |

**Campo clave:** `PPAMPL` — área extendida de 500 bytes con flags de configuración

**Subcampos críticos de PPAMPL:**

| Posición | Campo | Uso |
|----------|-------|-----|
| 1 | `PPCROT` | Activar categorías de rotación |
| 2 | `PPRISK` | Nivel de riesgo |
| 6 | `PPCROP` | Categoría rotación propuesta |
| 7-26 | `PPBBAC/PPBBAP` | Umbrales categoría A |
| 29-34 | `PPDWVT`-`PPDWFB` | Flags de control |

**Lógico:** PMEMPR00

---

### PMTABL / PMTABD — Tablas de parámetros

| Atributo | Valor |
|----------|-------|
| **Módulo** | Parametrización |
| **Tipo** | DDS Fichero Físico (PF) x2 |
| **Función** | Sistema de tablas de dominio configurables. PMTABL=cabecera, PMTABD=valores. |

**Uso principal:** tabla `PECLTA` para numeración de órdenes de venta, dominio de tipos de pedido, estados, etc.

---

### MSDDOM — Matriz de delegaciones

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros / DOM |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Define las relaciones permitidas entre delegaciones origen y destino, incluyendo calendarios de envío, transportistas, límites y prioridades especiales. |

**Clave:** `XACIAS + XADLGA + XADLG2 + XATIPD`

**Lógicos especializados:**
- `MSDDOM11` — por prioridad OB (Overstock)
- `MSDDOM12` — por prioridad OC (Obsolete)
- `MSDDOM13` — por prioridad SSTO (Super-Stock)
- `MSDDOM14` — por prioridad ArtB

---

## MÓDULO: EDI y Trazabilidad (TR/WR/EDTR)

---

### EDPAR4 — Prioridades de distribución DOM

| Atributo | Valor |
|----------|-------|
| **Módulo** | Motor DOM / EDI |
| **Tipo** | DDS Fichero Físico (PF) — Maestro principal DOM |
| **Función** | Tabla maestra de reglas de prioridad de distribución. Cada registro define qué almacén se prioriza para una combinación empresa/almacén/tipo de regla/criterio. |

**Clave:** `T6CIA1 + T6MAGA + T6TREG + [criterio variable según TREG]`

**Ver:** Documento 3 sección 4.1 para descripción completa

**Lógico:** EDPAR401

**Mantenido por:** DO0005, DO0025

**Auditado en:** EDHPA4 (histórico de cambios)

---

### EDTRLP — Trazabilidad de línea de pedido

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Trazabilidad |
| **Tipo** | DDS Fichero Físico (PF) — ~98 campos |
| **Función** | Registro de trazabilidad completa de cada línea de pedido en el proceso DOM/EDI. |

**Clave:** `Empresa + Delegación + OC + Tipo + Fecha + Hora + Línea + Artículo`

**Lógicos:** EDTRLP00, EDTRLP10, EDTRLP20, EDTRLP30, EDTRLP40

**Campos de estado clave:** T0EDSN (EDI enviado), T0RISK (riesgo), T0PHLD (retenida), T0MODI (modificada), T0CIA1-3 (referencias cruzadas OC múltiples)

**Histórico:** EDTRLX (mismo estructura, versión archivo/histórico)

---

### EDTRAU — Auditoría de eventos EDI

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Auditoría |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Log de todos los eventos significativos del proceso EDI. Un registro por evento. |

**Campos clave:** T1EDAU (código evento), T1EDAD (descripción), T1DATO (600 bytes de datos), T1USER, T1JOBR, T1PGMR, T1STUL (estado final)

**Lógico:** EDTRAU00

**Versión extendida:** EDTRAX / EDTRAX00

---

### TR0026 — Estructura línea pedido de venta (transaccional)

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Transacciones |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Definición de la estructura de datos transaccional de una línea de pedido de venta. Sirve como modelo de datos para el intercambio entre módulos. |

**Clave:** `S5DLGA + S5CIAS + S5DATP + S5NOPL + S5NPCO + S5TPED + S5LINE`

**Campos de cantidad:** S5CANT (pedida), S5CANS (servida), S5CANC (cancelada)

**Fichero de trabajo:** WR0026 (copia en QTEMP)

**Lógico:** TR002602

---

### TR0035 — Estructura cabecera OC (transaccional)

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Transacciones |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Definición de la estructura de datos de cabecera de Orden de Compra para el proceso transaccional. |

**Clave:** `C0DATP + C0CIAS + C0NPCO + C0PRVR`

**Campos clave:** C0FPED (fecha pedido), C0FRRE (fecha recepción real), C0FREP (esperada), C0TTTA (transportista), C0DTPP (descuento)

---

### TR0036 — Estructura línea OC (transaccional)

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Transacciones |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Definición de la estructura de datos transaccional de una línea de Orden de Compra. Paralelo a TR0026 para el lado de compras. |

**Fichero de trabajo:** WR0036 (copia en QTEMP)

**Lógico:** TR003602

---

### TRCPDE — Trazabilidad de entorno de OC

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Trazabilidad |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Registro de trazabilidad del proceso de la OC, complementario a TR0026. Incluye campos de entrada manual, tracking de destino y programa origen. |

**Campos adicionales respecto a TR0026:** RMAN (manual), DEST/TDES/CIDT/CLDT (tracking destino), PROG (programa origen)

**Lógicos:** TRCPDE01, TRCPDE02

---

## MÓDULO: Configuración y Auxiliares

---

### PMLIBL — Parámetros de biblioteca

| Atributo | Valor |
|----------|-------|
| **Módulo** | Parametrización |
| **Tipo** | DDS Fichero Físico (PF) + proceso RPG |
| **Función** | Gestiona la lista de librerías IBM i por empresa/delegación. Punto de inicio de la sesión: determina qué librerías de datos y programas usar. |

---

### PMEMPR00 — Parámetros de empresa (lógico)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Parametrización |
| **Tipo** | DDS Fichero Lógico (LF) sobre PMEMPR |
| **Función** | Acceso lógico al fichero de parámetros de empresa. |

---

### COCEDI — Configuración EDI de cliente

| Atributo | Valor |
|----------|-------|
| **Módulo** | EDI / Clientes |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Configuración EDI por cliente: qué mensajes EDIFACT se envían/reciben para cada cliente. Complementario a PUCEDI (que es para proveedores). |

**Clave:** `S0CIAS + S0CUST`

**Campos principales:** S0EFAC (factura), S0EALB (albarán), S0RPED (recibir OC), S0COMP/RECE/CFAC/PAGA (roles EDIFACT), S0CAUT (auto-close Edicom)

**Lógico:** COCEDI02

---

### PMUNAL — Unidades de medida

| Atributo | Valor |
|----------|-------|
| **Módulo** | Maestros |
| **Tipo** | DDS Fichero Físico (PF) |
| **Función** | Tabla de factores de conversión entre unidades de medida por artículo. Usada en CO2610 y en todos los procesos que necesitan convertir cantidades. |

**Clave:** `P6CIAS + P6ARTI + P6VARI + P6MODE + P6UNIM`

**Campos:** P6COFA (factor de conversión), P6TCOF (`'1'`=divisor, otro=multiplicador)

---

### UT2725 — Envío de email de confirmación

| Atributo | Valor |
|----------|-------|
| **Módulo** | Utilidades |
| **Tipo** | RPG Subprograma de servicio |
| **Función** | Envía un email de confirmación de pedido al cliente. Invocado por CO0120 y CO2519 tras confirmar/lanzar un pedido. |

**Llamado desde:** CO0120 *(mod. C1997, sep-2014)*, CO2519

---

### APIENWF — API de entrada a workflow

| Atributo | Valor |
|----------|-------|
| **Módulo** | Integraciones |
| **Tipo** | RPG Servicio API |
| **Función** | Punto de entrada para integraciones externas que disparan workflows en el sistema AMS. |

---

## Tabla Resumen de Todos los Programas Principales

| Programa | Módulo | Tipo | Pantalla | Líneas | Fecha |
|----------|--------|------|----------|--------|-------|
| `CM0085` | Ventas | SFL interactivo | CM0085FM | ~2.887 | Jul-1997 |
| `CO0120` | Ventas | SFL+Panel | varios | ~12.565 | Jul-1997 |
| `CO0140` | Ventas | SFL interactivo | varios | ~21.369 | Feb-1998 |
| `CO2519` | Ventas | Proceso | — | ~1.921 | Oct-1997 |
| `CO2593` | Ventas | Panel batch-safe | CO2593FM | ~380 | ~2004 |
| `CO2611` | Ventas/Util | Servicio | — | ~200 | Oct-2010 |
| `CO2696` | Ventas | Servicio | — | ~679 | Feb-2006 |
| `CO2733` | Ventas/Util | Servicio | — | ~173 | Ene-2013 |
| `CO2755` | Clasificación | Servicio | — | ~523 | May-2010 |
| `CO2610` | KORE | Servicio | — | ~218 | Mar-2013 |
| `PU2519` | Compras | SFL interactivo | — | — | — |
| `PU2600` | Compras | SFL+Panel | — | — | — |
| `PUPRPR` | Compras | Proceso | — | — | — |
| `ED3025CL` | DOM | CL Wrapper | — | ~30 | — |
| `ED3025` | DOM | Orquestador | — | ~237 | — |
| `ED3050` | DOM | Proceso batch | — | ~3.390 | — |
| `DOOM01` | DOM | Proceso batch | — | ~5.187 | — |
| `ED3061` | DOM | Panel | — | — | — |
| `ED0060` | DOM/EDI | Proceso batch | — | — | — |
| `ED0100` | DOM | Proceso batch | — | — | — |
| `ED0015` | DOM | Proceso batch | — | ~100 | — |
| `DO0001` | DOM/Config | SFL interactivo | DO0001FM | ~1.405 | Ene-2014 |
| `DO0005` | DOM/Config | Panel | DO0005FM | ~1.392 | Mar-2010 |
| `DO0010` | DOM/Config | SFL+Panel | varios | ~1.329 | Abr-2005 |
| `DO0020` | DOM/Config | SFL interactivo | DO0020FM | ~1.591 | Ene-2014 |
| `DO0025` | DOM/Config | Panel | DO0025FM | ~1.482 | Mar-2010 |
| `COCEDI` | EDI | DDS PF | — | — | — |
| `PUCEDI` | EDI | DDS PF | — | — | — |
| `EDPAR4` | DOM | DDS PF | — | — | — |
| `EDPARM` | DOM | DDS PF | — | — | — |
| `EDPARC` | DOM | DDS PF | — | — | — |
| `EDPAR2` | DOM | DDS PF | — | — | — |
| `EDPAR3` | DOM | DDS PF | — | — | — |
| `EDTRLP` | Trazab. | DDS PF | — | — | — |
| `EDTRAU` | Auditoría | DDS PF | — | — | — |
| `DOOMHT` | DOM | DDS PF | — | — | — |
| `MSDDOM` | Maestros | DDS PF | — | — | — |
| `MSARTI00` | Maestros | DDS PF | — | — | — |
| `MSSTOK` | Maestros | DDS PF | — | — | — |
| `MSALMA` | Maestros | DDS PF | — | — | — |
| `MSLOAD` | Maestros | DDS PF | — | — | — |
| `PMEMPR` | Paráms. | DDS PF | — | — | — |
| `PMTABL/D` | Paráms. | DDS PF | — | — | — |
| `PMUNAL` | Maestros | DDS PF | — | — | — |
| `TR0026` | EDI/Trans | DDS PF | — | — | — |
| `TR0035/36` | EDI/Trans | DDS PF | — | — | — |
| `TRCPDE` | EDI/Trans | DDS PF | — | — | — |
| `UT2725` | Utilidades | Servicio | — | — | Ago-2014 |
| `APIENWF` | Integración | API | — | — | — |

---

*Sistema AMS — Fluidra S.A. | Referencia Técnica de Programas | Versión 2.0 — Marzo 2026*
