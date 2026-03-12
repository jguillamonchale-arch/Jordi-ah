# Manual Funcional del Sistema AMS — Fluidra S.A.
## Descripción Funcional por Módulos de Negocio

**Versión:** 2.0 — Marzo 2026
**Audiencia:** Equipo funcional, equipo técnico AMS, gestores de operaciones
**Clasificación:** Interno

---

## Índice

1. [Módulo de Pedidos de Venta (CO/CM)](#1-módulo-de-pedidos-de-venta-cocm)
2. [Módulo de Compras (PU)](#2-módulo-de-compras-pu)
3. [Motor DOM y Distribución Directa (ED/DO)](#3-motor-dom-y-distribución-directa-eddo)
4. [Maestros y Parametrización (MS/PM/MM)](#4-maestros-y-parametrización-mspmmm)
5. [EDI e Intercambio Electrónico (TR/WR/COCEDI/PUCEDI)](#5-edi-e-intercambio-electrónico-trwrcocedipucedi)
6. [Configuración y Módulos Auxiliares](#6-configuración-y-módulos-auxiliares)
7. [Glosario y Conceptos Clave](#7-glosario-y-conceptos-clave)

---

## 1. Módulo de Pedidos de Venta (CO/CM)

### 1.1 Visión general del módulo

El módulo de Pedidos de Venta gestiona el ciclo completo de un pedido de cliente desde su creación hasta su lanzamiento al sistema de ejecución. Es el punto de entrada principal de la demanda en el sistema AMS.

**Características del módulo:**
- Multiempresa y multidelegación (CIAS + DLGA)
- Control de riesgo crediticio integrado
- Soporte a pedidos EDI (importados electrónicamente desde COCPED)
- Gestión de coenvíos (grupos de líneas que se expiden juntas)
- Integración directa con el Motor DOM para asignación automática de almacén
- Soporte a artículos conjuntos (kits) con cálculo de fechas por componente
- Clasificación automática de artículos MTS/MTO

**Prefijos de programas:** `CO`, `CM`
**Número de programas:** ~106

### 1.2 Flujo funcional del pedido de venta

```
┌──────────────────────────────────────────────────────────────────┐
│                    CICLO DEL PEDIDO DE VENTA                     │
│                                                                  │
│  1. CONSULTA PEDIDOS (CM0085)                                    │
│     └─ Lista paginada de pedidos activos                         │
│        Filtros: cliente, delegación, estado, gestor de riesgos   │
│                                │                                 │
│                                ▼                                 │
│  2. CABECERA DE PEDIDO (CO0120)                                   │
│     └─ Crear / modificar cabecera:                               │
│        Cliente, delegación, tipo pedido, fecha, referencia       │
│        Condiciones de pago, dirección entrega                    │
│        Control de riesgo crediticio                              │
│        Cálculo de plazo de entrega (método DPM / días laborables)│
│                                │                                 │
│                                ▼                                 │
│  3. LÍNEAS DE PEDIDO (CO0140)                                     │
│     └─ Artículo + variante + modelo                              │
│        Cantidad, unidad de medida, precio, descuentos            │
│        Ecotasa, coenvíos, compatibilidades ADR                   │
│        Fecha entrega por línea, stock disponible                 │
│                                │                                 │
│                                ▼                                 │
│  4. LANZAMIENTO ORDEN (CO2519)                                    │
│     └─ Asigna número de orden de venta (NPVE)                    │
│        Valida coenvíos, calcula fecha más desfavorable           │
│        Envía email de confirmación al cliente                    │
│                                │                                 │
│                                ▼                                 │
│  5. MOTOR DOM (ED3025 / DOOM)                                     │
│     └─ Asignación automática de almacén/proveedor                │
│        (ver capítulo 3 para detalle profundo)                    │
│                                │                                 │
│                                ▼                                 │
│  6. VALIDACIÓN COENVÍO (CO2696) — al expedir                     │
│     └─ Verifica que todas las líneas del grupo están al 100%    │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 Consulta de pedidos — CM0085

**Función:** Pantalla de lista (subfile) de pedidos activos. Punto de entrada del módulo.

**Vistas disponibles:**
- **Vista 1:** Cliente, fecha pedido, tipo, número, referencia, moneda, estado
- **Vista 2:** Descripción del cliente, importe total, condiciones de riesgo (RCON)
- **Vista 3** *(mod. 09022):* Ordenación descendente por número de pedido

**Filtros de búsqueda:**
- Código de cliente
- Delegación (DLGA)
- Gestor de riesgos asignado *(mod. FH002)*
- Estado del pedido (incluye estado '10' = pendiente autorización riesgo)

**Casos especiales:**
- Pedidos con exceso de riesgo → se muestran en estado `'1K'`
- Pedidos con bloqueo de albarán → pueden ocultarse de la lista *(mod. PG615, revertido para Italia en E7394)*

**Ficheros principales:** COCLIN, COCAPE (+ lógicos 23/25/38/A8), CMA040, PMSETP, ZQGSRK00, ZQGCSC00

### 1.4 Cabecera de pedido — CO0120

**Función:** Programa principal del módulo. Crea y gestiona la cabecera del pedido. Es el orquestador del ciclo de vida completo.

#### Creación de pedido
- Validación de cliente (crédito, estado, límite de riesgo)
- Asignación de delegación (DLGA) y tipo de pedido (TPED)
- Captura: referencia del cliente, condiciones de pago, dirección de entrega
- Pedidos EDI (PPED=99): cargados desde tabla `COCPED` *(mod. C0008)*

#### Control de riesgo crediticio
| Estado | Significado |
|--------|-------------|
| Normal | El cliente está dentro de su límite de riesgo |
| `'10'` | Pedido pendiente de autorización (riesgo excedido) |
| `'1K'` | Pedido bloqueado por exceso de riesgo *(mod. 08049)* |

El pedido no puede confirmarse si está en estado '1K'. La autorización lo desbloquea (incluso si estaba en '10').

#### Cálculo de fecha de entrega
- **Método DPM** *(mods. 25051/FQ944/FQ937, Sr. Rifà):* cálculo avanzado de plazo
- **CO2611:** suma de días laborables (excluye fines de semana)
- **Fluidra Direct:** cálculo de plazo propio *(mod. 03090)*
- La fecha confirmada en cabecera = la más desfavorable de las líneas *(mod. 06B12)*

#### Coenvíos
- Soporte a agrupación de líneas en grupos de expedición conjunta
- Opción `'RP'` → ajusta fechas de todas las líneas del coenvío
- Artículos de servicio excluidos del recálculo *(mod. 26712)*

**Servicios internos invocados:**
- `CO0140` — detalle de líneas
- `CO2519` — lanzamiento de orden
- `CO2593` — parámetros de impresión
- `CO2696` — validación de coenvío
- `CO2755` — categoría de rotación del artículo
- `CO2611` — suma días laborables
- `UT2725` — envío email de confirmación *(mod. C1997)*

### 1.5 Líneas de pedido — CO0140

**Función:** Captura y modificación del detalle de líneas (artículos) del pedido.

#### Captura de línea
- Artículo (código + variante + modelo), cantidad, unidad de medida
- Precio de venta, descuentos, condiciones de entrega
- Fecha de entrega estimada por línea

#### Reglas de negocio de línea

| Regla | Descripción |
|-------|-------------|
| **Control de precio** | Si precio venta ≤ precio coste → aviso al usuario *(mod. 07036)* |
| **Rendimiento mínimo** | Aviso si margen < mínimo configurado al confirmar *(mod. 04098)* |
| **Ecotasa** | Tratamiento automático según tipo de artículo *(mod. ECOP)* |
| **Compatibilidades ADR** | Bloquea combinaciones de artículos incompatibles *(mod. C0074)* |
| **Stock visible** | Muestra stock disponible en pantalla *(mod. RD 2004)* |
| **Idioma artículo** | Recupera descripción en el idioma del cliente *(mod. 24092)* |
| **Coenvío** | Identifica grupo de expedición conjunta (número NCOE) |
| **Conjuntos (kits)** | Para artículos conjuntos: calcula fecha por componente (fecha más alta) *(mod. 31712)* |
| **Servicios** | Artículos de servicio excluidos del cálculo de coenvíos *(mod. 26712)* |

### 1.6 Lanzamiento de orden — CO2519

**Función:** Genera el número de orden de venta y realiza el lanzamiento efectivo.

**Modos de operación (parámetro @PERRO entrada):**

| Valor | Operación |
|-------|-----------|
| `' '` | Solo crear número de orden |
| `'1'` | Solo lanzar (número ya existe) |
| `'2'` | Crear Y lanzar |

**Proceso de numeración:**
Búsqueda en cascada en `PMTABD` (tabla PECLTA), de más específica a más general:
1. CIAS + DLGA + TPED
2. CIAS + DLGA
3. CIAS + TPED
4. CIAS

**Validaciones antes del lanzamiento:**
- Integridad de coenvíos (CALL CO2696) — todos al 100% o ninguno
- Nivel de servicio del almacén (NDS) *(mods. 28099, 18129)*
- Fecha confirmada = fecha más desfavorable de las líneas *(mod. 06B12)*

**Acciones tras el lanzamiento:**
- Envío email de confirmación al cliente (CALL UT2725) *(mod. C1997)*
- Actualización de auditoría digital en CODEPE *(mod. PGE21, dic-2025)*

### 1.7 Validación de coenvíos — CO2696

**Función:** Servicio (sin pantalla) que verifica si un pedido puede expedirse según las reglas de coenvío.

**Regla de negocio:** Un coenvío = grupo de líneas que se expiden juntas al 100% o no se expiden.

| Código retorno | Significado |
|----------------|-------------|
| `' '` | OK — puede expedirse |
| `'0'` | OK — no es un coenvío |
| `'1'` | Error — línea no reservada al 100% |
| `'2'` | Error — faltan líneas por reservar |
| `'3'` | Atención — todas presentes pero alguna con reserva parcial |
| `'4'` | Error — dependiente anterior no está OK |

### 1.8 Servicios de soporte del módulo

#### CO2611 — Cálculo días laborables
Dado una fecha de inicio y N días laborables → devuelve la fecha resultante excluyendo sábados y domingos. **Nota:** no considera festivos nacionales.

#### CO2755 — Categoría de rotación
Determina si un artículo es **MTS** (Make-To-Stock) o **MTO** (Make-To-Order) y su subcategoría (A/B/C) según parámetros de empresa. Usado en CO0120, CO2519 y otros módulos.

#### CO2733 — Situación distribución conjuntos
Para artículos conjuntos (kits): determina si todos sus componentes son Trace, ninguno, o mezcla.

| Retorno | Significado |
|---------|-------------|
| `'1'` | Todos los componentes en Trace (MTS puro) |
| `'2'` | Ningún componente en Trace (MTO puro) |
| `'3'` | Conjunto combinado (mezcla Trace/no-Trace) |
| `'4'` | No es un conjunto de venta |

#### CO2610 / CO2770 — Pedidos pendientes (KORE)
Para un artículo y rango de fechas: calcula la cantidad total pendiente de servir en pedidos activos. Invocado por el motor externo KORE para cálculo de disponibilidad. La cantidad se devuelve en unidad base del artículo (con conversión automática si el pedido usa unidad diferente).

### 1.9 Configuración de impresión — CO2593

**Función:** Panel para configurar parámetros de impresión (formato, copias, impresora). Soporta modo batch (sin pantalla) para llamadas desde SalesForce API — detecta el tipo de trabajo con RTVTIJ.

---

## 2. Módulo de Compras (PU)

### 2.1 Visión general

El módulo de Compras gestiona el ciclo de vida de las **Órdenes de Compra (OC)** a proveedores. Trabaja en paralelo y coordinación con el módulo de Ventas: una OC de compra puede generarse automáticamente por el Motor DOM como consecuencia de un pedido de venta, o puede crearse manualmente.

**Prefijo:** `PU`
**Número de programas:** ~37

**Ficheros principales:**
- `PUCAPE` — Cabecera de Orden de Compra
- `PUDEPE` — Líneas de OC
- `PUPROV` — Maestro de proveedores
- `PUCEDI` — Configuración EDI por proveedor
- `PUHALD` — Histórico de albaranes de compra

### 2.2 Flujo funcional de una Orden de Compra

```
[Origen: Manual o automático por DOM]
              │
              ▼
┌─────────────────────────────────────────┐
│  1. CONSULTA OC (PU2519)                │
│     └─ Lista de OC activas              │
│        Filtros: proveedor, estado, fecha│
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  2. CABECERA OC (PU2600)                │
│     └─ Proveedor, delegación, fecha     │
│        Condiciones entrega, divisa      │
│        Tipo transporte, descuentos      │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  3. LÍNEAS OC (PUDEPE / proceso)        │
│     └─ Artículo, cantidad, precio       │
│        Fecha solicitud, fecha recepción │
│        Referencia al pedido de venta    │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  4. LANZAMIENTO / CONFIRMACIÓN          │
│     └─ PUPRPR (proceso de OC)           │
│        Genera EDI ORDERS si proveedor   │
│        está configurado en PUCEDI       │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  5. RECEPCIÓN / ALBARÁN                 │
│     └─ PUHALD (registro histórico)      │
│        PUDTO0/1 (datos de tránsito)     │
│        Actualiza stock (MSSTOK)         │
└─────────────────────────────────────────┘
```

### 2.3 Sincronización Compras ↔ Ventas (ED0015)

El programa `ED0015` gestiona la **sincronización de eventos entre el pedido de venta y la orden de compra** correspondiente. Cuando cambia el estado de la OC (confirmación, fecha, cantidad, cancelación), `ED0015` propaga los cambios al pedido de venta asociado.

**Eventos gestionados:**
- Evento 013: Confirmación de OC por el proveedor
- Evento 022: Cambio de cantidad en OC
- Evento 028: Cancelación de línea de OC

**Reglas de propagación:**
- Las fechas de entrega de la OC se reflejan en el pedido de venta
- Los cambios de cantidad solo se permiten en sentido positivo si el tipo es "Alta"
- Cambios negativos (reducción) siempre permitidos
- Detección de OC duplicadas (evento 15041)

### 2.4 Configuración EDI de proveedores — PUCEDI

Cada proveedor puede tener habilitado/deshabilitado el intercambio EDI. La tabla `PUCEDI` define:

| Campo | Descripción |
|-------|-------------|
| C9COMP | Rol comprador (UN-EDIFACT) |
| C9RECE | Rol receptor |
| C9CFAC | Rol facturado |
| C9PAGA | Rol pagador |
| C9EPED | Enviar OC (ORDERS) por EDI |
| C9RRSP | Recibir respuesta OC (ORDRSP) |
| C9RINV | Recibir informe inventario (INVRPT) |
| C9RALB | Enviar albarán (DESADV) |
| C9RFAC | Enviar factura (INVOIC) |

### 2.5 Programas principales del módulo PU

| Programa | Tipo | Función |
|----------|------|---------|
| `PU2519` | SFL interactivo | Lista de órdenes de compra |
| `PU2520` | Panel | Detalle de OC |
| `PU2527` | Panel | Consulta ampliada OC |
| `PU2560` | Panel | Mantenimiento OC |
| `PU2600` | SFL + Panel | Gestión completa de cabecera OC |
| `PU2601` | Panel | Detalle extendido de OC |
| `PUPRPR` | Proceso | Proceso de OC (cálculos y actualizaciones) |
| `PUPRPR03` | Proceso | Variante del proceso de OC |
| `PUPRMD` | SFL | Mantenimiento de parámetros de OC |
| `PUDTO0/1` | DDS | Datos de tránsito y recepción |
| `PUFACC` | Proceso | Facturación de compras |
| `PUHALD` | Proceso | Histórico de albaranes |
| `PUCEDI` | DDS/PF | Configuración EDI por proveedor |
| `PUPROV` | DDS/PF | Maestro de proveedores |
| `PUOBPE` | DDS/PF | Observaciones de OC |
| `PUDIVI` | Panel | Gestión de divisas en OC |
| `PUALBD` | Proceso | Albaranes de compra |
| `PUCAIM` | DDS/PF | Cabecera de importes OC |

### 2.6 Parámetros de compra — PUPRP0 / PUPRP004

La tabla `PUPRP0` almacena los parámetros de configuración del proceso de compra por empresa/delegación/almacén. Incluye:
- Horizonte de planificación (días)
- Días mínimos de entrega por proveedor
- Tratamiento de facturas y albaranes
- Restricciones de cantidad y precio

---

## 3. Motor DOM y Distribución Directa (ED/DO)

> **Para una descripción profunda del algoritmo DOOM y todas sus reglas de negocio, consultar el Documento 3: Motor DOM en Profundidad.**

### 3.1 Visión general

El Motor DOM (Distribution Order Management) es el **sistema de toma de decisiones logísticas** del AMS. Cuando se lanza un pedido de venta, el DOM determina automáticamente desde qué almacén o proveedor se servirá cada línea, optimizando en función de stock, capacidad, acuerdos logísticos, prioridades y reglas de negocio.

El Motor DOM conecta los tres pilares del sistema:
- **Ventas:** pedido de cliente (COCAPE/CODEPE) → origen de la demanda
- **Stock:** disponibilidad por almacén (MSSTOK) → fuente de suministro evaluada
- **Compras:** orden de compra (PUCAPE/PUDEPE) → resultado si no hay stock local

**Prefijos:** `ED`, `DO`, `DOM`
**Número de programas:** ~61

### 3.2 Dos sub-módulos del DOM

#### A) Configuración de prioridades DOM (programas DO*)

Los programas `DO*` permiten a los usuarios configurar las **reglas de prioridad** que el algoritmo DOOM usa para decidir el almacén óptimo.

| Programa | Función |
|----------|---------|
| `DO0001` | Lista de prioridades nivel delegación (variante 1) |
| `DO0005` | Panel de detalle/mantenimiento prioridad (variante 1) |
| `DO0010` | Parámetros de Fluidra Direct |
| `DO0020` | Lista de prioridades nivel delegación (variante 2) |
| `DO0025` | Panel de detalle/mantenimiento prioridad (variante 2) |
| `DO0030` | Mantenimiento prioridades nivel empresa |
| `DO0035` | Panel de prioridad nivel empresa |
| `DO0042/43` | Variantes de mantenimiento de prioridades |

**Jerarquía de tipos de regla (TREG):**

| TREG | Nivel | Criterio |
|------|-------|----------|
| `0` | Empresa | Prioridad general de empresa |
| `1` | Empresa | Excepción por proveedor |
| `2` | Empresa | Excepción por artículo |
| `3` | Empresa | Excepción por familia/subfamilia |
| `4` | Empresa | Excepción por familia/tarifa |
| `5` | Empresa | Excepción por cliente |
| `A` | Delegación | Prioridad general de delegación |
| `B` | Delegación | Excepción DLGA/proveedor |
| `C` | Delegación | Excepción DLGA/artículo |
| `D` | Delegación | Excepción DLGA/familia |
| `E` | Delegación | Excepción DLGA/fam-subfamilia |
| `F` | Delegación | Excepción DLGA/fam-tarifa |
| `G` | Delegación | Excepción DLGA/cliente |
| `H` | Delegación | Excepción DLGA/procedimiento pedido |

#### B) Motor DOOM — Proceso de decisión (programas ED*)

Los programas `ED*` implementan el algoritmo y el flujo de ejecución del DOM.

| Programa | Función |
|----------|---------|
| `ED3025CL` | CL wrapper — punto de entrada batch del DOM |
| `ED3025` | Orquestador del proceso DOM |
| `ED3050` | Algoritmo DOOM (selección de almacén óptimo) |
| `ED0060` | Generación de mensajes EDI tras confirmación DOM |
| `ED0100` | Auditoría y trazabilidad del proceso DOM |
| `ED1000CL` | CL wrapper — simulación entregas directas |
| `ED1000` | Simulación y confirmación de entregas directas |
| `ED3061` | Pantalla de confirmación del resultado DOM |
| `ED3062` | Pantalla de detalle DOM por línea |
| `ED3063` | Pantalla de excepciones DOM |
| `DOOM01` | Algoritmo central de decisión DOOM (núcleo) |

### 3.3 Tablas maestras del DOM

Las tablas maestras del DOM se describen en detalle en el Documento 3. En resumen:

| Tabla | Función |
|-------|---------|
| `EDPAR4` | Prioridades de distribución por empresa/almacén/tipo |
| `EDPAR2` | Parámetros secundarios por almacén (max líneas, horizonte) |
| `EDPAR3` | Restricciones de capacidad por almacén (peso, volumen, importe) |
| `EDPARC` | Reglas específicas por cliente |
| `EDPARM` | Parámetros DOM por empresa/delegación (valores por defecto) |
| `MSDDOM` | Matriz de delegaciones origen-destino (físico) |
| `MSDDOM11` | Lógico: prioridad Overstock (OB) |
| `MSDDOM12` | Lógico: prioridad Obsolete (OC) |
| `MSDDOM13` | Lógico: prioridad Super-Stock (SSTO) |
| `MSDDOM14` | Lógico: prioridad Artículo B (ArtB) |
| `DOOMHT` | Histórico de decisiones DOOM (auditoría) |

### 3.4 Flujo de activación del DOM

```
1. Usuario confirma pedido de venta en CO0120
2. CO2519 lanza la orden
3. ED3025CL se activa (batch CL wrapper)
4. ED3025 orquesta el proceso:
   a. Copia ficheros a QTEMP (aislamiento)
   b. Llama ED3050 / DOOM (si no es pedido manual o pre-dividido)
   c. DOOM evalúa cada almacén para cada línea
   d. DOOM actualiza CODEPE.S5LOGE con el almacén óptimo
   e. ED3025 presenta ED3061 (pantalla de confirmación)
   f. Usuario confirma (o ajusta manualmente)
   g. ED3025 llama ED0060 (genera EDI)
   h. ED3025 llama ED0100 (registra auditoría)
5. Limpieza de QTEMP
```

**Casos que omiten el algoritmo DOOM:**
- Pedidos de mostrador (`S0MOST='1'`)
- Pedidos pre-divididos (`S0SPLF flag`)
- Artículos de servicio (bypass automático)
- Proveedores no habilitados para ED (sin acuerdo de Entrega Directa)

### 3.5 Trazabilidad del proceso DOM

El sistema mantiene tres niveles de trazabilidad:

| Fichero | Granularidad | Contenido |
|---------|-------------|-----------|
| `DOOMHT` | Por proceso/decisión | Resumen de cada ejecución del DOOM |
| `EDTRAU` | Por evento | Log detallado de eventos del proceso EDI |
| `EDTRLP` | Por línea de pedido | Estado completo de cada línea: cantidades, fechas, flags, referencias cruzadas |

---

## 4. Maestros y Parametrización (MS/PM/MM)

### 4.1 Visión general

El módulo de Maestros contiene los **datos de referencia** que todos los demás módulos consumen: artículos, stocks, almacenes, unidades de medida, y la configuración central del sistema.

**Prefijos:** `MS` (Master Services), `PM` (Parameters Management), `MM` (Material Management)
**Número de programas:** ~93

### 4.2 Maestro de artículos — MSARTI00

**Tabla central del catálogo de productos de Fluidra.**

Campos clave:

| Campo | Descripción |
|-------|-------------|
| `M5CIAS` | Empresa |
| `M5ARTI` | Código de artículo |
| `M5VARI` | Variante |
| `M5MODE` | Modelo |
| `M5UNIM` | Unidad de medida base |
| `M5SIOL` | Indicador Trace (`'2'` = almacenado en Trace) |
| `M5TCOJ` | Tipo de conjunto (`'4'`=venta, `'1'`=compra) |
| `M5CROP` | Categoría de rotación propuesta |
| `M5ECOP` | Flag ecotasa |
| `M5OMP` | Order Management Parameter |
| `M5FAMI` | Familia del artículo |

Ficheros de acceso (lógicos): `MSARTI03` (por familia), `MSARTI07` (por almacén), `MSARTI10` (por código alternativo).

### 4.3 Stock por almacén — MSSTOK

**Disponibilidad en tiempo real de cada artículo en cada almacén físico.**

Campos clave:

| Campo | Descripción |
|-------|-------------|
| `M1CIAS` | Empresa |
| `M1MAGA` | Almacén físico |
| `M1ARTI` / `M1VARI` / `M1MODE` | Identificación artículo |
| `M1DISP` | Cantidad disponible |
| `M1RESD` | Cantidad reservada |
| `M1PEND` | Cantidad pendiente de recibir |

`MSSTST` complementa con stock estadístico histórico (usado en CO2755 para clasificación MTS/MTO).

### 4.4 Almacenes — MSALMA y MSLOAD

**MSALMA** = Maestro de almacenes **físicos** (edificio, ubicación real):

| Campo | Descripción |
|-------|-------------|
| `MHCIAS` | Empresa |
| `MHMAGA` | Código almacén físico |
| `MHNOMB` | Nombre |
| `MHUBIC` | Ubicación/dirección |
| `MHTIPO` | Tipo de almacén |

**MSLOAD** = Almacenes **lógicos** (perspectiva de negocio — empresa + delegación + almacén lógico LOGE):

| Campo | Descripción |
|-------|-------------|
| `MICIAS` | Empresa |
| `MIDLGA` | Delegación |
| `MILGCA` | Código de almacén lógico |
| `MIMAGA` | Almacén físico asociado |
| `MINDS` | Nivel de servicio del almacén |

La relación: un almacén físico (`MHMAGA`) puede tener múltiples vistas lógicas (`MILGCA`) en diferentes contextos empresa/delegación.

### 4.5 Conjuntos y kits — MSCOJC / MSCOJD

| Fichero | Contenido |
|---------|-----------|
| `MSCOJC` | Cabecera del conjunto (artículo padre, tipo de conjunto) |
| `MSCOJD` | Componentes del conjunto (artículos hijos, cantidades) |

Tipos de conjunto:
- `TCOJ='4'` → Conjunto de **venta** (explota en líneas de pedido de venta)
- `TCOJ='1'` → Conjunto de **compra** (explota en líneas de OC)

Un artículo puede ser simultáneamente conjunto de venta y de compra → en ese caso el Motor DOM lo trata de forma especial.

### 4.6 Parámetros de empresa — PMEMPR

**El hub central de configuración del sistema AMS.** Cada empresa tiene su propio registro en `PMEMPR` con decenas de campos de configuración que controlan el comportamiento de todos los módulos.

Estructura de campos extendidos (`PPAMPL`, 500 bytes):

| Posición | Campo | Descripción |
|----------|-------|-------------|
| 1 | `PPCROT` | Indicador de uso de categorías de rotación |
| 2 | `PPRISK` | Nivel de riesgo |
| 3 | `PPDICO` | Diccionario activo |
| 6 | `PPCROP` | Categoría de rotación propuesta |
| 7-26 | `PPBBAC/PPBBAP` | Umbrales de categoría A |
| 29-34 | `PPDWVT`-`PPDWFB` | Flags de control varios |

### 4.7 Tablas de parámetros — PMTABL / PMTABD

Sistema de tablas configurables de dominio:

| Tabla | Función |
|-------|---------|
| `PMTABL` | Maestro de tablas (cabecera de cada tabla) |
| `PMTABD` | Detalle de tablas (valores válidos de cada tabla) |
| `PMTAB2` | Tablas secundarias/auxiliares |

Usos principales:
- Tabla `PECLTA`: numeración de órdenes de venta
- Tabla `NPVE`: secuencia de números de OC
- Dominio de tipos de pedido (TPED)
- Dominio de estados de pedido
- Diccionarios de valores Sí/No

### 4.8 Unidades de medida — PMUNAL

Tabla de conversión de unidades de medida:

| Campo | Descripción |
|-------|-------------|
| `P6CIAS` | Empresa |
| `P6ARTI/VARI/MODE` | Artículo |
| `P6UNIM` | Unidad origen |
| `P6COFA` | Factor de conversión |
| `P6TCOF` | Tipo de factor (`'1'` = divisor, otro = multiplicador) |

Usado en CO2610 (conversión de cantidades al servir) y en todo cálculo de importes que involucra unidades diferentes.

### 4.9 Programas de gestión de maestros

#### Artículos (MSARTI)
| Programa | Función |
|----------|---------|
| `MSARTI` | Mantenimiento del maestro de artículos |
| `MSARTI00` | Fichero físico principal |
| `MSARID` | Identificación de artículos |
| `MSARAL` | Artículos alternativos/referencias cruzadas |

#### Almacenes y Stock (MSALMA/MSSTOK)
| Programa | Función |
|----------|---------|
| `MSALMA` | Mantenimiento de almacenes físicos |
| `MSLOAD` | Mantenimiento de almacenes lógicos |
| `MSSTOK` | Fichero físico de stock |
| `MSSTO0/01` | Mantenimiento de stock |

#### Delegaciones y Zonas (MSDIVI/MSSZON)
| Programa | Función |
|----------|---------|
| `MSDIVI` | Mantenimiento de divisiones/delegaciones |
| `MSDIV0/2` | Variantes de divisiones |
| `MSSZON` | Mantenimiento de zonas geográficas |
| `MSPAIS` | Maestro de países |
| `MSSOCI` | Maestro de socios/entidades |

#### Parámetros (PM*)
| Programa | Función |
|----------|---------|
| `PMEMPR` | Mantenimiento de parámetros de empresa |
| `PMTABL/PMTABD` | Mantenimiento de tablas de dominio |
| `PMUNAL` | Mantenimiento de unidades de medida |
| `PMJOBC` | Control de jobs/procesos batch |
| `PMLIBL` | Gestión de listas de librerías |
| `PMSETP` | Setup general del sistema |
| `PMCAOR/PMDEOR` | Órdenes de configuración |

#### Material Management (MM*)
| Programa | Función |
|----------|---------|
| `MM0006` | Mantenimiento de almacenes |
| `MM1410` | Selección logística de almacén |
| `MM1411/1413/1414` | Procesos de gestión de materiales |
| `MM2602/2604/2686` | Procesos de gestión avanzada de materiales |

---

## 5. EDI e Intercambio Electrónico (TR/WR/COCEDI/PUCEDI)

### 5.1 Visión general del subsistema EDI

El subsistema EDI permite a Fluidra intercambiar documentos comerciales electrónicos (según el estándar UN-EDIFACT) con clientes y proveedores a través de la plataforma Edicom.

**Componentes del subsistema:**

| Componente | Descripción |
|-----------|-------------|
| `COCEDI` | Configuración EDI por cliente (qué mensajes enviar/recibir) |
| `PUCEDI` | Configuración EDI por proveedor |
| `ED0060` | Generación de mensajes EDI |
| `TR0026/35/36/37/38` | Ficheros de trazabilidad de transacciones (órdenes) |
| `WR0026/36` | Ficheros de trabajo (Working Records) — copias en QTEMP |
| `EDTRAU/EDTRAX` | Auditoría de eventos EDI |
| `EDTRLP/EDTRLX` | Trazabilidad detallada por línea |
| `TRCPDE` | Trazabilidad de entorno de OC |

### 5.2 Configuración EDI — COCEDI (por cliente)

Cada cliente puede tener habilitado/deshabilitado el EDI. Campos principales:

| Campo | Descripción |
|-------|-------------|
| `S0COMP` | Rol comprador (EDIFACT) |
| `S0RECE` | Rol receptor |
| `S0CFAC` | Rol facturado |
| `S0PAGA` | Rol pagador |
| `S0EMIS` | Código emisor |
| `S0EFAC` | Habilitar factura electrónica (INVOIC) |
| `S0EALB` | Habilitar albarán electrónico (DESADV) |
| `S0RPED` | Habilitar recepción de PO del cliente (ORDERS) |
| `S0EFAB` | Habilitar nota de crédito |
| `S0EFCA` | Habilitar factura especial |
| `S0EBUL` | Envío masivo |
| `S0CAUT` | Auto-close Edicom |
| `FALT/FBAJ/CBJA` | Fecha alta/baja y código de baja |

### 5.3 Configuración EDI — PUCEDI (por proveedor)

| Campo | Descripción |
|-------|-------------|
| `C9EPED` | Enviar OC (ORDERS) al proveedor por EDI |
| `C9RRSP` | Recibir respuesta de OC (ORDRSP) |
| `C9RINV` | Recibir informe de inventario (INVRPT) |
| `C9RFAC` | Enviar factura (INVOIC) |
| `C9RALB` | Enviar albarán (DESADV) |

### 5.4 Mensajes EDIFACT soportados

| Mensaje | Dirección | Trigger | Descripción |
|---------|-----------|---------|-------------|
| `ORDERS` | AMS → Proveedor | ED0060 tras DOM | Orden de compra electrónica al proveedor |
| `ORDRSP` | Proveedor → AMS | Recepción Edicom | Respuesta del proveedor a la OC |
| `DESADV` | AMS → Cliente | Expedición | Aviso de expedición / albarán electrónico |
| `INVOIC` | AMS → Cliente | Facturación | Factura electrónica al cliente |
| `INVRPT` | Proveedor → AMS | Periódico | Informe de inventario del proveedor |

### 5.5 Ficheros de trazabilidad de transacciones (TR/WR)

Los ficheros con prefijo `TR` y `WR` son **definiciones de estructuras de datos** (DDS/PF o DDS/LF) que representan el modelo de datos transaccional de órdenes, con las mismas estructuras tanto en el lado de ventas (prefijo `S5`) como en el lado de compras (prefijo `C5`).

| Fichero | Prefijo campos | Contenido |
|---------|----------------|-----------|
| `TR0026` | `S5` | Estructura de línea de pedido de venta (transaccional) |
| `TR0035` | `C0` | Cabecera de orden de compra (estructura transaccional) |
| `TR0036` | `C5` | Línea de orden de compra (estructura transaccional) |
| `TR0037` | `C0` | Observaciones/mensajes de OC |
| `TR0038` | — | Datos adicionales de transacción |
| `TR0435` | — | Estructura de tránsito |
| `WR0026` | — | Copia de trabajo de TR0026 (en QTEMP) |
| `WR0036` | — | Copia de trabajo de TR0036 (en QTEMP) |

**Campos clave de TR0026 (línea de pedido venta):**

| Campo | Descripción |
|-------|-------------|
| `S5DLGA + S5CIAS + S5DATP + S5NOPL + S5NPCO + S5TPED + S5LINE` | Clave única de línea |
| `S5CANT` | Cantidad pedida |
| `S5CANS` | Cantidad entregada/servida |
| `S5CANC` | Cantidad cancelada |
| `S5PREC` | Precio de venta |
| `S5MONE` | Moneda |
| `S5LOGE` | Código de almacén lógico (resultado del DOM) |

**Campos clave de TR0035 (cabecera OC):**

| Campo | Descripción |
|-------|-------------|
| `C0DATP + C0CIAS + C0NPCO + C0PRVR` | Clave de OC |
| `C0FPED` | Fecha de pedido |
| `C0FRRE` | Fecha de recepción real |
| `C0FREP` | Fecha de recepción esperada |
| `C0TTTA` | Código de transportista |
| `C0TTTE` | Tipo de transporte |
| `C0DTPP` | Descuento de pronto pago |

### 5.6 Auditoría EDI — EDTRAU / EDTRLP

#### EDTRAU — Auditoría de eventos

Registra **cada evento significativo** del proceso EDI (confirmación, envío, error, recepción):

| Campo | Descripción |
|-------|-------------|
| `T1CIAS + T1DLGA + T1NPCO + T1TPED + T1DATP + T1HORA + T1LINE` | Clave |
| `T1EDAU` | Código de evento/auditoría |
| `T1EDAD` | Descripción del evento |
| `T1EDER` | Código de error (si aplica) |
| `T1DATO` | Datos del evento (600 bytes) |
| `T1USER` | Usuario que ejecutó |
| `T1JOBR` | Nombre del job |
| `T1PGMR` | Programa origen |
| `T1CANT` | Cantidad involucrada |
| `T1STUL` | Estado final |

#### EDTRLP — Trazabilidad de línea

Registra el **estado completo de cada línea** de pedido a lo largo del proceso:

| Campo | Descripción |
|-------|-------------|
| Clave | Empresa + Delegación + OC + Tipo + Fecha + Hora + Línea + Artículo |
| `T0CANT` | Cantidad solicitada |
| `T0CANR` | Cantidad recibida |
| `T0CANP` | Cantidad preparada |
| `T0CANS` | Cantidad servida |
| `T0CAND` | Diferencia |
| `T0FLAN` | Fecha de lanzamiento |
| `T0FREP` | Fecha de recepción esperada |
| `T0FURE` | Fecha solicitada por cliente |
| `T0EDSN` | Flag de envío EDI |
| `T0EDFC` | Código de función EDI |
| `T0RISK` | Flag de riesgo |
| `T0CIA[1-3]` / `T0DLG[1-3]` / `T0NPC[1-3]` | Referencias cruzadas a hasta 3 OC paralelas |

### 5.7 Trazabilidad de OC — TRCPDE

El fichero `TRCPDE` (con lógicos `TRCPDE01` y `TRCPDE02`) es la estructura de trazabilidad específica para el **entorno de proceso de la OC**, paralela a TR0026 pero orientada al seguimiento de compras.

Campos adicionales respecto a TR0026:
- `RMAN` — Flag de entrada manual
- `DEST/TDES/CIDT/CLDT` — Tracking de destino de la transacción
- `PROG` — Programa origen que generó el registro

---

## 6. Configuración y Módulos Auxiliares

### 6.1 Módulo de Balance Movex (MI*) — Integración ERP

> **Nota arquitectural:** Los programas `MI*` no son un módulo autónomo de inventario. Son la **interfaz de lectura del stock del ERP Movex/M3** replicado en IBM i. AMS lee este stock igual que cualquier fichero propio.

`MITBAL` es la réplica local del fichero de stock de Movex. Sus campos siguen el esquema nativo Movex:

| Campo | Descripción |
|-------|-------------|
| `MBCONO` | Código empresa Movex |
| `MBWHLO` | Almacén Movex (Warehouse Location) |
| `MBITNO` | Artículo Movex (Item Number) |
| `MBSTQT` | Stock aprobado disponible (on-hand approved) |
| `MBQUQT` | Cantidad en cuarentena/inspección |
| `MBAVAL` | Cantidad asignable (allocatable) |
| `MBORQT` | Cantidad pedida a proveedor (on order) |

| Programa | Función |
|----------|---------|
| `MITBAL` | Fichero físico de balance de stock Movex (fuente de datos para DOM) |
| `MITBAL00` | Índice alternativo por almacén |
| `MITBAL10` | Índice por tramos/rangos |
| `MITMAS` | Maestro de artículos Movex (información complementaria) |
| `MITMAS00` | Lógico del maestro de artículos |
| `MITFAC` | Facturación de inventario |
| `MITFAC00` | Lógico de facturación |

### 6.2 Gateway de Equivalencias Movex (GX*) — Traducción de Códigos

> **Nota arquitectural:** Los programas `GX*` no son gateways logísticos genéricos. Son el **motor de traducción bidireccional** entre los códigos internos de AMS (Aquaria/ASTRAL) y los códigos del ERP Movex/M3 (ITNO, WHLO, CONO). Sin ellos, AMS no puede comunicarse con Movex.

**Mecanismo de búsqueda (3 niveles):**
1. Equivalencia específica `CIAS + DLGA + CódigoAquaria`
2. Equivalencia a nivel empresa `CIAS + CódigoAquaria`
3. Equivalencia genérica del sistema `'*' + CódigoAquaria`

| Programa | Función |
|----------|---------|
| `GX0150` | Traducción de códigos AMS↔Movex (nivel empresa/delegación) |
| `GX2500` | Traducción de códigos AMS↔Movex (nivel avanzado / artículo) |
| `EQTABL00` | Fichero físico de tablas de equivalencias |
| `EQTABL10` | Índice lógico de equivalencias por tipo |
| `GXEQAR` | Equivalencia de artículos (equipos) |
| `GXEQAR03` | Variante de equivalencia de equipos |
| `GXEQPR` | Equivalencia de procesos de equipos |
| `GXSTO0` | Gateway de stock — consulta MITBAL con traducción de códigos |
| `GXSTO02` | Variante de consulta de stock Movex |

### 6.3 Gestión de Riesgos (ZQ*/ZA*)

| Programa | Función |
|----------|---------|
| `ZQGSRK` | Scoring de riesgo de cliente |
| `ZQGSRK00` | Fichero físico de riesgos |
| `ZQGCSC` | Gestor asignado al cliente |
| `ZQGCSC00` | Fichero físico de gestor asignado |
| `ZAQBAL` | Balance de riesgos |
| `ZAQBAL00` | Fichero físico de balance |
| `ZAQITN` | Itinerario/seguimiento de riesgos |
| `ZAQITN00` | Fichero físico |

### 6.4 Utilidades y Herramientas (UT*)

| Programa | Función |
|----------|---------|
| `UTCOM0` | Comunicación y mensajería general |
| `UTCOMA` | Variante de comunicación |
| `UTDEST` | Gestión de destinatarios de mensajes |
| `UT2725` | Envío de emails de confirmación (invocado por CO0120, CO2519) |

### 6.5 Data Processing / Procesos Batch (DP*)

| Programa | Función |
|----------|---------|
| `DP0090` | Proceso batch principal (job 90) |
| `DPMART` | Proceso de artículos en batch |
| `DPMART02` | Variante batch de artículos |
| `DPMENC` | Proceso de menciones/textos |
| `DPMENT` | Mantenimiento de entidades batch |

### 6.6 Online Order / eCommerce (OO*)

| Programa | Función |
|----------|---------|
| `OOLINE` | Gestión de líneas de pedido online |
| `OOLINE00` | Fichero físico |
| `OOLINE40` | Variante de líneas online |

### 6.7 Proveedores de Servicios (PV*)

| Programa | Función |
|----------|---------|
| `PVDEVD` | Devoluciones de proveedor (detalle) |
| `PVDEVD10` | Lógico de devoluciones |
| `PVDEVI` | Devoluciones de proveedor (inventario) |
| `PVDEVI05` | Lógico de devoluciones inventario |

### 6.8 API y Conectividad

| Programa | Función |
|----------|---------|
| `APIENWF` | API de entrada a workflow — punto de entrada para integraciones externas |
| `EIEXCE` | Exportación a Excel (interfaz con hoja de cálculo) |
| `SVUSER` | Supervisor de usuarios — gestión de sesiones |

### 6.9 Configuración del sistema (CM* / CF* / CC*)

> **Nota arquitectural:** `CMNCMP` y `CMNDIV` son los **maestros organizativos en formato Movex**. Sus campos (CONO, DIVI, WHLO, ITNO, FACI, PLNT) siguen el esquema del ERP Movex/M3 y son la fuente de verdad para las claves de acceso al ERP externo.

| Programa | Función |
|----------|---------|
| `CM0085CL` | CL wrapper para entrada al módulo de compras |
| `CMA040` | Datos adicionales de pedido |
| `CMNCMP` | Maestro de empresas en formato Movex (campos: CONO, FACI, WHLO) |
| `CMNDIV` | Maestro de divisiones en formato Movex (campos: CONO, DIVI, PLNT, ITNO) |
| `CMNUSR` | Mantenimiento de usuarios |
| `CFACIL` | Configuración de facilidades/instalaciones |
| `CCUDIV` | Configuración de divisiones de cliente |

---

## 7. Glosario y Conceptos Clave

| Término | Definición |
|---------|-----------|
| **AMS** | Application Maintenance and Support — nombre del sistema/proyecto |
| **Movex / M3** | ERP Infor M3 (anteriormente Lawson M3) sobre el que AMS opera como capa operacional. Los módulos CM/GX/MI son el puente de integración |
| **CONO** | Company Number — código de empresa en nomenclatura Movex/M3 |
| **DIVI** | Division — división organizativa en nomenclatura Movex/M3 |
| **WHLO** | Warehouse Location — almacén en nomenclatura Movex/M3 |
| **ITNO** | Item Number — código de artículo en nomenclatura Movex/M3 |
| **CIAS** | Código de empresa (3 dígitos) — el sistema es multiempresa |
| **DLGA** | Delegación (5 caracteres alfanuméricos) — unidad organizativa de ventas/compras |
| **LOGE** | Almacén lógico — vista de negocio de un almacén (empresa + delegación) |
| **MAGA** | Almacén físico — edificio/ubicación real |
| **TPED** | Tipo de pedido — determina reglas de comportamiento del pedido |
| **NPCO** | Número de pedido de cliente |
| **NORT** | Número de orden de venta/tracing |
| **NOPL** | Número de línea del pedido (implementado como timestamp) |
| **DATP** | Fecha del pedido |
| **DOM** | Distribution Order Management — sistema de gestión de distribución |
| **DOOM** | Algoritmo central del DOM: optimización por cubo de decisión 3D |
| **TREG** | Tipo de regla de distribución (0-5 nivel empresa, A-H nivel delegación) |
| **EDPAR4** | Fichero maestro de prioridades del DOM (empresa + almacén + tipo) |
| **MSDDOM** | Matriz de delegaciones: reglas de envío entre delegaciones |
| **Coenvío** | Grupo de líneas de pedido que se expiden juntas al 100% |
| **Entrega Directa (ED)** | Envío directo del proveedor al cliente, sin pasar por almacén Fluidra |
| **Fluidra Direct (FD)** | Canal de distribución directa desde empresa productiva |
| **MTS** | Make-To-Stock — artículo fabricado para stock, servido desde almacén |
| **MTO** | Make-To-Order — artículo fabricado bajo pedido |
| **Trace** | Sistema de gestión de stock; `SIOL='2'` indica almacenamiento en Trace |
| **KORE** | Motor externo de cálculo de disponibilidad; consume CO2610 |
| **Edicom** | Plataforma intermediaria para intercambio EDI |
| **EDIFACT** | Estándar UN/EDIFACT para mensajes electrónicos comerciales |
| **ORDERS** | Mensaje EDIFACT de Orden de Compra |
| **ORDRSP** | Mensaje EDIFACT de Respuesta a OC |
| **DESADV** | Mensaje EDIFACT de Aviso de Expedición |
| **INVOIC** | Mensaje EDIFACT de Factura |
| **INVRPT** | Mensaje EDIFACT de Informe de Inventario |
| **QTEMP** | Librería temporal IBM i (por job) — usada para aislamiento de procesos |
| **SFL** | Subfile — pantalla de lista paginada en IBM i |
| **ZXOPT** | Variable de control de flujo (máquina de estados) en programas RPG |
| **LDA** | Local Data Area — área de datos compartida dentro de un job IBM i |
| **PURLDADS** | Estructura de datos del LDA del módulo de compras |
| **@PERRO** | Parámetro de retorno/error estándar en programas de servicio |
| **DPM** | Método de cálculo de plazos de entrega (referencia Sr. Rifà) |
| **PECLTA** | Clave de tabla de numeración de órdenes en PMTABD |
| **QUSER** | Usuario especial IBM i para trabajos batch/API |

---

*Sistema AMS — Fluidra S.A. | Manual Funcional | Versión 2.0 — Marzo 2026*
