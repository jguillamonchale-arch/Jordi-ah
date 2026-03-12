# Arquitectura Técnica del Sistema AMS
## Sistema de Gestión de Pedidos y Logística — Fluidra S.A.

---

## Índice

1. [Entorno Técnico](#1-entorno-técnico)
2. [Estructura de Programas](#2-estructura-de-programas)
3. [Patrón de Control de Flujo ZXOPT](#3-patrón-de-control-de-flujo-zxopt)
4. [Convenciones de Nomenclatura](#4-convenciones-de-nomenclatura)
5. [Arquitectura de Ficheros de Base de Datos](#5-arquitectura-de-ficheros-de-base-de-datos)
6. [Modelo de Datos Principal](#6-modelo-de-datos-principal)
7. [Integración entre Programas](#7-integración-entre-programas)
8. [Mecanismos Técnicos Transversales](#8-mecanismos-técnicos-transversales)
9. [Integraciones Externas](#9-integraciones-externas)
10. [Gestión de Modificaciones](#10-gestión-de-modificaciones)

---

## 1. Entorno Técnico

| Atributo | Valor |
|----------|-------|
| **Plataforma** | IBM i (AS/400) |
| **Lenguaje** | RPG IV (RPGLE) — compatible RPG de 1994 (RPGLE desde OS/400 V3R2) |
| **Sistema Operativo** | OS/400 / IBM i OS |
| **Formato de fecha** | `*YMD/` (especificador H: `DATEDIT(*YMD/)`) — formato YY/MM/DD en pantalla |
| **Codificación fuente** | UTF-8 con terminadores CRLF |
| **Acceso a BD** | DISK con acceso keyed (CHAIN/SETLL/READE) |
| **Pantallas** | Subfile (SFL) y formatos WORKSTN via Display Files (.FM) |
| **Compilación** | `H OPTION(*NODEBUGIO:*SRCSTMT)` — sin debug de I/O, con números de sentencia fuente |

---

## 2. Estructura de Programas

### 2.1 Clasificación por tipo

| Tipo | Descripción | Programas |
|------|-------------|-----------|
| **Programa interactivo con SFL** | Lista paginada de registros. Tiene formato WORKSTN con SFILE. | CM0085, CO0120, DO0001, DO0020 |
| **Programa interactivo panel** | Formulario de detalle (sin SFL). Formato WORKSTN simple. | CO2593, DO0005, DO0010, DO0025 |
| **Subprograma de servicio** | Sin pantalla. Invocado por CALL desde otro programa. | CO2611, CO2696, CO2733, CO2755, CO2610 |
| **Programa de lanzamiento** | Orquesta operaciones de base de datos. Puede ser interactivo o batch. | CO2519, CO0140 |

### 2.2 Estructura interna estándar de un programa

Todo programa sigue la siguiente estructura:

```
H - Header (especificaciones de control)
    DATEDIT(*YMD/)
    OPTION(*NODEBUGIO:*SRCSTMT)

F - File declarations (declaración de ficheros)
    - Fichero de pantalla (WORKSTN) si es interactivo
    - Ficheros de BD (DISK): IF=input, UF=update, O=output, A=append

D - Data definitions
    - SDS (Program Status Data Structure): info del programa, job, usuario
    - FMTDS / SFLDS: info del fichero de pantalla (posición cursor, SFL record)
    - DSLDA: área de datos local (LDA) de compras (PURLDADS/PURLDADS)
    - Parámetros de programa (@Pxxxx)
    - Variables de trabajo (ZXxxxx)
    - ZXOPT: control de flujo
    - Indicadores (INDDS)

C - Calculations (código principal)
    *INZSR — Subrutina de inicialización (define *ENTRY PLIST, recupera LDA)
    DOW ZXOPT1 <> 'END'
      CASEQ 'CLR'  SUBCLR  → Limpiar SFL
      CASEQ 'INZ'  SUBINZ  → Inicializar/cargar SFL
      CASEQ 'DSP'  SUBDSP  → Visualizar pantalla (EXFMT)
      CASEQ 'CMD'  SUBCMD  → Validar teclas de función
      CASEQ 'CHK'  SUBCHK  → Validar campos de pantalla
      CASEQ 'UPD'  SUBUPD  → Actualizar ficheros de BD
    ENDDO
    SETON LR

    [Subrutinas de proceso específicas]
    [CS_ Common Subroutines al final del programa]
```

---

## 3. Patrón de Control de Flujo ZXOPT

El patrón `ZXOPT` es el mecanismo central de control de flujo utilizado en todos los programas interactivos de este sistema.

### 3.1 Estructura

```rpg
D  ZXOPT          DS
D    ZXOPT1               1      3    ← Nombre del proceso
D    ZXOPT2               4      6    ← Rutina dentro del proceso
```

### 3.2 Valores de ZXOPT1

| Prefijo | Tipo de proceso | Descripción |
|---------|----------------|-------------|
| `Snn` | Subfile N | Proceso del subfile nº N (nn=01..99). N coincide con el nº de lógico del fichero y del formato de pantalla |
| `Pnn` | Pantalla N | Proceso de formato de pantalla N (formulario) |
| `Wnn` | Ventana N | Proceso de ventana/popup N |
| `END` | Fin | Salida del programa |

### 3.3 Valores de ZXOPT2 (rutinas)

| Valor | Rutina | Descripción |
|-------|--------|-------------|
| `CLR` | SUBCLR | Limpiar el subfile (borrar registros cargados) |
| `INZ` | SUBINZ | Inicializar y cargar el subfile/pantalla con datos |
| `DSP` | SUBDSP | Visualizar la pantalla (EXFMT) y esperar respuesta del usuario |
| `CMD` | SUBCMD | Validar las teclas de función pulsadas |
| `CHK` | SUBCHK | Validar los campos modificados en pantalla |
| `UPD` | SUBUPD | Aplicar los cambios a la base de datos |

### 3.4 Flujo típico de un ciclo de subfile

```
INZ → cargar registros en SFL
  ↓
DSP → presentar SFL al usuario (EXFMT)
  ↓
CMD → analizar tecla pulsada (F3/F12 → END, otras → continuar)
  ↓
CHK → validar campos modificados por el usuario
  ↓
UPD → grabar cambios en BD
  ↓
CLR → limpiar SFL para nueva carga
  ↓
INZ → (nuevo ciclo)
```

### 3.5 Variables auxiliares de subfile

| Variable | Descripción |
|----------|-------------|
| `ZXNNNnn` | Último nº de registro cargado en el subfile (posición para lecturas secuenciales) |
| `ZXNRRnn` | Numerador de registros del subfile (contador de registros cargados) |
| `ZXCOUNT` | ZXNRRnn del primer registro de la última página visualizada |
| `ZXLOPnn` | Numerador de registros en carga de pantalla |
| `SUFRCDN` | SFLRCDNBR del subfile (posición del cursor en la lista) |

---

## 4. Convenciones de Nomenclatura

### 4.1 Nombres de programas

| Patrón | Módulo | Ejemplo |
|--------|--------|---------|
| `CM-nnnn` | Compras — nivel maestro/consulta | CM0085 |
| `CO-nnnn` | Compras — proceso operativo | CO0120, CO0140, CO2519 |
| `DO-nnnn` | Distribución / Logística | DO0001, DO0005 |

### 4.2 Prefijos de variables

| Prefijo | Tipo | Descripción |
|---------|------|-------------|
| `@P` | Parámetro | Parámetros de entrada/salida del programa (`@PCIAS`, `@PDLGA`...) |
| `@PE` | Parámetro exportación | Parámetros de llamada a subprogramas |
| `ZX` | Variable de trabajo | Variables internas del programa (`ZXOPT`, `ZXCIAS`...) |
| `CCC` | Campo común de cabecera | Campos que no varían entre cabeceras de la aplicación (nombre del pgm, del job...) |
| `CS_` | Common Subroutine | Subrutinas comunes reutilizables dentro del programa |
| `S5`, `S0`, `S2` | Prefijo de fichero | Campos del fichero con ese prefijo de formato |
| `M5`, `M1`, `M6` | Prefijo de fichero | Campos de maestros de artículo/stock |
| `PP` | Parámetros empresa | Campos del área de parámetros extendidos de empresa |

### 4.3 Prefijos de campos de ficheros

Los campos en RPG siguen el prefijo del nombre del fichero o formato lógico:

| Fichero | Prefijo típico | Ejemplo |
|---------|---------------|---------|
| CODEPE | S5 | S5CIAS, S5ARTI, S5CANT |
| COCAPE | S0 | S0CIAS, S0NPCO, S0TPED |
| MSARTI00 | M5 | M5CIAS, M5ARTI, M5SIOL |
| MSSTOK | M1 | M1CIAS, M1MAGA, M1ARTI |
| MSLOAD | MI | MICIAS, MIDLGA, MILGCA |
| PMEMPR | PP | PPCIAS (a través de PPAMPL) |
| PMUNAL | P6 | P6CIAS, P6ARTI, P6COFA |
| EDPAR4 | (sin prefijo fijo) | TREG, MAGA |

### 4.4 Marcas de modificación

Cada línea modificada en el código lleva un prefijo de 5 caracteres que identifica la modificación:

```
C0031C    PARM      @PEDLGA        ← mod. C0031 afecta a esta línea
FQ365F    COCAPE25  IF   E  ...    ← mod. FQ365 añadió este fichero
28099D    @NSCIAS   S   LIKE(...)  ← mod. 28099 definió esta variable
```

Las marcas comentadas con `*` indican código eliminado o desactivado:
```
MOVEXF*CNRAZSOC  IF A E  ...      ← fichero comentado por mod. MOVEX
```

---

## 5. Arquitectura de Ficheros de Base de Datos

### 5.1 Módulo de Compras — Ficheros de Pedidos

```
COCAPE     ─── Cabecera de pedido de cliente (maestro)
           ├── COCAPE23  (lógico — acceso por cliente)
           ├── COCAPE25  (lógico — incluye pedidos status 10)
           ├── COCAPE38  (lógico — ordenación descendente por pedido)
           └── COCAPEA8  (lógico — pedidos activos, usado por KORE)

CODEPE     ─── Detalle de líneas de pedido (maestro)
           ├── CODEPE08  (lógico — para cálculo importes)
           └── CODEPE42  (lógico — acceso por artículo, usado por KORE)

CODIPE     ─── Datos complementarios de línea de pedido
COOBPE     ─── Observaciones de pedido
COPECO     ─── Condiciones de pedido
COGAPE     ─── Agrupaciones de pedido
COCAIM     ─── Cabecera de importes del pedido
CODEP2     ─── Detalle pedido vista 2
CMA040     ─── Datos adicionales de pedido
           └── CMA04024, CMA04034  (lógicos)
COCLIN     ─── Maestro de clientes de pedido
```

### 5.2 Módulo de Logística — Ficheros DOM

```
EDPAR4     ─── Prioridades de distribución DOM (maestro principal)
EDPAR2     ─── Datos secundarios de prioridades
EDPARM     ─── Parámetros DOM nivel empresa
EDHPA4     ─── Histórico de cambios en EDPAR4
```

### 5.3 Maestros Compartidos

```
MSARTI00   ─── Maestro de artículos
MSSTOK     ─── Stock por almacén
MSSTST     ─── Stock estadístico
MSLOAD     ─── Almacenes logísticos (empresa+delegación+almacén lógico)
MSALMA     ─── Maestro de almacenes físicos
MSALMA05   ─── Lógico de almacenes (filtro específico)
MSALMA08   ─── Lógico de almacenes (Fluido Direct)
MSCOJC     ─── Cabecera de conjuntos de artículos
MSCOJD     ─── Detalle de conjuntos (componentes)
MSFAMI     ─── Maestro de familias de artículos
MSSGGA     ─── Segmentos/grupos de almacén
MSIVAS     ─── IVA/impuestos de artículos
MSDIV0     ─── Divisas
```

### 5.4 Ficheros de Parámetros y Maestros de Sistema

```
PMEMPR     ─── Parámetros de empresa (central de configuración)
PMLIBL     ─── Parámetros de biblioteca
PMTABL     ─── Tablas de parámetros (maestro)
PMTABD     ─── Detalle de tablas de parámetros
PMTAB2     ─── Tablas secundarias
PMSETP     ─── Setup general del sistema
PMUNAL     ─── Unidades de medida y factores de conversión
PUPROV     ─── Maestro de proveedores
ZQGSRK00   ─── Gestión de riesgos (scoring de riesgo)
ZQGCSC00   ─── Gestor asignado al cliente
```

### 5.5 Relaciones entre ficheros (modelo conceptual)

```
PMEMPR (empresa)
    └── define parámetros para →  EDPAR4 (prioridades DOM)
                                  MSARTI00 (clasificación artículos)

COCAPE (cabecera pedido)
    ├── 1:N → CODEPE (líneas de pedido)
    ├── N:1 → COCLIN (cliente)
    └── N:1 → MSLOAD (almacén asignado)

CODEPE (línea de pedido)
    ├── N:1 → MSARTI00 (artículo)
    ├── N:1 → MSSTOK (stock disponible)
    └── puede pertenecer a un conjunto → MSCOJC/MSCOJD
```

---

## 6. Modelo de Datos Principal

### 6.1 Estructura de la clave de pedido

Un pedido se identifica completamente por:

```
CIAS (3 num) + DLGA (5 alfa) + NPCO (num) + TPED (2 alfa)
```

Una línea de pedido se identifica por:
```
CIAS + DLGA + NPCO + TPED + DATP + NOPL
```

Donde:
- `DATP` = fecha del pedido
- `NOPL` = número de línea (PEHORA — timestamp que actúa como número de línea)

### 6.2 Tabla de numeración de órdenes (NPVE / PMTABD)

La numeración de órdenes de venta sigue la clave compuesta:

```
Clave de 10 caracteres en tabla PECLTA:
  Posiciones 1-3:   CIAS
  Posiciones 4-8:   DLGA
  Posiciones 9-10:  TPED
```

Búsqueda en cascada (de más específica a más general):
```
1. CIAS + DLGA + TPED
2. CIAS + DLGA + '  '
3. CIAS + '     ' + TPED
4. CIAS + '     ' + '  '
```

### 6.3 Estructura de prioridades DOM (EDPAR4)

Clave primaria de EDPAR4:
```
CIAS + MAGA + TREG + [criterio_específico_según_TREG]
```

Donde el criterio específico varía:
- TREG=0/A: sin criterio adicional (prioridad general)
- TREG=1/B: código de proveedor
- TREG=2/C: artículo + variante + modelo
- TREG=3/D: familia + subfamilia
- TREG=4/F: familia + tarifa
- TREG=5/G: código de cliente

---

## 7. Integración entre Programas

### 7.1 Mapa de llamadas (CALL)

```
CO0120 (cabecera pedido)
  ├── CALL CO0140          ← entrada detalle de líneas
  ├── CALL CO2519          ← lanzamiento de orden
  ├── CALL CO2593          ← parámetros de impresión
  ├── CALL CO2696          ← validación coenvío (antes de expedir)
  ├── CALL CO2755          ← categoría de rotación del artículo
  ├── CALL CO2611          ← cálculo de fecha de entrega (días laborables)
  ├── CALL UT2725          ← envío de mail de confirmación (mod. C1997)
  ├── CALL MM2559          ← verificación de impresora
  └── CALL MMPROM          ← búsqueda/validación en diccionario (F4)

CO2519 (lanzamiento orden)
  ├── CALL CO2755          ← categoría de rotación (para clasificación MTS/MTO)
  └── CALL UT2725          ← envío de mail

CM0085 (lista pedidos)
  └── CALL CO0120          ← al seleccionar un pedido de la lista

DO0001/DO0020 (lista prioridades)
  └── CALL DO0005/DO0025   ← al seleccionar opción de detalle

CO2610 (KORE — pedidos pendientes)
  ├── Invocado por motor KORE externo
  └── CALL @GETWEEK        ← (interno: no aplica a CO2610)

CO2611 (días laborables)
  └── CALL @GETWEEK        ← obtiene día de la semana de una fecha
```

### 7.2 Parámetros de LDA — PURLDADS

Todos los programas interactivos recuperan el área de datos local (LDA) de compras al inicio:

```rpg
C     *DTAARA    DEFINE    PURLDA    DSLDA
C                IN        DSLDA
```

La estructura `PURLDADS` contiene contexto de trabajo compartido:
- `ZZCIAS` — Empresa activa
- `ZZDLGA` — Delegación activa
- `ZZDEEM` — Delegación del empleado (alias sobre posición 30 de DSLDA)
- Otros campos de configuración del entorno de trabajo

### 7.3 Estructura de parámetros de programa (DS externa)

Los programas con múltiples formatos de pantalla usan estructuras de datos externas para pasar parámetros entre invocaciones:

```rpg
D  CM0085      E DS      EXTNAME(CM0085DS)   ← parámetros del pgm CM0085
D  DO0001      E DS      EXTNAME(DO0001DS)   ← parámetros del pgm DO0001
D  CO0120      E DS      EXTNAME(CO0120DS)   ← parámetros del pgm CO0120
```

Esto permite que un programa invocador pase el contexto completo al programa invocado sin enumerar cada parámetro individualmente.

---

## 8. Mecanismos Técnicos Transversales

### 8.1 Gestión de cursor en pantalla

```rpg
D  FMTDS      DS
D    FMTPOCUR       370   371B 0    ← posición del cursor (fila*256 + columna)
D    FMTPOS         378   379B 0    ← posición alternativa
D    FMTRCD     *RECORD             ← nombre del último formato activo

C     FMTPOCUR  DIV   256   CSRFILA    ← extrae fila
C               MVR         CSRCOLU    ← extrae columna (resto)
```

Esta estructura está disponible a través del INFDS del fichero de pantalla.

### 8.2 Indicadores como estructura de datos

Los indicadores del sistema se mapean sobre una estructura de datos basada en puntero:

```rpg
D  INDICADORS  E DS         BASED(INDICADORP)
D                            EXTNAME(INDDS:RIND0199)
D  INDICADORP  S    *        INZ(%ADDR(*IN))
```

Permite referenciar los indicadores por nombre en lugar de por número:
- `ERRGRL` → indicador de error general
- `EXIT`, `RETRO`, `HELP` → indicadores de teclas de función
- `WRKnn` → indicadores de trabajo (nn=01..99)
- `SETLnn`, `READnn` → indicadores de posicionamiento/lectura

### 8.3 Conversión de unidades de medida (subrutina CS_CNVUBS)

Presente en CO2610. Convierte una cantidad de la unidad de medida del pedido a la unidad base del artículo:

```
Algoritmo:
1. Lee PMUNAL con clave: CIAS + ARTI + VARI + MODE + UNIM_ORIGEN
2. Si no encuentra → reintenta con VARI=blancos, MODE=blancos
3. Si P6TCOF <> '1': resultado = cantidad × P6COFA  (factor multiplicador)
4. Si P6TCOF = '1': resultado = cantidad / P6COFA   (factor divisor)
5. Aplica redondeo/ajuste decimal (copia incluida de SRD001)
```

### 8.4 Detección de tipo de trabajo (interactivo vs batch)

Presente en CO2593 (mod. B0251). Se llama al programa utilitario `RTVTIJ`:

```rpg
C     CALL    'RTVTIJ'
C     PARM    *BLANKS    @PTIPJ    ← '0'=batch, '1'=interactivo
```

Si `@PTIPJ='0'` (batch/QUSER/API) → no se abre la pantalla y se usa valor por defecto.

### 8.5 Validación mediante diccionario MMPROM

Para campos con lista de valores válidos (tipo Sí/No, estado, etc.), se valida mediante:

```rpg
C     CALL    'MMPROM'
C     PARM    @PEOPT       ← valor a validar (o ' ' para F4)
C     PARM    *BLANKS   @PEPGM
C     PARM    *BLANKS   @PEFMT
C     PARM    'SINO    ' @PEFLD    ← campo del diccionario a usar
C     PARM    'V'       @PEERR    ← modo: 'V'=validar, ' '=listar (F4)
```

Retorna:
- `@PEERR=' '` → valor válido
- `@PEERR>'0'` → valor inválido o error
- `@PEERR='A'` → usuario canceló la selección en F4

### 8.6 Verificación de dispositivo de impresora (MM2559)

```rpg
C     CALL    'MM2559'
C     PARM    @PCIAS
C     PARM    @PDLGA
C     PARM    SDSPGM    @PPROG
C     PARM              @PRCDR    ← nombre del recurso impresora
C     PARM    'FMTO  '  @PFIEL    ← campo a verificar
C     PARM    '01'      @POPCI    ← opción
C     PARM    'LCM0035' @PCCIT    ← título para mensajes
C     PARM              @PJOBT
C     PARM              @PPRG2
C     PARM    '1'       @PPAR1
C     PARM              @PPAR2
C     PARM    '1'       @PEERR    ← resultado: ' '=OK, 'A'=cancelado, >0=error
```

### 8.7 MONITOR / ON-ERROR

Presente en CO2611 para captura de errores en conversión de fechas:

```rpg
C     MONITOR
C       MOVE  @PEDAT1   ZXFECHA_0    ← conversión que puede fallar
C     ON-ERROR
C       EVAL  @PEERRO='1'            ← captura el error y señaliza
C       GOTO  SIG
C     ENDMON
```

---

## 9. Integraciones Externas

### 9.1 SalesForce API

- **Programa:** CO2593 (PURLIBC8)
- **Integración:** Cuando SalesForce llama al programa vía web service, el trabajo se ejecuta como QUSER (batch)
- **Mecanismo:** `RTVTIJ` detecta el tipo de trabajo → CO2593 no abre pantalla → usa parámetros recibidos directamente
- **Referencia:** Modificación BD-251 (02/06/2020, Lluís Albert Grau)

### 9.2 Motor KORE

- **Programa:** CO2610 (referenciado como CO2770 en la librería)
- **Integración:** El motor KORE llama a CO2610 para obtener las cantidades pendientes de servir por artículo
- **Parámetros clave:** CIAS, DLGA, artículo, rango de fechas
- **Resultado:** Cantidad total pendiente en unidad base del artículo

### 9.3 Motor DOM (Distribution Order Management)

- **Programas:** DO0001, DO0005, DO0010, DO0020, DO0025
- **Integración:** Los programas DO\* mantienen las tablas EDPAR4 que el motor DOM lee en tiempo de ejecución para decidir el routing de órdenes
- **Fichero central:** EDPAR4 (prioridades de distribución)

### 9.4 Sistema M3 / Movex

- **Programas:** CO2519, CO0120
- **Integración:** Gestión de coenvíos y albaranes a terceros en coordinación con Movex
- **Referencia:** Modificación MOVEX (Agosto 2006) — anulación CNRAZSOC

### 9.5 EDI Internacional

- **Programas:** CO0120, CO0140
- **Integración:** Pedidos EDI cargados previamente en COCPED con PPED=99
- **Referencia:** Modificación C0008 (Vicky Garcés, 17/10/2005)

### 9.6 Servicio de email (UT2725)

- **Programa:** CO2519, CO0120
- **Integración:** Tras confirmar/lanzar un pedido, se envía mail de confirmación al cliente
- **Referencia:** Modificación C1997 (jrgonzalez, 24/09/2014)

---

## 10. Gestión de Modificaciones

### 10.1 Sistema de marcas

El código fuente implementa un sistema de trazabilidad de modificaciones mediante **marcas** de 5 caracteres al inicio de cada línea:

```
MARCA LLÍNEA_DE_CÓDIGO
```

Ejemplos:
- `FQ365` — Modificación realizada el día 23/03/2009 (FQ=número interno, 365=día del año aprox.)
- `C0031` — Modificación C0031 (código de cambio)
- `B0251` — Modificación BD-251
- `CR191` — Change Request 191
- `08835` — Modificación del 08/8/35 (formato alternativo)
- `PGE21` — Modificación PGE-21

Código comentado con `*` en la marca = código eliminado que se conserva para referencia:
```
C0088 * @PEFLD   S   LIKE(PMFIEL)   ← antigua declaración (eliminada)
C0088D  @PEFLD   S   LIKE(FIELD)    ← nueva declaración
```

### 10.2 Cabecera de modificación estándar

```rpg
* DATA DE MODIFICACIÓ: DD/MM/AAAA  MARCA: XXXXX    *
* AUTOR:  Nombre del autor                          *
* DOCUMENTACIÓ: Descripción de la modificación      *
```

O en formato largo:
```rpg
* Modificación: XXXXX              *
*        Autor: Nombre             *
* Fecha Inicio: DD/MM/AAAA         *
*  Descripción: Descripción        *
```

### 10.3 Historial cronológico de modificaciones relevantes

| Fecha | Marca | Autor | Impacto |
|-------|-------|-------|---------|
| 1997 | — | Dpto. Informática | Versiones originales CM0085, CO0120, CO0140, CO2519 |
| 2004 | 09024 | TL | Eliminar fichero PMPROM de CO2593 |
| 2005 | C0031 | Carlos del Valle | Coenvíos con integridad 100% |
| 2005 | C0008 | Vicky Garcés | EDI internacional |
| 2006 | MOVEX | Carlos del Valle | Integración Movex |
| 2009 | FQ365 | Josep Aragonès | Control de riesgo crediticio |
| 2010 | FQ284 | Carlos del Valle | Fluidra Direct |
| 2010 | 01090 | X.BOU | CO2755: eliminar LDA, usar PMEMPR |
| 2013 | CR191 | JMARTINEZP | CO2611: corrección algoritmo días laborables |
| 2013 | FDX01 | Carlos del Valle | CO2733: Fluido Direct |
| 2014 | C1997 | jrgonzalez | Email confirmación pedido |
| 2017 | C3758 | David Espuny | DO0005: excepciones DOM por artículo |
| 2020 | B0251 | Lluís Albert Grau | CO2593: soporte SalesForce API batch |
| 2021 | F-2OL | MMontana | CO2519: abrir a más de un OL |
| 2022 | IT244 | David Grau | DO0001: 2ª fase 2OL |
| 2023 | PG615 | David Grau | CM0085: retener pedidos por bloqueo albarán |
| 25/11/2025 | 08835 | Pere Vellet | CO2755: corrección MTS/MTO en Fluido Direct (SCE-104) |
| 15/12/2025 | PGE21 | JMARTINEZP / Lluís Albert Grau | CO2519: auditoría digital CODEPE |

---

*Documentación generada a partir del código fuente. Versión 1.0 — Marzo 2026.*
