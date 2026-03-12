# Manual Funcional del Sistema AMS
## Sistema de Gestión de Pedidos y Logística — Fluidra S.A.

---

## Índice

1. [Módulo de Compras — Pedidos de Cliente](#1-módulo-de-compras--pedidos-de-cliente)
   - [1.1 Consulta y selección de pedidos (CM0085)](#11-consulta-y-selección-de-pedidos-cm0085)
   - [1.2 Mantenimiento de cabecera de pedido (CO0120)](#12-mantenimiento-de-cabecera-de-pedido-co0120)
   - [1.3 Entrada de líneas de pedido (CO0140)](#13-entrada-de-líneas-de-pedido-co0140)
   - [1.4 Lanzamiento de órdenes de venta (CO2519)](#14-lanzamiento-de-órdenes-de-venta-co2519)
   - [1.5 Validación de coenvíos (CO2696)](#15-validación-de-coenvíos-co2696)
   - [1.6 Configuración de impresión (CO2593)](#16-configuración-de-impresión-co2593)
2. [Módulo de Logística — DOM](#2-módulo-de-logística--dom)
   - [2.1 Prioridades nivel delegación — lista (DO0001 / DO0020)](#21-prioridades-nivel-delegación--lista-do0001--do0020)
   - [2.2 Prioridades nivel DLGA — panel (DO0005 / DO0025)](#22-prioridades-nivel-dlga--panel-do0005--do0025)
   - [2.3 Parámetros Fluido Direct (DO0010)](#23-parámetros-fluido-direct-do0010)
3. [Módulo de Soporte](#3-módulo-de-soporte)
   - [3.1 Cálculo de días laborables (CO2611)](#31-cálculo-de-días-laborables-co2611)
   - [3.2 Situación distribución conjuntos (CO2733)](#32-situación-distribución-conjuntos-co2733)
   - [3.3 Categoría de rotación de artículo (CO2755)](#33-categoría-de-rotación-de-artículo-co2755)
   - [3.4 Pedidos pendientes de servir — KORE (CO2770/CO2610)](#34-pedidos-pendientes-de-servir--kore-co2770co2610)
4. [Conceptos Clave y Glosario](#4-conceptos-clave-y-glosario)

---

## 1. Módulo de Compras — Pedidos de Cliente

### 1.1 Consulta y selección de pedidos (CM0085)

**Función:** Pantalla de lista (subfile) que muestra los pedidos de cliente activos. Permite al usuario localizar y seleccionar un pedido para ver o gestionar su detalle.

**Características principales:**
- Muestra pedidos en dos vistas seleccionables:
  - **Vista 1**: Código de cliente, fecha de pedido, tipo de pedido, nº pedido, referencia, moneda, estado
  - **Vista 2**: Descripción del cliente, importe, condiciones de riesgo (RCON)
  - **Vista 3** *(mod. 09022)*: Ordenación por pedido descendente
- Filtros de selección disponibles:
  - Por código de cliente
  - Por delegación (DLGA)
  - Por gestor de riesgos asignado *(mod. FH002)*
  - Por estado del pedido (incluye estado '10' = pendiente autorización riesgo, desde *mod. FQ365*)
- Pedidos con exceso de riesgo se muestran en estado '1K' *(mod. 08049)*
- Pedidos bloqueados por bloqueo albarán se pueden ocultar *(mod. PG615, revertido para Italia en E7394)*

**Ficheros que utiliza:**
- `COCLIN` — Maestro de clientes
- `COCAPE` / `COCAPE23` / `COCAPE25` / `COCAPE38` — Cabeceras de pedido (acceso lógico múltiple)
- `CMA040` — Datos complementarios de pedido
- `PMSETP` — Setup de parámetros
- `PMTABD` — Tablas de dominio
- `CODEPE08` — Detalle de pedido (para cálculo de importes, *mod. FQ665*)
- `MSIVAS`, `PMUNAL`, `MSDIV0` — Para cálculo de importe valorado *(mod. FQ665)*
- `ZQGSRK00`, `ZQGCSC00` — Gestión de riesgos y gestor asignado *(mods. 2310B, FH002)*

**Acciones disponibles desde la lista:**
- Acceder a detalle de pedido → lanza `CO0120`
- Otras opciones configurables por tipo de pedido

---

### 1.2 Mantenimiento de cabecera de pedido (CO0120)

**Función:** Programa principal de mantenimiento de la cabecera de un pedido de cliente. Permite crear, modificar y confirmar pedidos. Es el núcleo del módulo de compras.

**Características principales:**

#### Creación de pedido
- Validación de cliente: datos de crédito, estado, límite de riesgo
- Asignación de delegación (DLGA) y tipo de pedido (TPED)
- Captura de referencia del cliente, condiciones de pago, dirección de entrega
- Soporte a pedidos EDI internacional (PPED=99) cargados desde `COCPED` *(mod. C0008)*

#### Control de riesgo crediticio *(mods. FQ365, 08049, 15049)*
- Al confirmar un pedido, verifica si el cliente excede su límite de riesgo
- Si excede: el pedido queda en estado '1K' (pendiente autorización)
- Si se autoriza el riesgo: el pedido pasa a estado confirmado (incluso si estaba en status '10')

#### Cálculo de fecha de entrega
- Calcula el plazo de entrega a partir de parámetros DOM y el almacén asignado
- Recalcula al confirmar pedido *(mod. 30056)*
- Nuevo esquema de cálculo DPM *(mods. 25051/FQ944/FQ937)* — método del Sr. Rifà
- Para Fluidra Direct (empresa productiva): cálculo de plazo propio *(mod. 03090)*

#### Fluidra Direct *(mod. FQ284)*
- Identifica pedidos de Fluidra Direct y aplica lógica de distribución específica
- Si tipo de plazo = 0 en FD → equivale a tipo 2 *(mod. 30914)*

#### Confirmación del pedido
- Al confirmar, graba la fecha confirmada en cabecera = la más desfavorable de las líneas *(mod. 06B12)*
- Opcionalmente reserva el pedido al confirmar (por setup) *(mod. 3003A)*
- Envío de mail de confirmación al cliente *(mod. C1997, vía UT2725)*

#### Coenvíos *(mods. C0031, FQ937)*
- Soporta agrupación de líneas en coenvíos
- Opción 'RP' → ajusta fechas de todas las líneas del coenvío
- Artículos de servicio excluidos del recálculo *(mod. 26712)*

#### Auditoría *(mod. FQ799)*
- Registra cambios en pedidos en fichero de auditoría

**Ficheros que utiliza:**
- `COCLIN` — Maestro de clientes
- `COCAPE`, `CODIPE`, `COOBPE`, `COPECO`, `CODEPE`, `COGAPE` — Cabecera, detalle, observaciones y demás ficheros del pedido
- `COCAIM` — Cabecera importe
- `CODEP2` — Detalle de pedido vista 2
- `PMTABL`, `PMTABD` — Tablas de parámetros
- `MSLOAD` — Almacenes logísticos
- `MSALMA08` — Maestro de almacenes *(Fluidra Direct, mod. F-2OL)*
- `PMUNAL` — Unidades de medida *(mod. 28099)*
- `PMSETP` — Parámetros de setup

---

### 1.3 Entrada de líneas de pedido (CO0140)

**Función:** Pantalla de subfile para la captura y modificación del detalle de líneas de un pedido de cliente.

**Características principales:**

#### Captura de línea
- Artículo (código + variante + modelo), cantidad, unidad de medida
- Precio de venta, descuentos, condiciones de entrega
- Fecha de entrega estimada por línea

#### Control de precio y rendimiento
- Si precio venta ≤ precio coste → aviso al usuario *(mod. 07036)*
- Cálculo de rendimiento (margen) por línea y pedido
- Al cambiar a vista 3, recupera correctamente el precio de venta *(mod. 21126)*
- Aviso si rendimiento < mínimo configurado al confirmar *(mod. 04098)*
- Costes adicionales para el cálculo de rendimiento *(mod. 14066)*

#### Ecotasa *(mod. ECOP — OS 23/03/2007)*
- Tratamiento automático de ecotasa según tipo de artículo
- Campo especial para importe de ecotasa en la línea

#### Control de compatibilidades *(mod. C0074)*
- Validación de compatibilidades e incompatibilidades ADR/Astralpool (campo ZZADCI)
- Bloquea combinaciones de artículos incompatibles en el mismo pedido

#### Coenvíos *(mod. C0031)*
- Identificación de líneas que pertenecen a un coenvío
- Gestión de número de coenvío (NCOE)
- Para artículos conjuntos: calcula plazo para cada componente y usa la fecha más alta *(mod. 31712)*
- Artículos de servicios excluidos del cálculo de coenvíos *(mod. 26712)*
- Ajuste de frecuencia semanal para pedidos con coenvío parcial *(mod. 04712)*

#### Stock e idioma de artículos
- Muestra stock disponible *(mod. RD 27/01/2004)*
- Recupera descripción del artículo en el idioma del cliente *(mod. 24092)*

#### Opciones de línea
- Opción 'NF' — Nueva facturación *(mod. 09074)*
- Si @PERRO se recibe con '1' → confirma la orden directamente

**Ficheros que utiliza:** *(extenso — incluye ficheros de pedido, artículo, stock, precios, compatibilidades)*

---

### 1.4 Lanzamiento de órdenes de venta (CO2519)

**Función:** Genera el número de orden de venta y realiza el lanzamiento efectivo de la orden al sistema de ejecución.

**Características principales:**

#### Asignación de número de orden
Busca el número de orden de venta en la tabla NPVE mediante la siguiente jerarquía de búsqueda (de más específica a más general):
1. CIAS + DLGA + TPED
2. CIAS + DLGA
3. CIAS + TPED
4. CIAS

#### Modos de operación (parámetro @PERRO en entrada)
| Valor | Acción |
|-------|--------|
| `' '` (blancos) | Solo crear número de orden |
| `'1'` | Solo lanzar orden (número ya existe) |
| `'2'` | Crear número de orden Y lanzar |

#### Resultado (parámetro @PERRO en salida)
| Valor | Significado |
|-------|-------------|
| `' '` (blancos) | OK, operación completada |
| `'1'` | Error: no se ha podido localizar el número de orden |

#### Integridad de coenvíos *(mod. C0031)*
- Albaranes a terceros desde Trace con integridad de envíos por grupos 100% (coenvíos)

#### Nivel de servicio *(mods. 28099, 18129)*
- Tratamiento del nivel de servicio (NDS)
- Modifica el campo de plazo en base al nivel de servicio del almacén

#### Confirmación de fecha *(mods. 06B12, 25051, C2549)*
- Al confirmar, pone en cabecera la fecha más desfavorable de las líneas
- Solución de incidencias en tipos de plazo

#### Auditoria digital *(mod. PGE21 — 15/12/2025)*
- Actualiza auditoría en CODEPE al lanzar

#### Email de confirmación *(mod. C1997)*
- Envío de mail de confirmación al cliente tras el lanzamiento

**Ficheros que utiliza:**
- `PMTABL`, `PMTABD` — Tablas para numeración
- `COCAPE`, `CODIPE`, `COOBPE`, `COPECO`, `CODEPE`, `COGAPE`, `COCAIM`, `CODEP2` — Ficheros del pedido
- `COCLIN` — Maestro de clientes
- `MSLOAD` — Almacenes
- `MSALMA08` — *(Fluidra Direct)*
- `PMUNAL` — Unidades de medida
- `PMSETP` — Parámetros de setup

---

### 1.5 Validación de coenvíos (CO2696)

**Función:** Servicio de validación que comprueba si un pedido (o línea) puede expedirse según las reglas de integridad de coenvíos.

**Lógica de negocio:**
- Un coenvío es un grupo de líneas que deben expedirse conjuntamente (100% o ninguna)
- Antes de la expedición, verifica que todas las líneas del grupo estén reservadas

**Códigos de retorno (@PERRO):**

| Valor | Significado |
|-------|-------------|
| `' '` | OK — puede expedirse |
| `'0'` | OK — no es un coenvío |
| `'1'` | Error — la línea no está reservada al 100% |
| `'2'` | Error — faltan líneas por reservar |
| `'3'` | Atención — todas las líneas están pero hay alguna reserva parcial |
| `'4'` | Error — el coenvío está OK pero su anterior dependiente no |

**Parámetros de entrada:**
- `@PCIAS` — Empresa
- `@PARTI` — Artículo
- `@PVARI` — Variante
- `@PMODE` — Modelo

**Ficheros que utiliza:**
- `COCLIN` — Maestro de clientes (cabecera de pedido)
- `CODEPE` — Detalle de pedido

---

### 1.6 Configuración de impresión (CO2593)

**Función:** Panel interactivo para que el usuario configure los parámetros de impresión (formato de salida, número de copias, impresora) antes de lanzar un proceso de impresión.

**Características principales:**
- Presenta un formulario con campos: formato de exportación (GEXP), número de copias (NPIC), impresora (PEPE)
- Valida los valores introducidos contra la tabla MMPROM (diccionario Si/No)
- Verifica la existencia del dispositivo de impresora mediante MM2559
- **Soporte batch (mod. B0251):** Si el trabajo viene de SalesForce API (usuario QUSER), no muestra pantalla y usa valores por defecto

**Comportamiento según modo de ejecución:**
| Condición | Comportamiento |
|-----------|---------------|
| Trabajo interactivo | Abre y muestra la pantalla CO2593FM |
| Trabajo batch (QUSER) | No abre pantalla, usa parámetros recibidos |
| Error + batch | Sale con error sin mostrar pantalla |
| Error + interactivo | Muestra pantalla con el error |

**Parámetros de entrada:**
- `@PCIAS` — Empresa
- `@PDLGA` — Delegación
- `@PPROG` — Programa que llama
- `@PCCIT` — Título de la pantalla
- `@PGEXP` — Formato de exportación (valor inicial propuesto)
- `@PNPIC` — Número de copias (valor inicial propuesto)
- `@PPEPE` — Impresora (valor inicial propuesto)
- `@PPAR1` — Si = '1', no debe presentarse pantalla
- `@PPAR2` — Libre
- `@PJOBT` — Nombre del trabajo

**Parámetros de salida:**
- `@PGEXP` — Formato de exportación seleccionado
- `@PNPIC` — Número de copias seleccionado
- `@PPEPE` — Impresora seleccionada
- `@PERRO` — `' '` = OK; `'A'` = usuario pulsó retroceso/cancelar; `'3'` = valor inválido

**Teclas de función:**
- `F4` — Ayuda / lista de valores válidos para el campo activo
- `F12` / `F3` — Cancelar (devuelve `@PERRO='A'`)

---

## 2. Módulo de Logística — DOM

### Conceptos previos

**DOM (Distribution Order Management):** Motor externo que decide desde qué almacén y en qué orden se sirven los pedidos. Los programas DO\* mantienen las tablas de configuración que alimentan ese motor.

**EDPAR4:** Fichero maestro de prioridades de distribución. Cada registro define una regla de prioridad identificada por:
- Empresa (CIAS)
- Almacén (MAGA)
- Tipo de registro (TREG): define el nivel de la regla (empresa, delegación, proveedor, artículo, familia, cliente...)

---

### 2.1 Prioridades nivel delegación — lista (DO0001 / DO0020)

**Función:** Pantallas de subfile que listan las reglas de prioridad DOM a nivel de empresa/almacén con posibilidad de ver también las de delegación. Punto de entrada para la gestión de prioridades.

**DO0001 vs DO0020:**
- DO0001 *(creado 16/01/2014)*: incluye fase 2OL — permite abrir a más de un OL *(mod. IT244, 27/05/2022)*
- DO0020 *(creado 16/01/2014)*: variante del mismo tipo

**Pantalla principal muestra para cada regla:**
- Empresa, almacén, tipo de registro (TREG)
- Prioridades asignadas (lista ordenada de almacenes fuente)
- Estado (activo/inactivo)

**Opciones de la lista:**
- `5 = Ver detalle` → lanza DO0005 o DO0025
- `2 = Modificar` → lanza DO0005 o DO0025 en modo edición

**Jerarquía de prioridades EDPAR4:**

```
Nivel empresa:
  TREG=0  Prioridad general de empresa
  TREG=1  Excepción por proveedor
  TREG=2  Excepción por artículo
  TREG=3  Excepción por familia/subfamilia
  TREG=4  Excepción por familia/tarifa
  TREG=5  Excepción por cliente

Nivel delegación (sobreescribe empresa):
  TREG=A  Prioridad general de delegación
  TREG=B  Excepción por DLGA/proveedor
  TREG=C  Excepción por DLGA/artículo
  TREG=D  Excepción por DLGA/familia
  TREG=E  Excepción por DLGA/fam-subfamilia
  TREG=F  Excepción por DLGA/fam-tarifa
  TREG=G  Excepción por DLGA/cliente
  TREG=H  Excepción por DLGA/procedimiento de pedido (solo DO0001)
```

**Ficheros que utiliza:**
- `EDPARM`, `EDPAR2`, `EDPAR4` — Prioridades de distribución
- `PMLIBL`, `PMEMPR` — Parámetros de empresa
- `MSALMA`, `MSSGGA` — Maestros de almacén
- `PMSETP` — Setup
- `MSARTI00` — Maestro de artículos *(para excepciones por artículo)*
- `PUPROV` — Maestro de proveedores *(para excepciones por proveedor)*
- `MSFAMI` — Maestro de familias
- `PMTABD` — Tablas de dominio
- `COCLIN` — Maestro de clientes *(para excepciones por cliente)*

---

### 2.2 Prioridades nivel DLGA — panel (DO0005 / DO0025)

**Función:** Formulario de detalle para crear, modificar, copiar o eliminar una regla de prioridad DOM a nivel de delegación.

**DO0005 vs DO0025:**
- DO0005 *(creado 18/03/2010)*: versión base con excepciones a nivel de artículo *(mod. C3758, 08/11/2017)*
- DO0025 *(creado 18/03/2010)*: variante con opciones extendidas

**Operaciones disponibles (DSP/CRT/DLT/CPY/CHG):**
- `DSP` — Visualizar una regla existente
- `CRT` — Crear nueva regla
- `DLT` — Eliminar regla
- `CPY` — Copiar regla a otra empresa/almacén/delegación
- `CHG` — Modificar regla existente

**Campos editables de una regla:**
- Empresa (CIAS), Almacén (MAGA), Delegación (DLGA)
- Tipo de registro (TREG): determina el tipo de criterio
- Criterio específico según TREG: proveedor, artículo, familia, cliente, etc.
- Lista de almacenes fuente en orden de prioridad (1º, 2º, 3º...)
- Estado activo/inactivo

**Historial de cambios en EDPAR4:**
- Al modificar, guarda el histórico en `EDHPA4`

**Ficheros que utiliza:**
- `EDPAR2`, `EDPAR4`, `EDHPA4` — Prioridades de distribución y su histórico
- `PMEMPR`, `PMTAB2`, `PMTABD` — Parámetros
- `MSARTI00`, `MSFAMI`, `MSALMA`, `PUPROV`, `COCLIN` — Maestros para validación de criterios

---

### 2.3 Parámetros Fluido Direct (DO0010)

**Función:** Mantenimiento de los parámetros de configuración del sistema Fluido Direct, que gestiona la distribución directa desde empresa productiva a cliente final.

**Características principales:**
- Permite configurar por empresa/almacén las reglas de funcionamiento de Fluido Direct
- Controla qué artículos/almacenes participan en el circuito de distribución directa
- Los parámetros afectan al cálculo de plazos de entrega en CO0120 y CO2519

**Estructura de proceso (ZXOPT):**
- Usa subfiles (SN) para listado y ventanas (WN) para formularios de detalle
- Rutinas estándar: INZ (inicializar), DSP (visualizar), CHK (validar), CLR (limpiar), UPD (actualizar)

**Ficheros que utiliza:** *(específicos de Fluido Direct — parámetros de empresa y almacén)*

---

## 3. Módulo de Soporte

Los programas de este módulo son **servicios internos** invocados mediante CALL desde otros programas. No tienen interfaz de usuario directa (excepto CO2593).

---

### 3.1 Cálculo de días laborables (CO2611)

**Función:** Dado una fecha de inicio y un número de días laborables, devuelve la fecha resultante descontando sábados y domingos.

> **Nota:** Solo descuenta sábados y domingos. No tiene en cuenta festivos nacionales/locales. Está documentado en el propio código como una limitación conocida.

**Algoritmo (mod. CR191):**
1. Determina el día de la semana de la fecha origen (usando @GETWEEK, donde 1=Lunes, ..., 5=Viernes, 6=Sábado, 0=Domingo)
2. Si la fecha origen cae en sábado → ajusta al lunes siguiente (+2 días)
3. Si la fecha origen cae en domingo → ajusta al lunes siguiente (+1 día)
4. Calcula semanas completas: `N_semanas = N_dias_laborables / 5`
5. Calcula días restantes: `resto = N_dias_laborables MOD 5`
6. Convierte a días naturales: `dias_naturales = N_semanas * 7 + resto`
7. Ajuste adicional si el resto hace saltar el fin de semana (según día de inicio)
8. Comprueba que la fecha resultado no caiga en fin de semana; si es así, ajusta

**Parámetros:**

| Parámetro | Tipo | E/S | Descripción |
|-----------|------|-----|-------------|
| `@PEDAT1` | 8,0 | Entrada | Fecha de inicio (formato YYYYMMDD) |
| `@PEDAYS` | 5,0 | Entrada | Número de días laborables a sumar |
| `@PEDAT2` | 8,0 | Salida | Fecha resultado (formato YYYYMMDD) |
| `@PEERRO` | 1A | Salida | `' '` = OK; `'1'` = Error (fecha inválida) |

**Ejemplo:**
- Entrada: fecha=20260310 (martes), días=3
- Resultado: 20260313 (viernes) — sin cruzar fin de semana

- Entrada: fecha=20260312 (jueves), días=3
- Resultado: 20260317 (martes) — salta el fin de semana

---

### 3.2 Situación distribución conjuntos (CO2733)

**Función:** Dado un artículo, determina si es un **conjunto de venta** y, si lo es, cómo se distribuyen sus componentes en términos de almacenamiento en Trace (sistema de gestión de stock).

**Lógica de negocio:**
- Un "conjunto de venta" (TCOJ='4') es un artículo que se compone de varios artículos componentes
- Un artículo puede ser simultáneamente "conjunto de venta" y "conjunto de compra" (TCOJ='1') → en ese caso NO se trata como conjunto de distribución (queda excluido)
- Para cada componente del conjunto, verifica si su campo SIOL='2' (almacenado en Trace) o no

**Códigos de retorno (@PERRO):**

| Valor | Significado |
|-------|-------------|
| `'1'` | Todos los componentes se almacenan en Trace (SIOL='2') |
| `'2'` | Ningún componente se almacena en Trace |
| `'3'` | Conjunto combinado: algunos componentes en Trace y otros no |
| `'4'` | No es un conjunto de venta |

> **Nota:** El código tiene preparado (pero comentado) el caso `'E'` = empresa no permite reservar conjuntos combinados (campo P7NRCC pendiente de implementar en PMEMPR).

**Parámetros:**

| Parámetro | Tipo | E/S | Descripción |
|-----------|------|-----|-------------|
| `@PCIAS` | Num | Entrada | Empresa |
| `@PARTI` | Alfa | Entrada | Código de artículo |
| `@PVARI` | Alfa | Entrada | Variante del artículo |
| `@PMODE` | Alfa | Entrada | Modelo del artículo |
| `@PERRO` | 1A | Salida | Código de resultado (ver tabla) |

**Ficheros que utiliza:**
- `MSARTI00` — Maestro de artículos (campo SIOL)
- `MSCOJC` — Cabecera de conjuntos
- `MSCOJD` — Detalle de conjuntos (componentes)

---

### 3.3 Categoría de rotación de artículo (CO2755)

**Función:** Determina la **categoría de rotación** de un artículo (clasificación MTS/MTO y subcategorías A/B/C) en función de los parámetros de empresa y el comportamiento del artículo en stock.

> Última modificación: 26/11/2025 (marca 08835) — corrección de clasificación MTS/MTO incorrecta en Fluido Direct (SCE-104).

**Lógica de clasificación:**
- Lee los parámetros de empresa (PMEMPR/PMLIBL) para obtener las reglas de clasificación
- Consulta el maestro de artículos (MSARTI00) y datos de stock (MSSTOK, MSSTST)
- Determina si el artículo es **MTS** (Make-To-Stock, fabricación para stock) o **MTO** (Make-To-Order, fabricación bajo pedido)
- Calcula tres categorías (A, B, C) según umbrales de rotación configurados en empresa
- La clasificación puede variar por almacén lógico (LOGE)

**Parámetros de entrada:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `@PCIAS` | Num | Empresa |
| `@PDLGA` | Alfa | Delegación |
| `@PLOGE` | Alfa | Almacén lógico |
| `@PARTI` | Alfa | Código de artículo |
| `@PVARI` | Alfa | Variante |
| `@PMODE` | Alfa | Modelo |
| `@PTCOJ` | 1A | Tipo de conjunto |
| `@PLOFD` | 1A | Indicador Fluido Direct (opcional, parámetro 12) |

**Parámetros de salida:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `@PCROT` | 2A | Categoría de rotación resultante |
| `@PCROTA` | 2A | Categoría rotación A (umbral bajo) |
| `@PCROTB` | 2A | Categoría rotación B (umbral medio) |
| `@PCROTC` | 2A | Categoría rotación C (umbral alto) |

**Estructura de parámetros de empresa (PPAMPL):**

| Campo | Pos | Descripción |
|-------|-----|-------------|
| PPCROT | 1 | Indicador uso rotación |
| PPRISK | 2 | Nivel de riesgo |
| PPDICO | 3 | Diccionario |
| PPCONI | 4 | Condición |
| PPOBJT | 5 | Objetivo |
| PPCROP | 6 | Categoría propuesta |
| PPBBAC | 7-16 | Umbral bajo categoría A |
| PPBBAP | 17-26 | Umbral alto categoría A |
| PPEFIN | 27 | Indicador fin |
| PPEQAR | 28 | Igualar artículo |
| PPDWVT, PPDWCT, PPDWPU, PPDWST, PPDWCV, PPDWFB | 29-34 | Flags de control adicionales |

**Ficheros que utiliza:**
- `PMLIBL` — Parámetros de biblioteca
- `PMEMPR` — Parámetros de empresa
- `MSLOAD` — Almacenes logísticos
- `MSSTOK` — Stock por almacén
- `MSSTST` — Stock estadístico *(mod. C1957)*
- `MSCOJD` — Componentes de conjuntos
- `MSARTI00` — Maestro de artículos
- `MSALMA05` — Almacenes *(mod. CR964)*

---

### 3.4 Pedidos pendientes de servir — KORE (CO2770/CO2610)

**Función:** Dado un artículo y un rango de fechas, calcula la cantidad total pendiente de servir en pedidos de clientes activos.

> El fichero CO2770.MBR contiene la referencia al programa `CO2610` (nombre objeto en la librería). El código fuente del programa es CO2610.

**Lógica de negocio:**
1. Busca en el detalle de pedidos (CODEPE42) todos los pedidos del artículo indicado
2. Filtra por rango de fechas (de @PDATD a @PDATH)
3. Para cada línea de pedido, verifica que la cabecera del pedido esté activa (COCAPEA8)
4. Calcula la cantidad pendiente = cantidad pedida - cantidad servida (CANT - CANS)
5. Si la unidad de medida del pedido difiere de la unidad base del artículo, convierte mediante el factor de conversión de PMUNAL
6. Acumula en @PCANT la cantidad total pendiente

**Gestión de cantidades canceladas:**
- Si existe cantidad cancelada (S5CANC > 0) → usa CANC como cantidad de referencia en lugar de CANT

**Filtro de pedidos modificados (S5TPE4):**
- Si `@POMFD='1'` → excluye líneas de tipo S5TPE4='2' (pedidos modificados/duplicados)

**Parámetros de entrada:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `@PCIAS` | Num | Empresa |
| `@PDLGA` | Alfa | Delegación |
| `@PARTI` | Alfa | Código de artículo |
| `@PVARI` | Alfa | Variante |
| `@PMODE` | Alfa | Modelo |
| `@PTCOJ` | Alfa | Tipo de conjunto |
| `@PDATD` | 8,0 | Fecha desde (YYYYMMDD) |
| `@PDATH` | 8,0 | Fecha hasta (YYYYMMDD) |
| `@PCAN1` | Num | Cantidad inicial (acumulador) |
| `@PLOGE` | Alfa | Almacén lógico |
| `@POMFD` | 1A | `'1'` = excluir pedidos modificados |

**Parámetros de salida:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `@PCANT` | Num | Cantidad total pendiente de servir (en unidad base del artículo) |
| `@PERRO` | 1A | Código de error (reservado para uso futuro) |

**Conversión de unidades:**
- Usa PMUNAL (tabla de unidades de medida) para convertir de la unidad del pedido a la unidad base
- Factor de conversión (P6COFA): si P6TCOF<>'1' multiplica; si P6TCOF='1' divide
- Si no encuentra la combinación arti+vari+mode → intenta con blancos en vari y mode

**Ficheros que utiliza:**
- `CODEPE42` — Detalle de pedidos de venta (lógico con clave artículo)
- `COCAPEA8` — Cabecera de pedidos activos
- `MSLOAD` — Almacenes logísticos
- `MSARTI00` — Maestro de artículos (para unidad base)
- `PMUNAL` — Unidades de medida y factores de conversión

---

## 4. Conceptos Clave y Glosario

| Término | Descripción |
|---------|-------------|
| **CIAS** | Código numérico de empresa (3 dígitos) |
| **DLGA** | Delegación (5 caracteres alfanuméricos) |
| **LOGE** | Almacén lógico |
| **MAGA** | Código de almacén (físico) |
| **TPED** | Tipo de pedido (determina comportamiento: normal, EDI, SC, etc.) |
| **NPCO** | Número de pedido de cliente |
| **NOPL** | Número de línea del pedido |
| **NORT** | Número de orden de venta/tracing |
| **MTS** | Make-To-Stock: artículo que se fabrica para stock y se sirve desde almacén |
| **MTO** | Make-To-Order: artículo que se fabrica contra pedido |
| **Coenvío** | Grupo de líneas de pedido vinculadas que deben expedirse juntas al 100% |
| **DOM** | Distribution Order Management: motor de gestión de órdenes de distribución |
| **Trace** | Sistema de gestión de stock y trazabilidad (campo SIOL='2' indica almacenamiento en Trace) |
| **Fluido Direct** | Canal de distribución directa desde empresa productiva a cliente final |
| **SFL** | Subfile: pantalla de lista en IBM i |
| **ZXOPT** | Variable de control de flujo del programa (6 caracteres: ZXOPT1=proceso, ZXOPT2=rutina) |
| **@PE** | Prefijo de parámetros de exportación/importación de programas |
| **CS_** | Prefijo de Common Subroutines (subrutinas comunes al final del programa) |
| **LDA** | Local Data Area: área de datos de trabajo del sistema IBM i |
| **PURLDADS** | Estructura del LDA de compras (datos de empresa/delegación del contexto de trabajo) |
| **EDI** | Electronic Data Interchange: intercambio electrónico de pedidos |
| **QUSER** | Usuario especial IBM i para trabajos batch (usado para detectar ejecución desde API) |
| **DPM** | Método de cálculo de plazos de entrega por el Sr. Rifà |

---

*Documentación generada a partir del código fuente. Versión 1.0 — Marzo 2026.*
