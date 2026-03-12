# Referencia Técnica de Programas
## Sistema de Gestión de Pedidos y Logística — Fluidra S.A.

---

## Índice

- [CM0085 — Lista de pedidos de cliente](#cm0085--lista-de-pedidos-de-cliente)
- [CO0120 — Mantenimiento de cabecera de pedido](#co0120--mantenimiento-de-cabecera-de-pedido)
- [CO0140 — Entrada de líneas de pedido](#co0140--entrada-de-líneas-de-pedido)
- [CO2519 — Lanzamiento de órdenes de venta](#co2519--lanzamiento-de-órdenes-de-venta)
- [CO2593 — Parámetros de impresión (PURLIBC8)](#co2593--parámetros-de-impresión-purlibc8)
- [CO2611 — Sumar días laborables a una fecha](#co2611--sumar-días-laborables-a-una-fecha)
- [CO2696 — Validación de coenvíos](#co2696--validación-de-coenvíos)
- [CO2733 — Situación distribución conjuntos de venta](#co2733--situación-distribución-conjuntos-de-venta)
- [CO2755 — Categoría de rotación de artículo](#co2755--categoría-de-rotación-de-artículo)
- [CO2610 / CO2770 — Pedidos pendientes de servir (KORE)](#co2610--co2770--pedidos-pendientes-de-servir-kore)
- [DO0001 — Lista prioridades DOM nivel delegación (v1)](#do0001--lista-prioridades-dom-nivel-delegación-v1)
- [DO0005 — Panel prioridades DOM nivel DLGA (v1)](#do0005--panel-prioridades-dom-nivel-dlga-v1)
- [DO0010 — Parámetros Fluido Direct](#do0010--parámetros-fluido-direct)
- [DO0020 — Lista prioridades DOM nivel delegación (v2)](#do0020--lista-prioridades-dom-nivel-delegación-v2)
- [DO0025 — Panel prioridades DOM nivel DLGA (v2)](#do0025--panel-prioridades-dom-nivel-dlga-v2)

---

## CM0085 — Lista de pedidos de cliente

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Nombre objeto** | CM0085 |
| **Fichero fuente** | CM0085.MBR |
| **Copyright** | Aquapoint S.A. → Fluidra S.A. |
| **Fecha creación** | 23/07/1997 |
| **Tipo** | Programa interactivo — Subfile |
| **Pantalla** | CM0085FM |

### Función
Lista paginada de pedidos de cliente activos. Permite filtrar, navegar y seleccionar un pedido para lanzar su mantenimiento en CO0120.

### Parámetros de entrada (PLIST)

Los parámetros se reciben a través de la estructura de datos externa `CM0085DS`:

| Nombre | Descripción |
|--------|-------------|
| `ZXOPT` | Control de flujo inicial (valor inicial: `'S01CLR'`) |
| Campos de contexto de empresa/delegación | Via LDA (PURLDADS) |

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| CM0085FM | CF/WORKSTN | Pantalla principal con subfile SUFD |
| COCLIN | IF/K DISK | Maestro de clientes |
| COCAPE | IF/K DISK | Cabecera de pedido |
| COCAPE23 | IF/K DISK | Lógico — acceso por cliente |
| COCAPE25 | IF/K DISK | Lógico — incluye pedidos status 10 *(FQ365)* |
| COCAPE38 | IF/K DISK | Lógico — ordenación descendente *(09022)* |
| CMA040 | IF/K DISK | Datos adicionales de pedido |
| CMA04024 | IF/K DISK | Lógico de CMA040 |
| CMA04034 | IF/K DISK | Lógico de CMA040 *(09022)* |
| PMSETP | UF A/K DISK | Setup del sistema |
| PMTABD | IF/K DISK | Tablas de parámetros |
| COCLIN09 | IF/K DISK (PREFIX C_) | Lógico clientes para importe *(FQ665)* |
| CODEPE08 | IF/K DISK | Detalle pedido para importes *(FQ665)* |
| MSIVAS | IF/K DISK | IVA artículos *(FQ665)* |
| PMUNAL | IF/K DISK | Unidades de medida *(FQ665)* |
| MSDIV0 | IF/K DISK | Divisas *(FQ665)* |
| ZQGSRK00 | IF/K DISK | Riesgos de cliente *(2310B)* |
| ZQGCSC00 | IF/K DISK | Gestor asignado a cliente *(FH002)* |

### Control de flujo ZXOPT

| ZXOPT1 | Proceso |
|--------|---------|
| `S01` | Subfile — lista de pedidos (vistas 1 y 2) |
| `S02` *(09022)* | Subfile — lista ordenada por pedido descendente |
| `END` | Fin del programa |

| ZXOPT2 | Rutina |
|--------|--------|
| `CLR` | Limpiar subfile |
| `DSP` | Visualizar lista |
| `CHK` | Validar teclas / opciones de línea |
| `UPD` | Procesar acción seleccionada |

### Modificaciones relevantes

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| FQ365 | 23/03/2009 | Incluir pedidos status '10' y confirmar si autorizado riesgo |
| FQ446 | 14/07/2009 | Incluir CC y PR en la lista |
| FQ665 | 23/06/2010 | Añadir importe a las dos vistas |
| 09022 | 09/02/2012 | Nueva vista por pedido descendente |
| FH002 | 25/04/2012 | Permitir selección por gestor de riesgos |
| 2310B | 23/10/2012 | Cambio forma de acceso a gestor asignado |
| PG615 | 22/02/2023 | Retener pedidos por bloqueo albarán |
| E7394 | 15/03/2023 | Revertir PG615 para Italia |

---

## CO0120 — Mantenimiento de cabecera de pedido

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Nombre objeto** | CO0120 |
| **Fichero fuente** | CO0120.MBR |
| **Copyright** | Aquapoint S.A. → Fluidra S.A. |
| **Fecha creación** | 23/07/1997 |
| **Tipo** | Programa interactivo — Subfile + Panel |
| **Líneas de código** | ~12.565 |

### Función
Programa principal del módulo de compras. Gestiona la cabecera del pedido de cliente: creación, modificación, confirmación y lanzamiento. Es el orquestador del ciclo de vida del pedido.

### Parámetros (estructura externa CO0120DS)

Los parámetros se pasan mediante la estructura de datos externa `CO0120DS`. Los campos clave incluyen:

| Campo | Dir. | Descripción |
|-------|------|-------------|
| `@PCIAS` | E | Empresa |
| `@PDLGA` | E | Delegación |
| `@PNPCO` | E/S | Número de pedido |
| `@PTPED` | E/S | Tipo de pedido |
| `@PDATP` | E/S | Fecha del pedido |
| `@PNOPL` | E/S | Número de línea (cuando vuelve de CO0140) |
| `@PERRO` | S | Código de error/resultado |
| `ZXOPT` | E/S | Control de flujo inicial |

*(La estructura completa se define en CO0120DS — fichero de descripciones de datos externas)*

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| PMTABL | IF/K DISK | Tablas de numeración |
| PMTABD | UF/K DISK | Tablas de parámetros (actualizable) |
| COCAPE | UF/K DISK | Cabecera de pedido |
| CODIPE | UF/K DISK | Detalle de pedido |
| COOBPE | UF/K DISK | Observaciones de pedido |
| COPECO | UF/K DISK | Condiciones del pedido |
| CODEPE | UF/K DISK | Líneas del pedido |
| COGAPE | UF/K DISK | Agrupaciones del pedido |
| COCAIM | UF/K DISK | Cabecera de importes |
| CODEP2 | UF/K DISK | Detalle pedido vista 2 |
| COCLIN | IF/K DISK (PREFIX CL_) | Maestro de clientes |
| MSLOAD | IF/K DISK | Almacenes logísticos |
| MSALMA08 | IF/K DISK (USROPN) | Almacenes — Fluido Direct *(F-2OL)* |
| PMUNAL | IF/K DISK | Unidades de medida *(28099)* |
| PMSETP | IF/K DISK | Parámetros de setup *(3003A)* |

### Llamadas a subprogramas

| CALL | Condición | Descripción |
|------|-----------|-------------|
| `CO0140` | Al entrar en detalle de líneas | Edición de líneas del pedido |
| `CO2519` | Al confirmar/lanzar | Generación número de orden y lanzamiento |
| `CO2593` | Al imprimir | Parámetros de impresión |
| `CO2696` | Al expedir | Validación de integridad de coenvío |
| `CO2755` | Por línea de artículo | Obtener categoría de rotación |
| `CO2611` | Al calcular plazo | Suma días laborables a fecha base |
| `UT2725` | Al confirmar *(C1997)* | Envío de mail de confirmación |
| `MM2559` | Al imprimir | Verificación de impresora |
| `MMPROM` | En F4 de campos | Búsqueda en diccionario de valores |

### Control de flujo ZXOPT

| ZXOPT1 | Proceso |
|--------|---------|
| `S01` | Subfile principal — lista/cabecera |
| `Pnn` | Formatos de panel (formularios de detalle) |
| `Wnn` | Ventanas popup |
| `END` | Fin del programa |

### Modificaciones relevantes (selección)

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| 07074 | 07/07/2004 | Aviso campo referencia duplicado |
| C0031 | 29/09/2005 | Coenvíos con integridad 100% |
| C0008 | 17/10/2005 | Pedidos EDI internacional desde COCPED |
| C0088 | 14/01/2006 | Presupuestos ECA |
| FQ365 | 20/03/2009 | No confirmar si excede riesgo |
| FQ284 | 28/01/2010 | Fluidra Direct |
| 3003A | 30/03/2011 | Reservar al confirmar (por setup) |
| FQ799 | 04/03/2011 | Auditoría de pedidos |
| FQ937 | 28/02/2012 | Ajustar fechas coenvíos (opción RP) |
| 06B12 | 06/11/2012 | Fecha confirmada = la más desfavorable |
| C1997 | 24/09/2014 | Email confirmación pedido |
| F-2OL | 28/09/2021 | Abrir a más de un OL |

---

## CO0140 — Entrada de líneas de pedido

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Nombre objeto** | CO0140 |
| **Fichero fuente** | CO0140.MBR |
| **Copyright** | Aquapoint S.A. → Fluidra S.A. |
| **Fecha creación** | 23/02/1998 |
| **Tipo** | Programa interactivo — Subfile |
| **Líneas de código** | ~21.369 |

### Función
Entrada y modificación de las líneas de detalle de un pedido de cliente. Gestiona artículos, cantidades, precios, descuentos, fechas, ecotasa y compatibilidades entre artículos.

### Parámetros clave

| Parámetro | Dir. | Descripción |
|-----------|------|-------------|
| `@PERRO` (entrada) | E | Si = '1' → confirma la orden directamente sin mostrar detalle |
| `@PCIAS` | E | Empresa |
| `@PDLGA` | E | Delegación |
| `@PNPCO` | E | Número de pedido |
| `@PTPED` | E | Tipo de pedido |
| `@PDATP` | E | Fecha del pedido |
| `@PERRO` (salida) | S | `' '` = OK, `'A'` = cancelado |

### Control de flujo ZXOPT

Idéntico a CO0120:
- ZXOPT1: `Snn` (subfile), `Pnn` (panel), `Wnn` (ventana), `END`
- ZXOPT2: `INZ`, `DSP`, `CHK`, `CLR`, `UPD`

### Modificaciones relevantes

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| 24092 | 24/09/2002 | Error al recuperar idioma de artículos |
| ECOP | 23/03/2007 | Tratamiento ecotasa |
| 09074 | 09/07/2004 | Añadir opción NF (nueva facturación) |
| C0031 | 29/09/2005 | Coenvíos con integridad 100% |
| C0008 | 14/10/2005 | Pedidos EDI internacional |
| C0074 | 15/03/2006 | Control compatibilidades/incompatibilidades ADR |
| 14066 | 14/06/2006 | Costes adicionales para cálculo de rendimiento |
| 04098 | 04/09/2008 | Aviso rendimiento < mínimo al confirmar |
| FQ284 | 28/01/2010 | Fluidra Direct |
| 31712 | 31/07/2012 | Conjuntos: calcular plazo por componente |
| 04712 | 04/07/2012 | Ajuste frecuencia semanal para coenvíos parciales |
| 26712 | 26/07/2012 | Excluir artículos de servicio de coenvíos |

---

## CO2519 — Lanzamiento de órdenes de venta

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Nombre objeto** | CO2519 |
| **Fichero fuente** | CO2519.MBR |
| **Copyright** | Aquapoint S.A. → Fluidra S.A. |
| **Fecha creación** | 17/10/1997 |
| **Tipo** | Programa de servicio / proceso batch o interactivo |
| **Líneas de código** | ~1.921 |

### Función
Genera el número de orden de venta y lanza la orden al sistema de ejecución. Gestiona la numeración de órdenes, integridad de coenvíos, fechas confirmadas y envío de confirmación por email.

### Parámetros de entrada (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PECIAS` | Num | E | Empresa |
| 2 | `@PEDLGA` | Alfa | E | Delegación |
| 3 | `@PENPCO` | Alfa | E | Número de pedido |
| 4 | `@PETPED` | Alfa | E | Tipo de pedido |
| 5 | `@PEDATP` | Alfa | E | Fecha del pedido |
| 6 | `@PENOPL` | Alfa | E | Número de línea *(C0031)* |
| 7 | `@PEMAGA` | Alfa | E | Almacén |
| 8 | `@PEARTI` | Alfa | E | Artículo |
| 9 | `@PEVARI` | Alfa | E | Variante |
| 10 | `@PEMODE` | Alfa | E | Modelo |
| 11 | `@PETCOJ` | Alfa | E | Tipo de conjunto |
| 12 | `@PECANT` | Num | E | Cantidad |
| 13 | `@PEUNIM` | Alfa | E | Unidad de medida |
| 14 | `@PETIPO` | 1A | E | Tipo de operación |
| — | `@PCIAS` | Num | E | Empresa (contexto general) |
| — | `@PDLGA` | Alfa | E | Delegación (contexto general) |
| — | `@PNPCO` | Alfa | E/S | Número de pedido resultado |
| — | `@PTPED` | Alfa | E | Tipo de pedido |
| — | `@PDATP` | Alfa | E | Fecha del pedido |
| — | `@PNOPL` | Alfa | E | Número de línea |
| — | `@PERRO` | 1A | E/S | Modo/resultado (ver tabla) |
| — | `@PEERR` | 1A | S | Error secundario |

### Códigos @PERRO (entrada — modo de operación)

| Valor | Operación solicitada |
|-------|---------------------|
| `' '` | Solo crear número de orden |
| `'1'` | Solo lanzar orden (número ya existe) |
| `'2'` | Crear número de orden Y lanzar |

### Códigos @PERRO (salida — resultado)

| Valor | Significado |
|-------|-------------|
| `' '` | OK — operación completada correctamente |
| `'1'` | Error — no se ha podido localizar el número de orden |

### Búsqueda de numeración (tabla NPVE / PMTABD)

La clave de búsqueda en PECLTA es de 10 caracteres:

```
Posiciones 1-3:   CIAS  (3 caracteres empresa)
Posiciones 4-8:   DLGA  (5 caracteres delegación)
Posiciones 9-10:  TPED  (2 caracteres tipo pedido)
```

Jerarquía de búsqueda (de más específica a menos):
1. `CIAS + DLGA + TPED`
2. `CIAS + DLGA + '  '`
3. `CIAS + '     ' + TPED`
4. `CIAS + '     ' + '  '`

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| PMTABL | IF/K DISK | Tablas de numeración (maestro) |
| PMTABD | UF/K DISK | Detalle tablas / numeración |
| COCAPE | UF/K DISK | Cabecera de pedido |
| CODIPE | UF/K DISK | Detalle de pedido |
| COOBPE | UF/K DISK | Observaciones del pedido |
| COPECO | UF/K DISK | Condiciones del pedido |
| CODEPE | UF/K DISK | Líneas del pedido |
| COGAPE | UF/K DISK | Agrupaciones |
| COCAIM | UF/K DISK | Cabecera importes |
| CODEP2 | UF/K DISK | Detalle vista 2 |
| COCLIN | IF/K DISK (PREFIX CL_) | Maestro de clientes |
| MSLOAD | IF/K DISK | Almacenes logísticos |
| MSALMA08 | IF/K DISK (USROPN) | Almacenes Fluido Direct *(F-2OL)* |
| PMUNAL | IF/K DISK | Unidades de medida *(28099)* |
| PMSETP | IF/K DISK | Parámetros de setup *(3003A)* |

### Modificaciones relevantes

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| C0031 | 29/09/2005 | Coenvíos con integridad de envío 100% |
| 28099 | 28/09/2009 | Tratamiento nivel de servicio |
| 18129 | 18/12/2009 | Modificar plazo en base a NDS del almacén |
| FQ284 | 08/02/2010 | Fluidra Direct |
| 01041 | 01/04/2011 | Grabar FREP=FLAN si es menor |
| 25051 | 25/05/2011 | Cálculo fecha de entrega por DPM (Sr. Rifà) |
| 06B12 | 06/11/2012 | Fecha confirmada = la más desfavorable |
| CR839 | 30/04/2013 | Lista valoración rotura stock |
| C1997 | 24/09/2014 | Email confirmación pedido (vía UT2725) |
| C2549 | 05/05/2015 | Solucionar incidencia tipos de plazo |
| F-2OL | 28/09/2021 | Abrir a más de un OL |
| PA064 | 09/09/2021 | Roturas clientes terceros en pedidos AS400/M3 |
| PGE21 | 15/12/2025 | Actualizar auditoría digital en CODEPE |

---

## CO2593 — Parámetros de impresión (PURLIBC8)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras |
| **Nombre objeto** | CO2593 (nombre alternativo: PURLIBC8) |
| **Fichero fuente** | CO2593.MBR |
| **Fecha creación** | No indicada (modificación más antigua: 09/02/2004) |
| **Tipo** | Programa interactivo — Panel (con soporte batch) |
| **Líneas de código** | ~380 |

### Función
Presenta una pantalla para que el usuario configure los parámetros de impresión de documentos (formato de exportación, número de copias, impresora). Soporta ejecución en modo batch (sin pantalla) para llamadas desde SalesForce API.

### Parámetros de entrada (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PCIAS` | Alfa | E | Empresa |
| 2 | `@PDLGA` | Alfa | E | Delegación |
| 3 | `@PPROG` | 10A | E | Nombre del programa invocador |
| 4 | `@PCCIT` | 7A | E | Título de la pantalla |
| 5 | `@PGEXP` | Num | E/S | Formato de exportación |
| 6 | `@PNPIC` | Num | E/S | Número de copias |
| 7 | `@PPEPE` | Alfa | E/S | Impresora (dispositivo) |
| 8 | `@PPAR1` | 1A | E | `'1'` = no mostrar pantalla (batch) |
| 9 | `@PPAR2` | 1A | E | Libre |
| 10 | `@PJOBT` | 18A | E | Nombre del trabajo |
| 11 | `@PERRO` | 1A | S | Resultado (ver tabla) |

### Códigos @PERRO (salida)

| Valor | Significado |
|-------|-------------|
| `' '` | OK — parámetros aceptados |
| `'A'` | Usuario canceló (F3/F12) |
| `'3'` | Error de validación de un campo |

### Comportamiento según modo de ejecución

| Condición | Comportamiento |
|-----------|---------------|
| Trabajo interactivo (`@PTIPJ >= '1'`) | Abre y muestra pantalla CO2593FM |
| Trabajo batch / QUSER (`@PTIPJ = '0'`) | No abre pantalla; usa @PPAR1='1' implícito |
| Recibe `@PPAR1='1'` en interactivo | No muestra pantalla en primera pasada (solo si hay error) |
| Error en validación + batch | Sale con error sin mostrar pantalla |
| Error en validación + interactivo | Muestra pantalla con el error resaltado |

### Llamadas a subprogramas

| CALL | Parámetros clave | Descripción |
|------|-----------------|-------------|
| `RTVTIJ` | `@PTIPJ` | Detecta tipo de trabajo (batch/interactivo) |
| `MMPROM` | `@PEOPT`, `'SINO    '`, `@PEERR` | Valida campo contra diccionario |
| `MM2559` | `@PCIAS`, `@PDLGA`, `@PPROG`, `'FMTO    '`, `'01'`, `'LCM0035'` | Verifica impresora |

### Subrutinas principales

| Subrutina | Función |
|-----------|---------|
| `SUBCLR` | Limpia indicadores → `INZ` |
| `SUBINZ` | Inicializa valores en pantalla |
| `SUBDSP` | Presenta pantalla (EXFMT W01) si no está en modo silencioso |
| `SUBCMD` | Gestiona teclas: F3/F12 → `@PERRO='A'`; HELP(F4) → `SUBF4` |
| `SUBF4` | Lanza MMPROM para selección asistida del campo activo |
| `SUBCHK` | Valida los tres campos (GEXP, NPIC, PEPE) contra MMPROM y verifica impresora |
| `SUBUPD` | Devuelve los valores confirmados en los parámetros de salida |
| `CS_POCUR2` | Extrae fila/columna del cursor desde FMTPOCUR |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| 09024 | 09/02/2004 | Eliminar fichero PMPROM |
| B0251 | 02/06/2020 | Soporte SalesForce API: detectar batch y no abrir pantalla |

---

## CO2611 — Sumar días laborables a una fecha

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras — Utilitario |
| **Nombre objeto** | CO2611 |
| **Fichero fuente** | CO2611.MBR |
| **Copyright** | Fluidra S.A. |
| **Fecha creación** | 06/10/2010 |
| **Autor** | KIKE |
| **Tipo** | Subprograma de servicio (sin pantalla) |
| **Líneas de código** | ~200 |

### Función
Dado una fecha de inicio y un número de días laborables, calcula la fecha resultante excluyendo sábados y domingos.

> **Limitación conocida:** Solo excluye sábados y domingos. No considera festivos nacionales ni locales. El propio código fuente documenta esta limitación.

### Parámetros (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PEDAT1` | 8,0 | E | Fecha de inicio (formato YYYYMMDD) |
| 2 | `@PEDAYS` | 5,0 | E | Número de días laborables a añadir |
| 3 | `@PEDAT2` | 8,0 | S | Fecha resultado (formato YYYYMMDD) |
| 4 | `@PEERRO` | 1A | S | `' '` = OK; `'1'` = Error (fecha inválida) |

### Algoritmo (mod. CR191)

```
1. Obtener día de la semana de @PEDAT1 mediante @GETWEEK
   (1=Lunes, 2=Martes, ..., 5=Viernes, 6=Sábado, 0=Domingo)

2. Si inicio es sábado (PM_DAY=6):
   - PM_DAY = 1 (lunes)
   - ZX_ADD += 2 (avanzar al lunes)

3. Si inicio es domingo (PM_DAY=0):
   - PM_DAY = 1 (lunes)
   - ZX_ADD += 1 (avanzar al lunes)

4. Calcular semanas y resto:
   zx_semanas5 = @PEDAYS / 5
   ZX_resto = @PEDAYS MOD 5

5. Calcular días naturales base:
   ZX_ADD += zx_semanas5 * 7 + ZX_resto

6. Ajuste adicional si el resto hace cruzar fin de semana:
   | PM_DAY | Necesita +2 si ZX_resto > |
   |--------|---------------------------|
   | 1 (L)  | Nunca                     |
   | 2 (M)  | 3                         |
   | 3 (X)  | 2                         |
   | 4 (J)  | 1                         |
   | 5 (V)  | 0 (siempre si resto > 0)  |

7. ADDDUR ZX_ADD días a @PEDAT1 → fecha intermedia

8. Verificar que la fecha resultado no caiga en fin de semana:
   - Si sábado: +2 días
   - Si domingo: +1 día

9. MOVE fecha resultado → @PEDAT2
```

### Llamadas a subprogramas

| CALL | Descripción |
|------|-------------|
| `@GETWEEK` | Obtiene el día de la semana de una fecha (1=Lunes, 0=Domingo) |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| C0088/CR191 | 03/05/2013 | Corrección del algoritmo de cálculo (JMARTINEZP) |

---

## CO2696 — Validación de coenvíos

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras — Utilitario |
| **Nombre objeto** | CO2696 |
| **Fichero fuente** | CO2696.MBR |
| **Copyright** | Aquapoint S.A. |
| **Fecha creación** | Febrero 2006 |
| **Autor** | Carlos del Valle |
| **Tipo** | Subprograma de servicio (sin pantalla) |
| **Líneas de código** | ~679 |

### Función
Comprueba la validez de un coenvío para su expedición. Un coenvío es un grupo de líneas que deben expedirse conjuntamente al 100%.

### Parámetros (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PCIAS` | Num | E | Empresa |
| 2 | `@PARTI` | Alfa | E | Código de artículo |
| 3 | `@PVARI` | Alfa | E | Variante del artículo |
| 4 | `@PMODE` | Alfa | E | Modelo del artículo |
| 5 | `@PERRO` | 1A | S | Código de resultado |

### Códigos de retorno (@PERRO)

| Valor | Significado |
|-------|-------------|
| `' '` | OK — puede expedirse |
| `'0'` | OK — no es un coenvío (no aplica validación) |
| `'1'` | Error — la línea no está reservada al 100% |
| `'2'` | Error — faltan líneas del coenvío por reservar |
| `'3'` | Atención — todas las líneas presentes pero alguna con reserva parcial |
| `'4'` | Error — el coenvío está OK pero su dependiente anterior no |

> **Nota:** El código tiene preparado (comentado) el valor `'E'` para cuando la empresa no permita reservar conjuntos combinados, pendiente de implementar campo P7NRCC en PMEMPR.

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| COCLIN | IF/K DISK | Maestro de clientes / cabecera de pedido |
| CODEPE | IF/K DISK | Líneas del pedido |

---

## CO2733 — Situación distribución conjuntos de venta

| Atributo | Valor |
|----------|-------|
| **Módulo** | Compras — Utilitario |
| **Nombre objeto** | CO2733 |
| **Fichero fuente** | CO2733.MBR |
| **Función original** | FDX01 — Fluidra Direct |
| **Fecha creación** | 03/01/2013 |
| **Autor** | Carlos del Valle |
| **Tipo** | Subprograma de servicio (sin pantalla) |
| **Líneas de código** | ~173 |

### Función
Determina la situación de distribución de un conjunto de venta: si todos sus componentes se almacenan en Trace (SIOL='2'), ninguno, o es un conjunto combinado.

### Parámetros (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PCIAS` | Num | E | Empresa |
| 2 | `@PARTI` | Alfa | E | Código de artículo |
| 3 | `@PVARI` | Alfa | E | Variante |
| 4 | `@PMODE` | Alfa | E | Modelo |
| 5 | `@PERRO` | 1A | S | Código de resultado |

### Códigos de retorno (@PERRO)

| Valor | Significado |
|-------|-------------|
| `'1'` | Todos los componentes se almacenan en Trace (SIOL='2') → MTS puro |
| `'2'` | Ningún componente se almacena en Trace → MTO puro |
| `'3'` | Conjunto combinado: mezcla de componentes Trace y no-Trace |
| `'4'` | No es un conjunto de venta |

### Lógica de detección de conjunto

```
1. Buscar en MSCOJC con TCOJ='4' (conjunto de venta)
   → Si NO existe: @PERRO='4' (no es conjunto)

2. Buscar en MSCOJC con TCOJ='1' (conjunto de compra)
   → Si TAMBIÉN es conjunto de compra: ZXTCOJ=' ' (no se trata como conjunto de distribución)
   → Si SOLO es conjunto de venta: ZXTCOJ='4' (se procesa)

3. Para cada componente (MSCOJD donde TCOJ='4' y BAJA<>'1'):
   → Leer MSARTI00 del componente
   → Si M5SIOL='2': ZXSIOL++ (componente en Trace)
   → Si M5SIOL<>'2': ZXNOOL++ (componente NO en Trace)

4. Resultado:
   ZXSIOL>0 y ZXNOOL>0 → @PERRO='3' (combinado)
   ZXSIOL>0 y ZXNOOL=0 → @PERRO='1' (todos en Trace)
   ZXSIOL=0 y ZXNOOL=0 → @PERRO='4' (sin componentes válidos)
   ZXSIOL=0 y ZXNOOL>0 → @PERRO='2' (ninguno en Trace)
```

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| MSARTI00 | IF/K DISK | Maestro de artículos (campo SIOL) |
| MSCOJC | IF/K DISK | Cabecera de conjuntos |
| MSCOJD | IF/K DISK | Componentes de conjuntos |

### Claves de acceso

| Clave | Campos |
|-------|--------|
| KMSARTICO | @PCIAS + NDCOMP + NDVAR2 + NDMOD2 |
| KMSCOJC | NDCIAS + NDARTI + NDVARI + NDMODE + NDTCOJ |

---

## CO2755 — Categoría de rotación de artículo

| Atributo | Valor |
|----------|-------|
| **Módulo** | Seguridad / Clasificación |
| **Nombre objeto** | CO2755 |
| **Fichero fuente** | CO2755.MBR |
| **Copyright** | Fluidra Services |
| **Fecha creación** | 14/05/2010 |
| **Autor** | X.BOU |
| **Tipo** | Subprograma de servicio (sin pantalla) |
| **Líneas de código** | ~523 |

### Función
Determina la categoría de rotación de un artículo (MTS/MTO y subcategorías A/B/C) según los parámetros de empresa y el comportamiento del artículo en stock y almacén.

### Parámetros (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PCIAS` | Num | E | Empresa |
| 2 | `@PDLGA` | Alfa | E | Delegación |
| 3 | `@PLOGE` | Alfa | E | Almacén lógico |
| 4 | `@PARTI` | Alfa | E | Código de artículo |
| 5 | `@PVARI` | Alfa | E | Variante |
| 6 | `@PMODE` | Alfa | E | Modelo |
| 7 | `@PTCOJ` | 1A | E | Tipo de conjunto |
| 8 | `@PCROT` | 2A | S | Categoría de rotación resultante |
| 9 | `@PCROTA` | 2A | S | Categoría rotación umbral A |
| 10 | `@PCROTB` | 2A | S | Categoría rotación umbral B |
| 11 | `@PCROTC` | 2A | S | Categoría rotación umbral C |
| 12 | `@PLOFD` | 1A | E | Indicador Fluido Direct *(opcional — parámetro 12, mod. CR964)* |

> **Nota:** El parámetro 12 (@PLOFD) es opcional. El programa comprueba `%PARMS=12` para determinar si se ha pasado.

### Estructura de parámetros de empresa (PPAMPL)

| Campo | Posición | Tipo | Descripción |
|-------|----------|------|-------------|
| `PPCROT` | 1 | 1A | Indicador de uso de rotación |
| `PPRISK` | 2 | 1A | Nivel de riesgo |
| `PPDICO` | 3 | 1A | Diccionario |
| `PPCONI` | 4 | 1A | Condición |
| `PPOBJT` | 5 | 1A | Objetivo |
| `PPCROP` | 6 | 1A | Categoría propuesta |
| `PPBBAC` | 7-16 | 10A | Umbral bajo — categoría A |
| `PPBBAP` | 17-26 | 10A | Umbral alto — categoría A |
| `PPEFIN` | 27 | 1A | Indicador fin |
| `PPEQAR` | 28 | 1A | Igualar artículo |
| `PPDWVT` | 29 | 1A | Flag control venta |
| `PPDWCT` | 30 | 1A | Flag control coste |
| `PPDWPU` | 31 | 1A | Flag control compra |
| `PPDWST` | 32 | 1A | Flag control stock |
| `PPDWCV` | 33 | 1A | Flag control conversión |
| `PPDWFB` | 34 | 1A | Flag control feedback |
| `PPBLAN` | 35-500 | 466A | Reservado (blancos) |

### Campos extendidos de artículo (M5AMPL)

| Campo | Posición | Descripción |
|-------|----------|-------------|
| `M5CTEC` | 1 | Categoría técnica |
| `M5PNAT` | 2-3 | Procedencia/naturaleza |
| `M5AUES` | 5 | Indicador especial |
| `M5OMP` | 7 | OMP (Order Management Parameter) |
| `M5ECOP` | 8 | Ecotasa |
| `M5CLOT` | 44 | Clasificación lote |
| `M5CROB` | 46 | Categoría de rotación base |
| `m5TpOb` | 47-48 | Tipo objetivo |
| `m5PrOb` | 49-50 | Prioridad objetivo |
| `M5CROP` | 60-61 | Categoría de rotación propuesta |

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| PMLIBL | IF/K DISK | Parámetros de biblioteca |
| PMEMPR | IF/K DISK | Parámetros de empresa |
| MSLOAD | IF/K DISK | Almacenes logísticos |
| MSSTOK | IF/K DISK | Stock por almacén |
| MSSTST | IF/K DISK | Stock estadístico *(C1957)* |
| MSCOJD | IF/K DISK | Componentes de conjuntos |
| MSARTI00 | IF/K DISK | Maestro de artículos |
| MSALMA05 | IF/K DISK | Almacenes (lógico 05) *(CR964)* |

### Claves de acceso

| Clave | Campos |
|-------|--------|
| KMSSTOK | M1CIAS + M1MAGA + M1ARTI + M1VARI + M1MODE |
| KMSSTST *(C1957)* | M6CIAS + M6DLGA + M6ARTI + M6VARI + M6MODE + M6NETL |
| KMSLOAD | MICIAS + MIDLGA + MILGCA |
| KMSCOJD | NDCIAS + NDARTI + NDVARI + NDMODE + NDTCOJ |
| KMSARTI00 | M5CIAS + M5ARTI + M5VARI + M5MODE |
| KMSALMA05 *(CR964)* | (campos de almacén) |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| 01090 | 01/09/2010 | Eliminar acceso a LDA; usar PMEMPR para recuperar empresa |
| FQ709 | 01/12/2010 | Rellenar rotación en líneas de pedido de compra |
| CR964 | 17/05/2013 | Modificaciones cubos/estadísticas |
| 08835 | 26/11/2025 | Corrección clasificación MTS/MTO incorrecta en Fluido Direct (SCE-104) |

---

## CO2610 / CO2770 — Pedidos pendientes de servir (KORE)

| Atributo | Valor |
|----------|-------|
| **Módulo** | KORE |
| **Nombre objeto** | CO2610 (referenciado como CO2770 en librería) |
| **Fichero fuente** | CO2770.MBR |
| **Copyright** | Fluidra S.A. |
| **Fecha creación** | 07/03/2013 |
| **Tipo** | Subprograma de servicio (sin pantalla) |
| **Líneas de código** | ~218 |

### Función
Para un artículo dado y un rango de fechas, calcula la cantidad total de unidades pendientes de servir en pedidos de cliente activos. Invocado por el motor KORE para cálculo de disponibilidad.

### Parámetros (*ENTRY PLIST)

| # | Parámetro | Tipo | E/S | Descripción |
|---|-----------|------|-----|-------------|
| 1 | `@PCIAS` | Num | E | Empresa |
| 2 | `@PDLGA` | Alfa | E | Delegación |
| 3 | `@PARTI` | Alfa | E | Código de artículo |
| 4 | `@PVARI` | Alfa | E | Variante |
| 5 | `@PMODE` | Alfa | E | Modelo |
| 6 | `@PTCOJ` | Alfa | E | Tipo de conjunto |
| 7 | `@PDATD` | 8,0 | E | Fecha desde (YYYYMMDD) |
| 8 | `@PDATH` | 8,0 | E | Fecha hasta (YYYYMMDD) |
| 9 | `@PCANT` | Num | E/S | Cantidad acumulada (entrada: valor inicial) |
| 10 | `@PCAN1` | Num | S | Cantidad resultado |
| 11 | `@PLOGE` | Alfa | E | Almacén lógico |
| 12 | `@POMFD` | 1A | E | `'1'` = excluir pedidos modificados (S5TPE4='2') |
| 13 | `@PERRO` | 1A | S | Código de error (reservado) |

### Lógica de cálculo

```
Para cada línea de pedido (CODEPE42) del artículo @PARTI:

  1. Filtrar por fecha: @PDATD <= S5FRRR <= @PDATH

  2. Si @POMFD='1' y S5TPE4='2' → saltar línea (pedido modificado)

  3. Verificar que el pedido existe en COCAPEA8 (pedido activo)
     → Si no existe en cabeceras activas → saltar línea

  4. Calcular cantidad pendiente:
     ZXCANT = S5CANT (cantidad pedida)
     Si S5CANC > 0: ZXCANT = S5CANC (usa cantidad cancelada si existe)
     ZXCANT -= S5CANS (restar cantidad ya servida)

  5. Si unidad pedido (S5UNIM) ≠ unidad base artículo (M5UNIM):
     → Convertir ZXCANT a unidad base mediante CS_CNVUBS (tabla PMUNAL)

  6. Si ZXCANT > 0: @PCANT += ZXCANT

Resultado final: @PCANT = cantidad total pendiente en unidad base
```

### Subrutina de conversión CS_CNVUBS

```
Entrada: ZXAUXIBS (cantidad), ZXARTIBS, ZXVARIBS, ZXMODEBS, ZXUNIMBS

1. CHAIN PMUNAL con clave: CIAS + ARTI + VARI + MODE + UNIM
2. Si no encuentra: VARI='', MODE='' → CHAIN PMUNAL (intento sin variante/modelo)
3. Si P6TCOF <> '1': resultado = XZAUXI × P6COFA  (multiplicar)
4. Si P6TCOF = '1' y P6COFA ≠ 0: resultado = XZAUXI / P6COFA  (dividir)
5. Aplicar ajuste decimal (COPY SRD001)

Salida: ZXAUXIBS (cantidad convertida)
```

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| CODEPE42 | IF/K DISK | Detalle pedidos — lógico por artículo |
| COCAPEA8 | IF/K DISK | Cabecera pedidos activos |
| MSLOAD | IF/K DISK | Almacenes logísticos |
| MSARTI00 | IF/K DISK | Maestro de artículos (unidad base) |
| PMUNAL | IF/K DISK | Unidades de medida y factores de conversión |

### Claves de acceso

| Clave | Campos (en CODEPE42) |
|-------|---------------------|
| KCODEPE42 | S5CIAS + S5ARTI + S5VARI + S5MODE + S5TCOJ + S5LOGE + S5FRRR |
| KCODEPE42_S | S5CIAS + S5ARTI + S5VARI + S5MODE + S5TCOJ + S5LOGE |
| KCOCAPEA8 | S5CIAS + S5DLGA + S5NPCO + S5TPED + S5DATP + S5NOPL |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| I1021 | — | Ajuste uso de S5DATD/S5DATH como fecha de referencia |

---

## DO0001 — Lista prioridades DOM nivel delegación (v1)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Logística / DOM |
| **Nombre objeto** | DO0001 |
| **Fichero fuente** | DO0001.MBR |
| **Copyright** | Fluidra |
| **Fecha creación** | 16/01/2014 |
| **Tipo** | Programa interactivo — Subfile |
| **Pantalla** | DO0001FM |
| **Líneas de código** | ~1.405 |

### Función
Lista paginada de reglas de prioridad DOM a nivel empresa/almacén, mostrando también las reglas de delegación. Permite gestionar las prioridades del motor de distribución.

### Control de flujo ZXOPT

| ZXOPT1 | Proceso |
|--------|---------|
| `Snn` | Subfile N — lista de reglas de prioridad |
| `END` | Fin del programa |

| ZXOPT2 | Rutina |
|--------|--------|
| `INZ` | Inicializar y cargar el subfile con reglas de EDPAR4 |
| `DSP` | Visualizar la lista |
| `CHK` | Validar teclas / opciones |
| `CLR` | Limpiar subfile |
| `UPD` | Procesar acción (→ llama DO0005/DO0025) |

### Organización de EDPAR4

```
TREG=0   Prioridades a nivel de empresa (general)
TREG=1   Excepción a nivel de proveedor
TREG=2   Excepción a nivel de artículo
TREG=3   Excepción a nivel de familia/subfamilia
TREG=4   Excepción a nivel de familia/tarifa
TREG=5   Excepción a nivel de cliente
TREG=A   Prioridades a nivel de delegación (general)
TREG=B   Excepción a nivel de DLGA/proveedor
TREG=C   Excepción a nivel de DLGA/artículo
TREG=D   Excepción a nivel de DLGA/familia
TREG=E   Excepción a nivel de DLGA/fam-subfamilia
TREG=F   Excepción a nivel de DLGA/fam-tarifa
TREG=G   Excepción a nivel de DLGA/cliente
TREG=H   Excepción a nivel de DLGA/procedimiento de pedido (solo DO0001)
```

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| DO0001FM | CF/WORKSTN | Pantalla con SFILE SUFD |
| EDPARM | IF/K DISK | Parámetros DOM empresa |
| EDPAR2 | IF/K DISK | Datos secundarios prioridades |
| EDPAR4 | UF/K DISK | Prioridades de distribución (principal) |
| PMLIBL | IF/K DISK | Parámetros de biblioteca |
| PMEMPR | IF/K DISK | Parámetros de empresa |
| MSALMA | IF/K DISK | Maestro de almacenes |
| MSSGGA | IF/K DISK | Segmentos de almacén |
| PMSETP | UF A/K DISK | Setup del sistema |
| MSARTI00 | IF/K DISK | Maestro de artículos |
| PUPROV | IF/K DISK | Maestro de proveedores |
| MSFAMI | IF/K DISK | Maestro de familias |
| PMTABD | IF/K DISK | Tablas de dominio |
| COCLIN | IF/K DISK | Maestro de clientes |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| IT244 | 27/05/2022 | 2ª fase 2OL — abrir a más de un OL |

---

## DO0005 — Panel prioridades DOM nivel DLGA (v1)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Logística / DOM |
| **Nombre objeto** | DO0005 |
| **Fichero fuente** | DO0005.MBR |
| **Copyright** | Fluidra Services |
| **Fecha creación** | 18/03/2010 |
| **Tipo** | Programa interactivo — Panel (formulario) |
| **Pantalla** | DO0005FM |
| **Operaciones** | DSP / CRT / DLT / CPY / CHG |
| **Líneas de código** | ~1.392 |

### Función
Formulario de alta, baja, copia y modificación de reglas de prioridad DOM a nivel de delegación. Es el programa de detalle invocado desde DO0001.

### Operaciones soportadas

| Operación | Descripción |
|-----------|-------------|
| `DSP` | Visualizar una regla existente (solo lectura) |
| `CRT` | Crear nueva regla de prioridad |
| `DLT` | Eliminar regla existente |
| `CPY` | Copiar regla a otra empresa/almacén/delegación |
| `CHG` | Modificar campos de una regla existente |

### Control de flujo ZXOPT

| ZXOPT1 | Proceso |
|--------|---------|
| `Pnn` | Formato de pantalla N (formulario de detalle) |
| `Wnn` | Ventana N (popup auxiliar) |
| `END` | Fin del programa |

| ZXOPT2 | Rutina |
|--------|--------|
| `INZ` | Inicializar el formulario |
| `DSP` | Visualizar pantalla (EXFMT) |
| `CMD` | Validar teclas de función |
| `CHK` | Validar campos del formulario |
| `CLR` | Limpiar la pantalla |
| `UPD` | Grabar cambios en EDPAR4 |

### Ficheros utilizados

| Fichero | Modo | Descripción |
|---------|------|-------------|
| DO0005FM | CF/WORKSTN | Pantalla de formulario |
| EDPAR2 | IF/K DISK | Datos secundarios prioridades |
| EDPAR4 | UF A/K DISK | Prioridades de distribución |
| EDHPA4 | O/K DISK | Histórico de cambios |
| PMEMPR | IF/K DISK | Parámetros de empresa |
| PMTAB2 | IF/K DISK | Tablas secundarias |
| MSARTI00 | IF/K DISK | Maestro de artículos |
| PMTABD | IF/K DISK | Tablas de dominio |
| MSALMA | IF/K DISK | Maestro de almacenes |
| PUPROV | IF/K DISK | Maestro de proveedores |
| MSFAMI | IF/K DISK | Maestro de familias |
| COCLIN | IF/K DISK | Maestro de clientes |

### Modificaciones

| Marca | Fecha | Descripción |
|-------|-------|-------------|
| C3758 | 08/11/2017 | Añadir excepciones DOM a nivel de artículo |

---

## DO0010 — Parámetros Fluido Direct

| Atributo | Valor |
|----------|-------|
| **Módulo** | Logística |
| **Nombre objeto** | DO0010 |
| **Fichero fuente** | DO0010.MBR |
| **Copyright** | Fluidra Services |
| **Fecha creación** | 05/04/2005 |
| **Tipo** | Programa interactivo — Subfile + Panel |
| **Líneas de código** | ~1.329 |

### Función
Mantenimiento de los parámetros de configuración del sistema Fluido Direct. Controla la distribución directa desde empresa productiva a cliente final.

### Control de flujo ZXOPT

| ZXOPT1 | Proceso |
|--------|---------|
| `Snn` | Subfile N — lista de parámetros |
| `Wnn` | Ventana N — formulario de parámetro |
| `END` | Fin del programa |

| ZXOPT2 | Rutina |
|--------|--------|
| `INZ` | Inicializar |
| `DSP` | Visualizar pantalla |
| `CHK` | Validar |
| `CLR` | Limpiar |
| `UPD` | Actualizar ficheros |

---

## DO0020 — Lista prioridades DOM nivel delegación (v2)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Logística / DOM |
| **Nombre objeto** | DO0020 |
| **Fichero fuente** | DO0020.MBR |
| **Copyright** | Fluidra |
| **Fecha creación** | 16/01/2014 |
| **Tipo** | Programa interactivo — Subfile |
| **Líneas de código** | ~1.591 |

### Función
Variante de DO0001. Lista de reglas de prioridad DOM a nivel delegación. Comparte la misma lógica y estructura que DO0001, con diferencias menores en el diseño de la pantalla o configuración de opciones.

**Referencia:** Ver [DO0001](#do0001--lista-prioridades-dom-nivel-delegación-v1) para la documentación completa.

---

## DO0025 — Panel prioridades DOM nivel DLGA (v2)

| Atributo | Valor |
|----------|-------|
| **Módulo** | Logística / DOM |
| **Nombre objeto** | DO0025 |
| **Fichero fuente** | DO0025.MBR |
| **Copyright** | Fluidra Services |
| **Fecha creación** | 18/03/2010 |
| **Tipo** | Programa interactivo — Panel (formulario) |
| **Operaciones** | DSP / CRT / DLT / CPY / CHG |
| **Líneas de código** | ~1.482 |

### Función
Variante de DO0005. Formulario de alta/baja/copia/modificación de reglas de prioridad DOM nivel DLGA, con opciones extendidas respecto a DO0005.

**Referencia:** Ver [DO0005](#do0005--panel-prioridades-dom-nivel-dlga-v1) para la documentación base.

---

## Tabla resumen de todos los programas

| Programa | Módulo | Tipo | Pantalla | Líneas | Fecha Creación |
|----------|--------|------|----------|--------|----------------|
| CM0085 | Compras | SFL interactivo | CM0085FM | ~2.887 | 23/07/1997 |
| CO0120 | Compras | SFL + Panel | (varios) | ~12.565 | 23/07/1997 |
| CO0140 | Compras | SFL interactivo | (varios) | ~21.369 | 23/02/1998 |
| CO2519 | Compras | Proceso/batch | — | ~1.921 | 17/10/1997 |
| CO2593 | Compras | Panel (batch-safe) | CO2593FM | ~380 | ~2004 |
| CO2611 | Utilitarios | Servicio | — | ~200 | 06/10/2010 |
| CO2696 | Compras | Servicio | — | ~679 | Feb 2006 |
| CO2733 | Utilitarios | Servicio | — | ~173 | 03/01/2013 |
| CO2755 | Seguridad | Servicio | — | ~523 | 14/05/2010 |
| CO2610 | KORE | Servicio | — | ~218 | 07/03/2013 |
| DO0001 | Logística | SFL interactivo | DO0001FM | ~1.405 | 16/01/2014 |
| DO0005 | Logística | Panel | DO0005FM | ~1.392 | 18/03/2010 |
| DO0010 | Logística | SFL + Panel | (varios) | ~1.329 | 05/04/2005 |
| DO0020 | Logística | SFL interactivo | DO0020FM | ~1.591 | 16/01/2014 |
| DO0025 | Logística | Panel | DO0025FM | ~1.482 | 18/03/2010 |

**Total: ~48.114 líneas de código fuente**

---

*Documentación generada a partir del código fuente. Versión 1.0 — Marzo 2026.*
