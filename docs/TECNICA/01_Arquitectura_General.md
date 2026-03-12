# Arquitectura General del Sistema AMS — Fluidra S.A.
## Documentación Técnica de Arquitectura, Patrones y Flujos

**Versión:** 2.0 — Marzo 2026
**Audiencia:** Técnico senior, arquitecto de sistemas, equipo AMS externo
**Clasificación:** Interno — Técnico

---

## Índice

1. [Entorno Técnico](#1-entorno-técnico)
2. [Mapa de Módulos y Dependencias](#2-mapa-de-módulos-y-dependencias)
3. [Arquitectura de Programas RPG](#3-arquitectura-de-programas-rpg)
4. [Patrón de Control de Flujo ZXOPT](#4-patrón-de-control-de-flujo-zxopt)
5. [Modelo de Datos Global](#5-modelo-de-datos-global)
6. [Arquitectura de Ficheros DDS](#6-arquitectura-de-ficheros-dds)
7. [Flujo de Llamadas entre Programas](#7-flujo-de-llamadas-entre-programas)
8. [Integraciones Externas](#8-integraciones-externas)
9. [Mecanismos Técnicos Transversales](#9-mecanismos-técnicos-transversales)
10. [Convenciones de Nomenclatura](#10-convenciones-de-nomenclatura)
11. [Gestión de Modificaciones y Trazabilidad del Código](#11-gestión-de-modificaciones-y-trazabilidad-del-código)
12. [Entornos de Ejecución y Aislamiento](#12-entornos-de-ejecución-y-aislamiento)

---

## 1. Entorno Técnico

| Atributo | Valor |
|----------|-------|
| **Plataforma** | IBM i (AS/400 / iSeries) |
| **Lenguaje principal** | RPG IV (RPGLE) — Free-format + fixed-format mixto |
| **Lenguaje secundario** | CL (Control Language) — 4 programas wrapper/orquestadores |
| **Sistema Operativo** | OS/400 / IBM i OS |
| **Formato de fecha** | `*YMD/` — `DATEDIT(*YMD/)` — formato YY/MM/DD en pantalla |
| **Codificación fuente** | UTF-8 con terminadores CRLF |
| **Formato de miembros** | `.MBR` — miembros de fichero fuente IBM i |
| **Acceso a BD** | Nativo IBM i DB2: CHAIN/SETLL/READE/READ/READPE con acceso keyed |
| **Pantallas** | Subfile (SFL) y formatos WORKSTN via Display Files (.FM) |
| **Opciones de compilación** | `H OPTION(*NODEBUGIO:*SRCSTMT)` |
| **Total programas** | 418 miembros fuente |
| **Total líneas de código** | ~153.000 |

### 1.1 Librerías del sistema

El sistema usa un esquema de librerías IBM i típico:

| Librería | Propósito |
|----------|-----------|
| **Producción** | Contiene los objetos compilados (*PGM, *FILE) de producción |
| **QTEMP** | Librería temporal de trabajo por job — usada para copias de ficheros aisladas durante el proceso DOM |
| **Librería de datos** | Contiene los ficheros físicos de BD (PF) y lógicos (LF) |
| **Librería de programas** | Contiene los programas compilados |

La lista de librerías (`PMLIBL`) se recupera dinámicamente por empresa/delegación al inicio de cada sesión.

---

## 2. Mapa de Módulos y Dependencias

### 2.1 Diagrama de alto nivel

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SISTEMA AMS — FLUIDRA                          │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  PEDIDOS DE  │    │   COMPRAS    │    │   MOTOR DOM / DOOM       │  │
│  │  VENTA (CO)  │───▶│    (PU)      │◀───│   (ED / DO / DOM)        │  │
│  │   ~106 pgms  │    │   ~37 pgms   │    │   ~61 pgms               │  │
│  └──────┬───────┘    └──────┬───────┘    └────────────┬─────────────┘  │
│         │                   │                         │                │
│         └───────────────────┴─────────────────────────┘                │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │           MAESTROS Y PARAMETRIZACIÓN (MS / PM / MM)              │  │
│  │   Artículos · Stock · Almacenes · Unidades · Parámetros Empresa  │  │
│  │                        ~93 programas                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                             │                                           │
│         ┌───────────────────┴────────────────────┐                    │
│         ▼                                         ▼                    │
│  ┌─────────────────┐                   ┌──────────────────────────┐   │
│  │  EDI / INTERCAM.│                   │  CONFIG. Y AUXILIARES    │   │
│  │  (TR/WR/COCEDI) │                   │  (CM/GX/CF/MI/ZQ/UT)     │   │
│  │    ~20 pgms     │                   │    ~101 pgms             │   │
│  └─────────────────┘                   └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Dependencias entre módulos

| Módulo origen | Módulo destino | Tipo de dependencia |
|--------------|----------------|---------------------|
| CO (Pedidos) | MS (Maestros) | Lee artículos, stock, almacenes, clientes |
| CO (Pedidos) | PM (Parámetros) | Lee configuración de empresa/delegación |
| CO (Pedidos) | ED/DOM | Activa motor DOM al confirmar pedido |
| ED/DOM | PU (Compras) | Genera órdenes de compra tras decisión |
| ED/DOM | MS (Stock) | Lee stock disponible por almacén |
| ED/DOM | TR/WR (EDI) | Genera mensajes EDI al proveedor |
| PU (Compras) | MS (Maestros) | Lee artículos, proveedores, almacenes |
| TR/WR (EDI) | COCEDI/PUCEDI | Lee configuración EDI del cliente/proveedor |
| CM/GX/CF | PM (Parámetros) | Mantiene tablas de configuración |

---

## 3. Arquitectura de Programas RPG

### 3.1 Clasificación por tipo funcional

| Tipo | Descripción | Ejemplos |
|------|-------------|----------|
| **Programa interactivo SFL** | Lista paginada de registros (Subfile). Tiene formato WORKSTN con SFILE. | CM0085, CO0120, DO0001, DO0020, DO0030, DO0035 |
| **Programa interactivo panel** | Formulario de detalle (sin SFL). Formato WORKSTN simple. | CO2593, DO0005, DO0010, DO0025, ED3025 |
| **Subprograma de servicio** | Sin pantalla. Invocado por CALL desde otro programa. | CO2611, CO2696, CO2733, CO2755, CO2610 |
| **Programa de proceso/batch** | Orquesta operaciones BD. Puede ser interactivo o batch. | CO2519, ED3050, DOOM01, ED0060 |
| **Fichero físico (PF)** | Definición DDS de tabla de base de datos. | COCAPE, CODEPE, EDPAR4, MSSTOK |
| **Fichero lógico (LF)** | Vista/índice sobre un PF. Acceso keyed alternativo. | COCAPE23, CODEPE42, EDTRLP00 |
| **Fichero pantalla (DSPF)** | Definición DDS de formatos de pantalla/subfile. | CM0085FM, CO0120FM, DO0001FM |
| **Programa CL** | Wrapper de entrada batch o inicialización de entorno. | ED1000CL, ED3025CL, CM0085CL, CONOCL |

### 3.2 Estructura interna estándar de un programa RPG

Todos los programas siguen esta estructura:

```
┌─────────────────────────────────────────────────────────┐
│  H - Header (control)                                   │
│      DATEDIT(*YMD/)  OPTION(*NODEBUGIO:*SRCSTMT)        │
├─────────────────────────────────────────────────────────┤
│  F - File declarations                                  │
│      ├── Fichero de pantalla (WORKSTN)  — si interactivo│
│      └── Ficheros de BD (DISK):                         │
│          IF=input, UF=update, O=output, A=append        │
├─────────────────────────────────────────────────────────┤
│  D - Data definitions                                   │
│      ├── SDS  — Program Status Data Structure           │
│      ├── DSLDA — Local Data Area (LDA) de compras       │
│      ├── @Pxxxx — Parámetros de entrada/salida          │
│      ├── ZXxxxx — Variables de trabajo internas         │
│      ├── ZXOPT  — Control de flujo (máquina de estados) │
│      └── INDDS  — Indicadores de pantalla               │
├─────────────────────────────────────────────────────────┤
│  C - Calculations                                       │
│      *INZSR → Inicialización (*ENTRY PLIST, IN LDA)     │
│                                                         │
│      DOW ZXOPT1 <> 'END'                                │
│        CASEQ 'CLR'  SUBCLR  → Limpiar SFL/pantalla      │
│        CASEQ 'INZ'  SUBINZ  → Cargar datos/SFL          │
│        CASEQ 'DSP'  SUBDSP  → Visualizar (EXFMT)        │
│        CASEQ 'CMD'  SUBCMD  → Validar teclas función    │
│        CASEQ 'CHK'  SUBCHK  → Validar campos pantalla   │
│        CASEQ 'UPD'  SUBUPD  → Actualizar BD             │
│      ENDDO                                              │
│      SETON LR                                           │
│                                                         │
│      [Subrutinas de proceso específicas]                │
│      [CS_ Common Subroutines — al final del programa]   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Patrón de Control de Flujo ZXOPT

El patrón `ZXOPT` es el mecanismo central de control de flujo utilizado en **todos** los programas interactivos. Es una implementación de máquina de estados sobre una variable de 6 caracteres.

### 4.1 Estructura

```rpg
D  ZXOPT          DS
D    ZXOPT1               1      3    ← Nombre del proceso activo
D    ZXOPT2               4      6    ← Rutina dentro del proceso
```

### 4.2 Valores de ZXOPT1 (proceso)

| Prefijo | Tipo de proceso | Descripción |
|---------|----------------|-------------|
| `Snn` | Subfile N | Proceso del subfile nº N (nn=01..99). N coincide con el número de formato de pantalla |
| `Pnn` | Pantalla N | Proceso de formato de panel/formulario |
| `Wnn` | Ventana N | Proceso de ventana/popup |
| `END` | Fin | Salida del programa (SETON LR) |

### 4.3 Valores de ZXOPT2 (rutina)

| Valor | Rutina | Descripción |
|-------|--------|-------------|
| `CLR` | SUBCLR | Limpiar el subfile/pantalla (borrar registros cargados) |
| `INZ` | SUBINZ | Inicializar y cargar el subfile/pantalla con datos |
| `DSP` | SUBDSP | Visualizar la pantalla (EXFMT) y esperar respuesta del usuario |
| `CMD` | SUBCMD | Validar las teclas de función pulsadas (F3, F12, Enter, etc.) |
| `CHK` | SUBCHK | Validar los campos modificados en pantalla |
| `UPD` | SUBUPD | Aplicar los cambios a la base de datos |

### 4.4 Ciclo de vida de un subfile

```
                    ┌─────────────────────────────────────┐
                    │         Inicio del programa          │
                    │   ZXOPT = 'S01CLR'                  │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │  CLR — Limpiar SFL                  │
                    │  Borrar registros cargados           │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │  INZ — Inicializar/Cargar SFL        │
                    │  READE fichero → WRITE SFILE         │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │  DSP — Visualizar pantalla           │
             ┌──────┤  EXFMT pantalla → esperar usuario   │
             │      └──────────────────┬──────────────────┘
             │                         │
             │      ┌──────────────────▼──────────────────┐
             │      │  CMD — Analizar tecla pulsada        │
             │      │  F3/F12 → ZXOPT='END'               │
             │      │  Pg↑/Pg↓ → ZXOPT='S01INZ' (repag.) │
             │      │  Enter   → ZXOPT='S01CHK'           │
             │      └──────────────────┬──────────────────┘
             │                         │
             │      ┌──────────────────▼──────────────────┐
             │      │  CHK — Validar campos/opciones       │
             │      │  Error → ZXOPT='S01DSP' (vuelve)    │
             │      │  OK    → ZXOPT='S01UPD'             │
             │      └──────────────────┬──────────────────┘
             │                         │
             │      ┌──────────────────▼──────────────────┐
             └──────┤  UPD — Actualizar BD + CALL externos │
                    │  Procesa opción seleccionada         │
                    │  → ZXOPT='S01CLR' (nuevo ciclo)     │
                    └──────────────────────────────────────┘
```

### 4.5 Variables auxiliares de subfile

| Variable | Descripción |
|----------|-------------|
| `ZXNNNnn` | Último nº de registro leído para el subfile nn (marcador de posición para READE) |
| `ZXNRRnn` | Contador de registros cargados en el subfile |
| `ZXCOUNT` | Posición del primer registro de la última página visualizada |
| `ZXLOPnn` | Contador de registros en la página actual |
| `SUFRCDN` | SFLRCDNBR — posición del cursor en la lista (campo de pantalla) |

---

## 5. Modelo de Datos Global

### 5.1 Entidades principales y relaciones

```
PMEMPR (Empresa/Parámetros)
   ├── define límites y reglas para → EDPAR4, MSARTI00, PMTABD
   └── es el hub central de configuración del sistema

MSALMA (Almacén físico)
   └── 1:N → MSLOAD (Almacén lógico por empresa/delegación)

MSARTI00 (Artículo maestro)
   ├── 1:N → MSSTOK  (Stock por almacén)
   ├── 1:N → MSCOJD  (Componentes si es un kit)
   └── N:1 → MSFAMI  (Familia del artículo)

COCAPE (Cabecera pedido venta)
   ├── N:1 → COCLIN  (Cliente)
   ├── N:1 → MSLOAD  (Almacén asignado)
   └── 1:N → CODEPE  (Líneas del pedido)

CODEPE (Línea pedido venta)
   ├── N:1 → MSARTI00  (Artículo)
   ├── N:1 → MSSTOK    (Stock disponible)
   └── N:1 → EDPAR4    (Reglas logísticas DOM)

PUCAPE (Cabecera orden de compra)
   ├── N:1 → PUPROV   (Proveedor)
   └── 1:N → PUDEPE   (Líneas de la OC)

EDPAR4 (Prioridades de distribución DOM)
   ├── N:1 → MSALMA   (Almacén)
   ├── N:1 → PUPROV   (Proveedor)
   └── conecta CODEPE con PUCAPE a través del DOM
```

### 5.2 Clave de pedido

Un pedido de venta se identifica unívocamente por:

```
CIAS (3 num) + DLGA (5 alfa) + NPCO (num) + TPED (2 alfa) + DATP (fecha)
```

Una línea de pedido:
```
CIAS + DLGA + NPCO + TPED + DATP + NOPL
```

Donde `NOPL` = número de línea (implementado como timestamp — `PEHORA`).

Una orden de compra:
```
CIAS + DLGA + NPCO + TPED + DATP
```
*(misma estructura, distinto módulo)*

### 5.3 Tabla de numeración de órdenes

La numeración de órdenes de venta se gestiona mediante la tabla `PMTABD` (clave `PECLTA`).
Búsqueda en cascada (de más específica a menos):

```
1. CIAS + DLGA + TPED   → numerador específico empresa/delegación/tipo
2. CIAS + DLGA          → numerador empresa/delegación
3. CIAS + TPED          → numerador empresa/tipo
4. CIAS                 → numerador genérico de empresa
```

---

## 6. Arquitectura de Ficheros DDS

### 6.1 Módulo de Pedidos de Venta (CO)

```
COCAPE          — Cabecera de pedido (físico)
├── COCAPE01    — Lógico: acceso estándar
├── COCAPE02    — Lógico: acceso por estado
├── COCAPE03    — Lógico: acceso por fecha
├── COCAPE04    — Lógico: acceso por delegación
├── COCAPE05    — Lógico: acceso por tipo
├── COCAPE06    — Lógico: acceso por almacén
├── COCAPE07    — Lógico: acceso por artículo
├── COCAPE08    — Lógico: para cálculo de importes
├── COCAPE11    — Lógico: uso específico
├── COCAPE21    — Lógico: uso específico
├── COCAPE23    — Lógico: acceso por cliente
├── COCAPE25    — Lógico: incluye pedidos status '10' (pendientes autorización riesgo)
├── COCAPE27    — Lógico: acceso específico
├── COCAPE31    — Lógico: acceso por referencia
├── COCAPE38    — Lógico: ordenación descendente por pedido
└── COCAPEA8    — Lógico: pedidos activos (usado por KORE)

CODEPE          — Detalle/líneas de pedido (físico)
├── CODEPE01-09 — Lógicos varios (acceso por distintos criterios)
├── CODEPE12    — Lógico: uso específico
├── CODEPE20-22 — Lógicos: variantes de acceso
├── CODEPE26    — Lógico: acceso específico
├── CODEPE35    — Lógico: acceso por estado
├── CODEPE40    — Lógico: acceso por fecha entrega
└── CODEPE42    — Lógico: acceso por artículo (usado por KORE CO2610)

CODIPE          — Datos complementarios de línea
COOBPE          — Observaciones de pedido
COPECO          — Condiciones de pedido
COGAPE          — Agrupaciones de pedido
COCAIM          — Cabecera de importes
CODEP2          — Detalle pedido vista 2
CMA040          — Datos adicionales de pedido
├── CMA04024    — Lógico
└── CMA04034    — Lógico
COCLIN          — Maestro de clientes de pedido
├── COCLIN09    — Lógico: para cálculo de importes
COCPED          — Pedidos EDI pendientes de importar
```

### 6.2 Módulo de Compras (PU)

```
PUCAPE          — Cabecera de orden de compra (físico)
└── PUCAPE21    — Lógico de acceso

PUDEPE          — Líneas de orden de compra (físico)
PUDIPE          — Datos complementarios de línea de compra
PUOBPE          — Observaciones de OC
PUPECO          — Condiciones de OC
PUCAIM          — Cabecera de importes OC
PUCEDI          — Configuración EDI por proveedor
PUPROV          — Maestro de proveedores
PUHALD          — Histórico de albaranes de compra
└── PUHALD17    — Lógico de histórico
PUPRMD          — Parámetros de OC
└── PUPRMD02    — Lógico de parámetros
PUDTO0/1        — Datos de tránsito y recepción
└── DUDTO101    — Lógico
PUPRP0          — Parámetros del proceso de compra
└── PUPRP004    — Lógico
```

### 6.3 Módulo DOM / Distribución (ED/DO)

```
EDPAR4          — Prioridades de distribución DOM (físico principal)
└── EDPAR401    — Lógico de acceso

EDPAR2          — Parámetros secundarios de prioridades (por almacén)
EDPAR3          — Restricciones de capacidad por almacén
EDPARC          — Reglas específicas por cliente
EDPARM          — Parámetros DOM por empresa/delegación
EDHPA4          — Histórico de cambios en EDPAR4
└── EDHPAR      — Lógico del histórico

EDTRLP          — Trazabilidad de línea por pedido (físico)
├── EDTRLP00    — Lógico: acceso estándar
├── EDTRLP10    — Lógico: por empresa
├── EDTRLP20    — Lógico: por delegación
├── EDTRLP30    — Lógico: por artículo
└── EDTRLP40    — Lógico: por estado

EDTRLX          — Trazabilidad extendida / histórico
├── EDTRLX00-40 — Lógicos equivalentes a EDTRLP

EDTRAU          — Auditoría de eventos EDI (físico)
└── EDTRAU00    — Lógico de acceso

EDTRAX          — Auditoría extendida / excepciones
└── EDTRAX00    — Lógico de acceso

DOOMHT          — Histórico de decisiones DOOM
DOM005-008      — Tablas de trabajo DOOM
└── DOM00500-800 — Lógicos correspondientes

DO0050W-053W    — Ficheros de trabajo DOM (en QTEMP durante proceso)
```

### 6.4 Maestros Compartidos (MS/PM)

```
MSARTI00        — Maestro de artículos (principal)
├── MSARTI03    — Lógico: por familia
├── MSARTI07    — Lógico: por almacén
└── MSARTI10    — Lógico: por código alternativo

MSSTOK          — Stock por almacén (físico)
└── MSSTOK01    — Lógico de acceso

MSALMA          — Almacenes físicos (físico)
├── MSALMA00    — Lógico estándar
├── MSALMA02-03 — Lógicos adicionales
├── MSALMA05    — Lógico: para CO2755 (clasificación rotación)
└── MSALMA08    — Lógico: Fluidra Direct

MSLOAD          — Almacenes logísticos (físico)
└── MSLOAD02    — Lógico de acceso

MSCOJC          — Cabecera de conjuntos/kits (físico)
MSCOJD          — Componentes de conjuntos (físico)
MSFAMI          — Maestro de familias
MSDDOM          — Delegaciones origen-destino (físico)
├── MSDDOM11    — Lógico: prioridad OB (Overstock)
├── MSDDOM12    — Lógico: prioridad OC (Obsolete)
├── MSDDOM13    — Lógico: prioridad SSTOCK (Super-stock)
└── MSDDOM14    — Lógico: prioridad ART.B

PMEMPR          — Parámetros de empresa (hub central de config)
└── PMEMPR00    — Lógico de acceso

PMTABL          — Tablas de parámetros (maestro)
PMTABD          — Detalle de tablas
└── PMTABD90    — Lógico específico

PMUNAL          — Unidades de medida y factores de conversión
MSTRTA          — Tipos de transporte/carrier
MSTRG1          — Tabla de rangos/grupos
```

---

## 7. Flujo de Llamadas entre Programas

### 7.1 Flujo del pedido de venta completo

```
CL wrapper (CONOCL / CM0085CL)
   │
   ▼
CM0085  ←→  CO0120  ←→  CO0140
(lista)     (cabecera)   (líneas)
                │
                ├── CALL CO2611 (calcular fecha entrega)
                ├── CALL CO2755 (categoría rotación artículo)
                ├── CALL CO2696 (validar coenvío)
                ├── CALL CO2593 (parámetros impresión)
                ├── CALL UT2725 (email confirmación)
                └── CALL CO2519 (lanzar orden)
                        │
                        ├── CALL CO2755 (rotación)
                        └── CALL UT2725 (email)
```

### 7.2 Flujo del Motor DOM

```
ED3025CL  (CL wrapper — punto de entrada batch)
   │
   ▼
ED3025  (orquestador DOM)
   │
   ├── CALL ED3050  (DOOM algorithm)
   │       │
   │       ├── Lee COCAPE/CODEPE (pedido)
   │       ├── Lee MSALMA/MSLOAD (almacenes)
   │       ├── Lee MSSTOK (stock)
   │       ├── Lee EDPAR2/3/4/C/M (parámetros)
   │       ├── Lee MSDDOM/11-14 (delegaciones)
   │       ├── Lee PUPROV/MSTRTA (proveedores/transporte)
   │       ├── Lee MSCOJC/MSCOJD (kits)
   │       └── Escribe CODEPE.S5LOGE (resultado: almacén óptimo)
   │
   ├── CALL ED3061  (pantalla confirmación — si no hay error de riesgo)
   │
   ├── CALL ED0060  (generación EDI — tras confirmación)
   │       │
   │       ├── Lee EDPARM/EDPARC/EDPAR4 (qué socios reciben EDI)
   │       ├── Lee PUCEDI (configuración EDI proveedor)
   │       └── Genera mensajes: ORDERS, DESADV, INVOIC, ORDRSP
   │
   └── CALL ED0100  (trazabilidad y auditoría)
           │
           └── Escribe EDTRAU, EDTRLP, DOOMHT
```

### 7.3 Flujo de compras (PU)

```
PU2519  (consulta OC)  ←→  PU2600 (cabecera OC)  ←→  PUPRPR (proceso OC)
                                │
                                ├── PUPRMD (parámetros modificación)
                                ├── PUFACC (facturación compras)
                                └── PUHALD (histórico albaranes)
```

### 7.4 Flujo EDI (mensajes con el exterior)

```
ED0060 (generación mensajes)
   │
   ├── ORDERS (OC a proveedor)
   │       └── vía PUCEDI → Edicom → Proveedor externo
   │
   ├── DESADV (aviso entrega del proveedor)
   │       └── vía COCEDI → Edicom → Cliente
   │
   ├── INVOIC (factura al cliente)
   │       └── vía COCEDI → Edicom → Cliente
   │
   ├── ORDRSP (respuesta OC del proveedor)
   │       └── del Proveedor → Edicom → EDTRAU (registro)
   │
   └── INVRPT (informe inventario)
           └── vía PUCEDI.C9RINV → proveedor habilitado
```

---

## 8. Integraciones Externas

### 8.1 EDI / Edicom (UN-EDIFACT)

- **Protocolo:** UN-EDIFACT (estándar europeo de intercambio electrónico)
- **Plataforma intermediaria:** Edicom
- **Configuración:** COCEDI (por cliente) y PUCEDI (por proveedor)
- **Mensajes soportados:**

| Mensaje EDIFACT | Dirección | Descripción |
|-----------------|-----------|-------------|
| `ORDERS` | AMS → Proveedor | Orden de compra |
| `ORDRSP` | Proveedor → AMS | Respuesta a la OC |
| `DESADV` | AMS → Cliente | Aviso de expedición |
| `INVOIC` | AMS → Cliente | Factura electrónica |
| `INVRPT` | Proveedor → AMS | Informe de inventario del proveedor |

- **Programas clave:** `ED0060` (generación), `EDTRAU`/`EDTRLP` (auditoría), `COCEDI`/`PUCEDI` (configuración socios)
- **Roles EDIFACT:** COMP (comprador), RECE (receptor), CFAC (facturado), PAGA (pagador)

### 8.2 SalesForce API

- **Tipo:** Llamada batch desde web service externo
- **Mecanismo:** SalesForce invoca programas AMS con usuario `QUSER` (batch IBM i)
- **Detección:** Programa `RTVTIJ` retorna `@PTIPJ='0'` (batch) → programas no abren pantalla
- **Programa afectado:** `CO2593` (modificación `B0251`, jun-2020, autor: Lluís Albert Grau)

### 8.3 M3 / Movex

- **Tipo:** Integración de coenvíos y albaranes a terceros
- **Programas afectados:** `CO2519`, `CO0120` (modificación `MOVEX`, ago-2006)
- **Estado:** Parte del código comentado (`MOVEXF*`) indica que algunas funcionalidades han sido desactivadas

### 8.4 Motor KORE

- **Tipo:** Consulta externa de disponibilidad de artículos
- **Mecanismo:** El motor KORE llama a `CO2610` pasando empresa, delegación, artículo y rango de fechas
- **Resultado:** Cantidad total pendiente de servir en unidad base del artículo
- **Alias:** El programa físico es `CO2610` pero se referencia como `CO2770` en algunas librerías

### 8.5 Fluidra Direct (FD)

- **Tipo:** Canal de distribución directa — empresa productiva → cliente final
- **Impacto:** Lógica especial de cálculo de plazos en `CO0120`, `CO2519`
- **Identificación:** Flag `LOFD='1'` en parámetros, o MSALMA08 para almacenes FD
- **Programas específicos:** `ED1000`/`ED1000CL` (simulación entregas directas), `DO0010` (parámetros FD)

---

## 9. Mecanismos Técnicos Transversales

### 9.1 Local Data Area (LDA) — PURLDADS

Todos los programas interactivos recuperan el área de datos local al inicio. La estructura `PURLDADS` comparte el contexto de trabajo entre programas del mismo job:

```rpg
C     *DTAARA    DEFINE    PURLDA    DSLDA
C                IN        DSLDA
```

Campos principales de `PURLDADS`:

| Campo | Descripción |
|-------|-------------|
| `ZZCIAS` | Empresa activa del contexto de trabajo |
| `ZZDLGA` | Delegación activa |
| `ZZDEEM` | Delegación del empleado (posición 30 del LDA) |

### 9.2 Estructuras de datos externas (DS externas)

Los programas con múltiples formatos de pantalla usan DS externas para pasar parámetros entre invocaciones sin enumerar cada campo:

```rpg
D  CM0085    E DS    EXTNAME(CM0085DS)    ← parámetros del pgm CM0085
D  CO0120    E DS    EXTNAME(CO0120DS)    ← parámetros del pgm CO0120
D  DO0001    E DS    EXTNAME(DO0001DS)    ← parámetros del pgm DO0001
```

### 9.3 Gestión de cursor en pantalla

```rpg
D  FMTDS      DS
D    FMTPOCUR       370   371B 0    ← posición del cursor (fila*256 + columna)
D    FMTRCD    *RECORD               ← nombre del último formato activo

C     FMTPOCUR  DIV   256   CSRFILA    ← extrae fila del cursor
C               MVR         CSRCOLU    ← extrae columna (resto)
```

### 9.4 Indicadores como estructura de datos

Los indicadores del sistema se mapean sobre una estructura basada en puntero:

```rpg
D  INDICADORS  E DS         BASED(INDICADORP)
D                             EXTNAME(INDDS:RIND0199)
D  INDICADORP  S    *         INZ(%ADDR(*IN))
```

Indicadores con nombre en lugar de número:
- `ERRGRL` → error general
- `EXIT`, `RETRO`, `HELP` → teclas de función (F3, F12, F1)
- `WRKnn` → indicadores de trabajo temporales
- `SETLnn`, `READnn` → posicionamiento/lectura de ficheros

### 9.5 Detección de tipo de trabajo (batch vs interactivo)

```rpg
C     CALL    'RTVTIJ'
C     PARM    *BLANKS    @PTIPJ    ← '0'=batch, '1'=interactivo
```

Usado en `CO2593` para soportar llamadas desde SalesForce API sin abrir pantallas.

### 9.6 Aislamiento de procesos con QTEMP

El motor DOM copia los ficheros necesarios a la librería temporal QTEMP al inicio del proceso `ED3025`. Esto garantiza:
- Aislamiento de transacciones concurrentes
- Sin bloqueos de registros entre jobs paralelos
- Limpieza automática al finalizar el job

Ficheros copiados a QTEMP: `COCAPE`, `CODEPE`, `CODEPE20`, `CODEPE26`, `ED0050W`, `COHFEN`.

### 9.7 Conversión de unidades (CS_CNVUBS)

Subrutina presente en `CO2610` y otros programas de servicio:

```
1. Lee PMUNAL: CIAS + ARTI + VARI + MODE + UNIM_ORIGEN
2. Si no encuentra: reintenta con VARI=blancos, MODE=blancos
3. Si P6TCOF <> '1': resultado = cantidad × P6COFA  (multiplicar)
4. Si P6TCOF = '1': resultado = cantidad / P6COFA   (dividir)
5. Ajuste decimal (COPY SRD001)
```

### 9.8 Validación mediante diccionario MMPROM

Para campos con lista de valores válidos:

```rpg
C     CALL    'MMPROM'
C     PARM    @PEOPT       ← valor a validar (o ' ' para listar)
C     PARM    *BLANKS   @PEPGM
C     PARM    *BLANKS   @PEFMT
C     PARM    'SINO    ' @PEFLD    ← campo del diccionario a usar
C     PARM    'V'       @PEERR    ← modo: 'V'=validar, ' '=listar (F4)
```

---

## 10. Convenciones de Nomenclatura

### 10.1 Prefijos de módulo en nombres de programas

| Prefijo | Módulo | Tipo |
|---------|--------|------|
| `CM` | Compras — consulta/maestro | Interactivo |
| `CO` | Compras — proceso operativo | Interactivo / servicio |
| `PU` | Compras (Purchases) | Interactivo / servicio |
| `DO` | Distribución / DOM — configuración | Interactivo |
| `DOM` | Motor DOM — tablas de trabajo | DDS/PF/LF |
| `ED` | Entregas Directas / Motor DOM — proceso | Proceso / batch |
| `MS` | Maestros compartidos | DDS/PF + servicios |
| `PM` | Parametrización de empresa | Interactivo + DDS |
| `MM` | Material Management | Interactivo + DDS |
| `TR` | Transfer / transacciones de órdenes | DDS/PF/LF |
| `WR` | Working Records — copias de trabajo | DDS/PF/LF |
| `GX` | Gateway / Exchange logístico | Interactivo + DDS |
| `MI` | Inventory — inventario y balances | Proceso + DDS |
| `ZQ/ZA` | Specialized — gestión de riesgos, etc. | Servicio + DDS |
| `CM` | Configuration Management | Interactivo |
| `CF` | Configuration Facilities | DDS + servicio |
| `CC` | Customer Care | Servicio |
| `UT` | Utilities | Servicio |
| `DP` | Data Processing | Proceso batch |
| `OO` | Online Order | Servicio API |
| `AQ` | Aquapoint — legacy específico | Proceso |

### 10.2 Sufijos de versión/variante

| Sufijo | Significado |
|--------|-------------|
| `00`, `01`, `02`... | Versiones de un mismo fichero lógico (LF) |
| `CL` | Programa CL wrapper (punto de entrada batch) |
| `W` | Window/working variant — variante con ventana o fichero de trabajo |
| `@nnn` | Versión especial (ej: `PMDEOR@005`) |
| `10`, `20`... | Variantes alternativas de acceso |

### 10.3 Prefijos de variables de programa

| Prefijo | Tipo | Descripción |
|---------|------|-------------|
| `@P` | Parámetro | Parámetros de entrada/salida del programa (`@PCIAS`, `@PDLGA`...) |
| `@PE` | Parámetro externo | Parámetros para llamada a subprogramas |
| `ZX` | Variable trabajo | Variables internas (`ZXOPT`, `ZXCIAS`, `ZXCANT`...) |
| `CCC` | Campo de cabecera | Campos comunes no variables (nombre pgm, job, usuario) |
| `CS_` | Common Subroutine | Subrutinas reutilizables dentro del programa |
| `S5` | Fichero CODEPE | Campos de líneas de pedido de venta |
| `S0` | Fichero COCAPE | Campos de cabecera de pedido de venta |
| `C5` | Fichero PUDEPE | Campos de líneas de OC |
| `C0` | Fichero PUCAPE | Campos de cabecera de OC |
| `M5` | Fichero MSARTI00 | Campos del maestro de artículos |
| `M1` | Fichero MSSTOK | Campos del stock |
| `MI` | Fichero MSLOAD | Campos de almacenes logísticos |
| `PP` | Parámetros empresa | Campos del área extendida de PMEMPR |
| `T6` | Fichero EDPAR4 | Campos de prioridades de distribución |
| `T3` | Fichero EDPARM | Campos de parámetros DOM por delegación |
| `T4` | Fichero EDPARC | Campos de parámetros DOM por cliente |
| `T0` | Fichero EDTRLP | Campos de trazabilidad de línea |
| `T1` | Fichero EDTRAU | Campos de auditoría de eventos |
| `XA` | Fichero MSDDOM | Campos de matriz de delegaciones |

---

## 11. Gestión de Modificaciones y Trazabilidad del Código

### 11.1 Sistema de marcas en el código fuente

Cada modificación al código queda marcada con un código de 5 caracteres al inicio de la línea:

```rpg
C0031C    PARM    @PEDLGA         ← línea añadida por modificación C0031
FQ365F    COCAPE25  IF  E  ...    ← fichero añadido por FQ365
28099D    @NSCIAS  S   LIKE(...)  ← variable definida en modificación 28099
```

Las líneas marcadas con `*` = código eliminado pero conservado para referencia:
```rpg
MOVEXF*   CNRAZSOC  IF A E  ...  ← fichero eliminado por MOVEX (desactivado)
C0088*    @PEFLD    S   LIKE(...) ← definición antigua reemplazada
```

### 11.2 Cabecera de modificación estándar

```rpg
* DATA DE MODIFICACIÓ: DD/MM/AAAA  MARCA: XXXXX          *
* AUTOR: Nombre del autor                                  *
* DOCUMENTACIÓ: Descripción de la modificación            *
```

### 11.3 Historial cronológico de modificaciones relevantes

| Fecha | Marca | Autor | Impacto en el sistema |
|-------|-------|-------|-----------------------|
| 1997 | — | Dpto. Informática | Versiones originales: CM0085, CO0120, CO0140, CO2519 |
| 2004 | 09024 | TL | Refactoring CO2593 (eliminar PMPROM) |
| 2005 | C0031 | Carlos del Valle | Coenvíos con integridad 100% |
| 2005 | C0008 | Vicky Garcés | EDI internacional (pedidos PPED=99) |
| 2006 | MOVEX | Carlos del Valle | Integración Movex (parcialmente revertida) |
| 2006 | C0074 | — | Control compatibilidades ADR/Astralpool |
| 2007 | ECOP | — | Tratamiento automático de ecotasa |
| 2009 | FQ365 | Josep Aragonès | Control de riesgo crediticio (estado '1K', status '10') |
| 2010 | FQ284 | Carlos del Valle | Fluidra Direct: canal distribución directa |
| 2010 | FQ665 | — | CM0085: añadir importe a vistas de lista |
| 2011 | 3003A | — | Reservar al confirmar pedido (por setup) |
| 2011 | FQ799 | — | Auditoría de cambios en pedidos |
| 2012 | FQ937 | — | Ajuste fechas coenvíos (opción RP) |
| 2012 | 06B12 | — | Fecha confirmada = la más desfavorable de las líneas |
| 2013 | CR191 | JMARTINEZP | Corrección algoritmo días laborables (CO2611) |
| 2013 | FDX01 | Carlos del Valle | Fluidra Direct: CO2733 (situación conjuntos) |
| 2013 | C3758 | David Espuny | DOM: excepciones por artículo en DO0005 |
| 2014 | C1997 | jrgonzalez | Email confirmación pedido (UT2725) |
| 2020 | B0251 | Lluís Albert Grau | CO2593: soporte SalesForce API modo batch |
| 2021 | F-2OL | MMontana | CO2519/CO0120: abrir a más de un OL |
| 2022 | IT244 | David Grau | DO0001: 2ª fase multi-OL |
| 2023 | PG615 | David Grau | CM0085: retener pedidos por bloqueo albarán |
| 2023 | E7394 | — | CM0085: revertir PG615 para Italia |
| Nov-2025 | 08835 | Pere Vellet | CO2755: corrección MTS/MTO en Fluidra Direct (SCE-104) |
| Dic-2025 | PGE21 | JMARTINEZP + LAGrau | CO2519: auditoría digital en CODEPE |

---

## 12. Entornos de Ejecución y Aislamiento

### 12.1 Tipos de ejecución

| Tipo | Descripción | Identificación |
|------|-------------|----------------|
| **Interactivo** | Job iniciado por un usuario en terminal 5250 | `RTVTIJ` → `'1'` |
| **Batch** | Job lanzado por scheduler o API externa | `RTVTIJ` → `'0'` |
| **API/WS** | Llamada desde SalesForce u otros sistemas vía web service | Usuario = `QUSER` |

### 12.2 Flujo de inicialización de sesión

```
Usuario conecta (terminal 5250 o sesión)
   │
   ▼
LDA (Local Data Area) se carga con: ZZCIAS, ZZDLGA, ZZDEEM
   │
   ▼
PMLIBL recupera lista de librerías para la empresa/delegación
   │
   ▼
Programa principal lee PMLIBL para determinar librería de datos y programas
   │
   ▼
Programas operativos (CM0085, CO0120...) leen LDA con IN DSLDA
```

### 12.3 Aislamiento del motor DOM en QTEMP

```
ED3025CL (CL wrapper)
   │ CPYF COCAPE → QTEMP/COCAPE
   │ CPYF CODEPE → QTEMP/CODEPE
   │ CPYF CODEPE20 → QTEMP/CODEPE20
   │ (etc.)
   ▼
ED3025 (opera sobre copias en QTEMP — sin bloqueos)
   │
   ▼
Limpieza automática de QTEMP al finalizar el job
```

---

*Sistema AMS — Fluidra S.A. | Documentación Técnica | Versión 2.0 — Marzo 2026*
