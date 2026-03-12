# Motor DOM / DOOM — Análisis en Profundidad
## Sistema AMS — Fluidra S.A.

**Versión:** 1.0 — Marzo 2026
**Audiencia:** Equipo técnico senior, equipo funcional clave, AMS externo
**Clasificación:** Interno — Sensible

---

## Índice

1. [¿Qué es el Motor DOM?](#1-qué-es-el-motor-dom)
2. [Arquitectura del Motor DOM](#2-arquitectura-del-motor-dom)
3. [El Algoritmo DOOM — Cubo de Decisión 3D](#3-el-algoritmo-doom--cubo-de-decisión-3d)
4. [Tablas Maestras del DOM](#4-tablas-maestras-del-dom)
5. [Árboles de Decisión](#5-árboles-de-decisión)
6. [Flujo Completo de un Pedido por el DOM](#6-flujo-completo-de-un-pedido-por-el-dom)
7. [Generación de Mensajes EDI tras el DOM](#7-generación-de-mensajes-edi-tras-el-dom)
8. [Trazabilidad y Auditoría](#8-trazabilidad-y-auditoría)
9. [Configuración del DOM — Guía Operativa](#9-configuración-del-dom--guía-operativa)
10. [Casos Especiales y Excepciones](#10-casos-especiales-y-excepciones)
11. [Diagnóstico y Resolución de Problemas](#11-diagnóstico-y-resolución-de-problemas)
12. [Programas del Motor DOM — Referencia Rápida](#12-programas-del-motor-dom--referencia-rápida)

---

## 1. ¿Qué es el Motor DOM?

### 1.1 Definición

El **Motor DOM** (Distribution Order Management) es el sistema de inteligencia de negocio que decide, de forma automática y en tiempo real, **desde qué almacén o proveedor se servirá cada línea de un pedido de cliente**.

Cuando un pedido de venta se lanza en el sistema, el Motor DOM:
1. Analiza cada línea del pedido
2. Evalúa todos los almacenes y proveedores posibles como fuente de suministro
3. Aplica un conjunto de reglas de negocio, prioridades y restricciones
4. Selecciona el almacén/proveedor óptimo para cada línea
5. Actualiza el pedido con el código logístico asignado (campo `S5LOGE`)
6. Genera los mensajes EDI al proveedor seleccionado

### 1.2 Conexión entre Ventas, Compras y Stock

El DOM es el **nexo central** que conecta los tres pilares del ciclo de suministro:

```
VENTAS (COCAPE/CODEPE)
      │ "¿Qué necesita el cliente?"
      │ Pedido con artículos, cantidades, fechas
      ▼
MOTOR DOM (ED3025/DOOM/ED0050)
      │ "¿Quién puede servir esto mejor?"
      │ Evalúa: stock + acuerdos + reglas + capacidad
      ├────────────────────┐
      │                    │
      ▼                    ▼
STOCK (MSSTOK)        COMPRAS (PUCAPE/PUDEPE)
"Hay stock en         "Hay que comprar a
almacén propio"       proveedor externo"
      │                    │
      └────────────────────┘
                │
                ▼
      EDI (ORDERS → Proveedor)
      "Ejecutar el aprovisionamiento"
```

### 1.3 El concepto de "Entrega Directa" (ED / Fluido Direct)

Una **Entrega Directa** es el modo de funcionamiento en el que el proveedor envía directamente al cliente de Fluidra, **sin que el producto pase por un almacén de Fluidra**. El DOM puede decidir que una línea se sirva mediante ED si:
- El artículo no tiene stock en ningún almacén propio
- Un proveedor está configurado para hacer ED
- Las reglas de prioridad favorecen la ED para ese artículo/cliente

**Fluido Direct** es el canal de distribución directa desde la empresa productiva de Fluidra (empresa productora → cliente final), una variante específica de la ED.

---

## 2. Arquitectura del Motor DOM

### 2.1 Programas principales

```
┌──────────────────────────────────────────────────────────────────────┐
│                   ARQUITECTURA DEL MOTOR DOM                         │
│                                                                      │
│  ED3025CL ────────────────────────────────────────────────────────┐  │
│  (CL Wrapper)                                                     │  │
│      │ Copia ficheros a QTEMP                                     │  │
│      │ Llama ED3025                                               │  │
│      ▼                                                            │  │
│  ED3025 (Orquestador)                                             │  │
│  ┌────────────────────────────────────────────────────────────┐  │  │
│  │                                                            │  │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐ │  │  │
│  │  │ ED3050   │   │ ED3061   │   │ ED0060   │   │ ED0100 │ │  │  │
│  │  │ DOOM     │──▶│ Confirm. │──▶│ EDI Gen. │──▶│ Audit  │ │  │  │
│  │  │ Algorithm│   │ Screen   │   │          │   │        │ │  │  │
│  │  └────┬─────┘   └──────────┘   └──────────┘   └────────┘ │  │  │
│  │       │                                                    │  │  │
│  │       ▼                                                    │  │  │
│  │  DOOM01 / ED0050                                           │  │  │
│  │  (Core Decision Engine)                                    │  │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │  │
│  │  │  Lee: COCAPE, CODEPE, MSALMA, MSSTOK                 │ │  │  │
│  │  │       EDPAR2, EDPAR3, EDPAR4, EDPARC, EDPARM         │ │  │  │
│  │  │       MSDDOM(11-14), PUPROV, MSTRTA, MSCOJC/D        │ │  │  │
│  │  │                                                      │ │  │  │
│  │  │  Escribe: CODEPE.S5LOGE (almacén asignado)          │ │  │  │
│  │  │           DOOMHT (histórico decisión)                │ │  │  │
│  │  │           DO0050W-053W (ficheros trabajo QTEMP)      │ │  │  │
│  │  └──────────────────────────────────────────────────────┘ │  │  │
│  │                                                            │  │  │
│  └────────────────────────────────────────────────────────────┘  │  │
│                                                                   │  │
│  Limpieza QTEMP ◄─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Descripción de cada programa

| Programa | Tipo | Líneas | Función |
|----------|------|--------|---------|
| `ED3025CL` | CL Wrapper | ~30 | Punto de entrada batch. Copia ficheros a QTEMP. Llama ED3025. |
| `ED3025` | Orquestador RPG | ~237 | Gestiona el flujo completo: decide cuándo llamar a DOOM, presenta confirmación, llama EDI y auditoría |
| `ED3050` | Algoritmo DOOM | ~3.390 | Implementación completa del algoritmo de selección de almacén. Lee todos los datos, aplica todas las reglas, actualiza CODEPE |
| `DOOM01` | Algoritmo núcleo | ~5.187 | Núcleo del DOOM: cubo de decisión 3D. Alternativa/complemento a ED3050 |
| `ED3061` | Pantalla | — | Pantalla de confirmación del resultado DOM |
| `ED3062` | Pantalla | — | Detalle por línea del resultado DOM |
| `ED3063` | Pantalla | — | Pantalla de excepciones y errores DOM |
| `ED0060` | Proceso | — | Generación de mensajes EDI tras confirmación |
| `ED0100` | Proceso | — | Trazabilidad y auditoría del proceso |
| `ED0015` | Proceso | ~100 | Sincronización eventos ventas ↔ compras |
| `ED0025` | Proceso | — | Proceso auxiliar DOM |
| `ED0051` | Proceso | — | Proceso auxiliar DOM |
| `ED0053` | Proceso | — | Proceso auxiliar DOM |
| `EDINCI` | Proceso | — | Inicialización del proceso DOM |
| `EDINCI30` | Proceso | — | Variante de inicialización |

---

## 3. El Algoritmo DOOM — Cubo de Decisión 3D

### 3.1 Concepto del Cubo 3D

El algoritmo DOOM evalúa la asignación de almacén mediante lo que internamente se puede conceptualizar como un **cubo de tres dimensiones**:

```
      VALIDACIONES
      (reglas de negocio)
          │
          │
          ├─────────────────────────
         /                          \
        /    ALMACENES               \
       /     (opciones disponibles)   \
      /                               \
     ─────────────────────────────────
          ARTÍCULOS
          (líneas del pedido)
```

Para **cada artículo** (dimensión Z) del pedido, se evalúa **cada almacén** (dimensión Y) posible, aplicando **todas las validaciones** (dimensión X) de las reglas de negocio.

### 3.2 Entrada del algoritmo

El algoritmo DOOM recibe como entrada:
- El pedido de venta completo (`COCAPE` + `CODEPE`)
- El contexto de empresa/delegación
- Los parámetros de configuración DOM (`EDPAR2/3/4/C/M`)

### 3.3 Carga inicial de datos

```
PASO 1: Cargar cabecera del pedido (COCAPE)
   └─ Cliente, delegación, tipo de pedido, fechas

PASO 2: Cargar líneas del pedido (CODEPE)
   └─ Artículo, variante, modelo, cantidad, unidad

PASO 3: Cargar almacenes elegibles (MSALMA + MSLOAD)
   └─ Todos los almacenes activos de la empresa
      Filtrar por delegación (EDPARM.T3DLGA)

PASO 4: Cargar artículos (MSARTI00)
   └─ Datos del maestro: SIOL, TCOJ, familia, unidad base

PASO 5: Cargar stock (MSSTOK)
   └─ Disponibilidad actual por almacén/artículo

PASO 6: Cargar parámetros DOM
   ├─ EDPARM  — parámetros por delegación
   ├─ EDPARC  — parámetros por cliente
   ├─ EDPAR2  — parámetros por almacén (horizonte, capacidad)
   ├─ EDPAR3  — restricciones de capacidad física
   ├─ EDPAR4  — prioridades específicas (proveedor/artículo/familia/cliente)
   └─ MSDDOM  — matriz de delegaciones origen-destino

PASO 7: Cargar proveedores y transporte
   ├─ PUPROV  — maestro de proveedores
   └─ MSTRTA  — tipos de transporte/carrier

PASO 8: Cargar kits/conjuntos (si aplica)
   ├─ MSCOJC  — cabecera del conjunto
   └─ MSCOJD  — componentes del conjunto
```

### 3.4 Tiers de validación (en orden de evaluación)

Para cada par (artículo, almacén), el algoritmo evalúa los siguientes tiers de validación en orden:

#### Tier 1: Artículo de Servicio → Bypass total
```
SI MSARTI00.M5TIPO = 'SERV' O M5SIOL = 'S'
  → Omitir DOOM para este artículo
  → No se asigna almacén DOM
  → Se gestiona manualmente
FIN SI
```

#### Tier 2: Proveedor no habilitado para ED → Bypass total
```
SI EDPAR4 no tiene registro para este proveedor
  → Omitir DOOM para este artículo
  → No hay acuerdo de Entrega Directa
  → Se gestiona como pedido normal
FIN SI
```

#### Tier 3: Existencia del artículo en almacén
```
CHAIN MSSTOK con clave: CIAS + MAGA + ARTI + VARI + MODE
SI NO ENCONTRADO
  → Almacén INELEGIBLE para este artículo
  → Continuar con siguiente almacén
FIN SI
```

#### Tier 4: Disponibilidad de stock
```
SI MSSTOK.M1DISP >= cantidad_pedida
  → Almacén ELEGIBLE por stock suficiente
SINO
  SI MSSTOK.M1DISP > 0 (stock parcial)
    → Almacén PARCIALMENTE ELEGIBLE (registrar para análisis)
  SINO
    → Almacén INELEGIBLE por stock insuficiente
  FIN SI
FIN SI
```

#### Tier 5: Check de obsolescencia (OBSO)
```
SI EDPARM.T3OBSO = '1' O EDPARC.T4OBSO = '1'
  → Verificar si el artículo tiene flag de obsolescencia
  SI MSARTI00.M5OBSO = '1' Y almacén no está en MSDDOM12 (prioridad OC)
    → Almacén INELEGIBLE por política de obsolescencia
  FIN SI
FIN SI
```

#### Tier 6: Check de sobrestock (SSTO)
```
SI EDPARM.T3SSTO = '1' O EDPARC.T4SSTO = '1'
  → Verificar si hay sobrestock en este almacén
  SI stock > umbral_sobrestock Y almacén no está en MSDDOM13 (prioridad SSTO)
    → Almacén INELEGIBLE por política de sobrestock
  FIN SI
FIN SI
```

#### Tier 7: Acuerdo logístico proveedor-almacén (EDPAR4)
```
CHAIN EDPAR4 con clave: CIAS + MAGA + TREG + [criterio]
SI NO ENCONTRADO
  → No hay acuerdo logístico para este par proveedor/almacén
  → Almacén INELEGIBLE logísticamente
FIN SI
```

#### Tier 8: Restricciones de capacidad física (EDPAR3)
```
SI total_peso_OC > EDPAR3.T5PESO
  → Almacén INELEGIBLE por exceso de peso
FIN SI
SI total_volumen_OC > EDPAR3.T5VOLU
  → Almacén INELEGIBLE por exceso de volumen
FIN SI
SI total_importe_OC < EDPAR3.T5IMPO (importe mínimo)
  → Almacén INELEGIBLE por no alcanzar el importe mínimo
FIN SI
SI EDPAR3.T5MLIA >= EDPAR3.T5MLIN (max líneas)
  → Almacén INELEGIBLE por límite de líneas alcanzado
FIN SI
```

#### Tier 9: Restricciones de días de envío (MSDDOM)
```
CHAIN MSDDOM con clave: CIAS + DLGA_ORIGEN + DLGA_DESTINO
SI ENCONTRADO
  día_semana_hoy = @GETWEEK(fecha_actual)
  SI MSDDOM.XADI[día_semana_hoy] = '0'
    → No está permitido enviar hoy desde este almacén
    → Almacén INELEGIBLE por calendario
  FIN SI
FIN SI
```

#### Tier 10: Horizonte de planificación
```
CHAIN EDPAR2 con clave: CIAS + MAGA
SI ENCONTRADO
  SI fecha_solicitada < fecha_actual + EDPAR2.T4HORZ
    → El artículo se necesita antes del horizonte mínimo
    → Almacén INELEGIBLE por horizonte de planificación
  FIN SI
FIN SI
```

### 3.5 Scoring y selección del almacén óptimo

Una vez evaluados todos los tiers, para los almacenes que han pasado todas las validaciones:

```
PRIORIDAD 1: Almacén con excepción explícita en EDPAR4 (TREG=2 artículo, TREG=C DLGA/artículo)
   └─ Tiene prioridad máxima — override de toda la lógica general

PRIORIDAD 2: Almacén configurado en MSDDOM con mayor prioridad (XAPRIS)
   └─ Primer almacén en la lista de prioridades de la delegación

PRIORIDAD 3: Almacén con mayor disponibilidad de stock (M1DISP DESC)
   └─ Desempate por disponibilidad

PRIORIDAD 4: Almacén con menor tiempo de entrega (EDPAR2.T4HORZ ASC)
   └─ Desempate por rapidez

SELECCIÓN FINAL: primer almacén que supera todos los tiers y tiene mayor prioridad
```

### 3.6 Actualización del resultado

Tras la selección:

```rpg
* Actualizar CODEPE con el almacén seleccionado
CHAIN CODEPE con clave del pedido + línea
EVAL  S5LOGE = almacen_seleccionado    ← RESULTADO DOM
EVAL  S5PROV = proveedor_seleccionado  ← proveedor asignado
UPDATE CODEPE

* Registrar en histórico DOOMHT
WRITE DOOMHT con:
  - CIAS, DLGA, NPCO, TPED, fecha/hora
  - almacén y proveedor seleccionados
  - todos los datos de la decisión (DODATA)
  - usuario y programa (DOUSUA, DOPGMR)
```

---

## 4. Tablas Maestras del DOM

### 4.1 EDPAR4 — Prioridades de Distribución (fichero principal)

**El fichero maestro del DOM.** Cada registro define una regla de prioridad de distribución.

**Estructura de clave:**
```
T6CIA1 (empresa) + T6MAGA (almacén) + T6TREG (tipo regla) + [criterio variable]
```

Donde el criterio variable depende de `T6TREG`:

| TREG | Criterio variable | Descripción |
|------|-----------------|-------------|
| `0` | — | Prioridad general de empresa |
| `1` | `T6PRVR` (proveedor) | Por proveedor específico |
| `2` | `T6ARVT + variante + modelo` | Por artículo específico |
| `3` | `T6FAMT` (familia/subfam) | Por familia de artículo |
| `4` | `T6FAMT` (familia/tarifa) | Por familia/tarifa |
| `5` | Código cliente | Por cliente específico |
| `A` | `T6DLGA` (delegación) | Prioridad general delegación |
| `B` | `T6DLGA + T6PRVR` | Por delegación y proveedor |
| `C` | `T6DLGA + T6ARVT` | Por delegación y artículo |
| `D` | `T6DLGA + T6FAMT` | Por delegación y familia |
| `E` | `T6DLGA + T6FAMT` | Por delegación, fam y subfam |
| `F` | `T6DLGA + T6FAMT` | Por delegación, fam y tarifa |
| `G` | `T6DLGA + cliente` | Por delegación y cliente |
| `H` | `T6DLGA + proc.` | Por delegación y procedimiento |

**Campos de datos principales:**

| Campo | Descripción |
|-------|-------------|
| `T6CIA2` | Empresa destino |
| `T6MAGA` | Almacén destino |
| `T6PRIO` | Prioridad nivel 1 |
| `T6PRI2` | Prioridad nivel 2 |
| `T6OIMP` | Omitir importe mínimo en ED forzada |
| `T6OCAP` | Omitir check de capacidad |
| `T6COEN` | Forzar coenvío |
| `T6DFTO` | Código de almacén por departamento |
| `T6CSTO` | Código de C.STORAGE |
| `T6DAT1` / `T6DAT8` | Fecha de creación / modificación |
| `T6USUA` | Usuario de última modificación |
| `T6PANT` | Pantalla de última modificación |

**Jerarquía de búsqueda en EDPAR4 (de más específica a menos):**

```
1. CIAS + MAGA + TREG=C + DLGA + ARTÍCULO  (nivel DLGA + artículo)
2. CIAS + MAGA + TREG=2 + ARTÍCULO          (nivel empresa + artículo)
3. CIAS + MAGA + TREG=B + DLGA + PROVEEDOR  (nivel DLGA + proveedor)
4. CIAS + MAGA + TREG=1 + PROVEEDOR         (nivel empresa + proveedor)
5. CIAS + MAGA + TREG=E + DLGA + FAMILIA    (nivel DLGA + fam+subfam)
6. CIAS + MAGA + TREG=D + DLGA + FAMILIA    (nivel DLGA + familia)
7. CIAS + MAGA + TREG=3 + FAMILIA           (nivel empresa + familia)
8. CIAS + MAGA + TREG=A + DLGA              (nivel DLGA general)
9. CIAS + MAGA + TREG=0                     (nivel empresa general)
```

### 4.2 EDPAR2 — Parámetros por Almacén

Configuración logística básica por almacén. **Controla qué almacenes son candidatos y con qué parámetros mínimos.**

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `T4CIAS` | Num | Empresa |
| `T4MAGA` | Alfa | Almacén |
| `T4MLIN` | Num | Máximo de líneas por OC |
| `T4CAPA` | Num | Capacidad total del almacén |
| `T4DISP` | Num | Porcentaje disponible para DOM |
| `T4IMPO` | Num | Importe mínimo de pedido |
| `T4PESO` | Num | Peso máximo por expedición |
| `T4VOLU` | Num | Volumen máximo por expedición |
| `T4HORZ` | Num | Horizonte de planificación (días) |
| `T4PMIN` | Num | % por encima del stock mínimo |
| `T4ACTO` | Alfa | Incluir almacén propio de la delegación |
| `T4DOBC` | Alfa | `'1'` = omitir check de obsolescencia |

### 4.3 EDPAR3 — Restricciones de Capacidad por Almacén

Complementa EDPAR2 con límites físicos más detallados:

| Campo | Descripción |
|-------|-------------|
| `T5MLIN / T5MLIA` | Max líneas totales / actual |
| `T5CAPA` | Capacidad total |
| `T5DISP` | % disponible |
| `T5IMPO / T5PESO / T5VOLU` | Límites absolutos |
| `T5HORZ` | Horizonte de planificación |
| `T5RFLU` | Flag Red Fluidra (distribución interna) |

### 4.4 EDPARC — Parámetros por Cliente

Reglas específicas para clientes en el proceso DOM. Permite sobreescribir los defaults de delegación para un cliente concreto.

| Campo | Descripción |
|-------|-------------|
| `T4CIAS + T4CUST` | Empresa + cliente (clave) |
| `T4NIMF` | Omitir importe mínimo en ED forzada |
| `T4OBSO` | `'1'` = verificar artículos obsoletos |
| `T4SSTO` | `'1'` = verificar sobrestock |
| `T4PEXP` | Número máximo de puntos de expedición |
| `T4LOGI` | Validar restricciones logísticas |
| `T4STCK` | Validar restricciones de stock |
| `T4EDFO` | Solo servir por ED forzada (omitir almacenes propios) |
| `T4LIMP / T4LIME` | Límites de coste por pedido / por punto de expedición |

### 4.5 EDPARM — Parámetros por Empresa/Delegación

Valores por defecto del DOM para una empresa/delegación. Actúa como template que `EDPARC` sobreescribe por cliente.

| Campo | Descripción |
|-------|-------------|
| `T3CIAS + T3DLGA` | Empresa + delegación (clave) |
| `T3NIMF` | Omitir importe mínimo |
| `T3OBSO` | Verificar obsolescencia |
| `T3SSTO` | Verificar sobrestock |
| `T3PEXP` | Max puntos de expedición |
| `T3ENIN` | Habilitar entrada de incidencias |
| `T3LIMP / T3LIME` | Límites de coste |
| `T3DIBE` | Días de bloqueo previos a expedición |
| `T3LOGI / T3STCK / T3EDFO` | Flags de lógica DOM |

### 4.6 MSDDOM — Matriz de Delegaciones

**Define las relaciones permitidas entre delegaciones de origen y destino**, incluyendo calendarios de envío, transportistas, límites y prioridades especiales.

**Estructura de clave:**
```
XACIAS (empresa) + XADLGA (delegación destino) + XADLG2 (delegación origen) + XATIPD (tipo)
```

**Campos principales:**

| Campo | Descripción |
|-------|-------------|
| `XATTRA` | Tipo de transporte (shipment directo flag) |
| `XAAGEN` | Código de agencia de transporte |
| `XASSTR` | Flag para omitir check de envío |
| `XASSDP` | Flag para omitir check de almacén |
| `XASSST` | Flag para omitir check de stock |
| `XAPRIS` / `XAPRIR` | Prioridades nivel 1 y 2 |
| `XALIMI / XALIMV` | Límites de coste/volumen por delegación |
| `XADI01-07` | Días de semana habilitados para envío (1=Lun...7=Dom) |
| `XAPR01-04` | Prioridades para categorías OB, OC, SSTO, ArtB |
| `XAIR01-04` | Restricciones de importe para OB, OC, SSTO, ArtB |

**Lógicos especializados:**

| Fichero | Ordenación | Uso en DOM |
|---------|-----------|------------|
| `MSDDOM11` | Por prioridad OB (Overstock) | Cuando el DOM busca almacenes prioritarios para liquidar overstock |
| `MSDDOM12` | Por prioridad OC (Obsolete) | Cuando el DOM busca almacenes para liquidar obsoletos |
| `MSDDOM13` | Por prioridad SSTO (Super-Stock) | Para liquidación de sobrestock crítico |
| `MSDDOM14` | Por prioridad ArtB (Artículo B) | Para artículos de categoría B (baja rotación) |

### 4.7 Tablas de trabajo DOM (DOM005-008)

Tablas de trabajo usadas por el algoritmo DOOM durante el proceso (en QTEMP):

| Tabla | Uso |
|-------|-----|
| `DOM005` / `DOM00500` | Auditoría de cambios en maestro de proveedores |
| `DOM006` / `DOM00600` | Reglas de asignación proveedor-almacén con límites de cantidad y volumen |
| `DOM007` / `DOM00700` | Elegibilidad de artículo en almacén con tracking de cambios |
| `DOM008` / `DOM00800` | Capacidades específicas del almacén por artículo |

---

## 5. Árboles de Decisión

### 5.1 Árbol 1: ¿Se activa el Motor DOM?

```
¿El pedido es de tipo mostrador (S0MOST='1')?
├── SÍ → NO se activa DOOM → procesamiento manual
└── NO
    │
    ¿El pedido está pre-dividido (S0SPLF flag)?
    ├── SÍ → NO se activa DOOM → ya tiene almacén asignado
    └── NO
        │
        ¿El artículo es de servicio (sin stock, tipo SERV)?
        ├── SÍ → NO se activa DOOM para esta línea
        └── NO
            │
            ¿El proveedor está habilitado para ED en EDPAR4?
            ├── NO → NO se activa DOOM → pedido normal sin ED
            └── SÍ → SE ACTIVA DOOM ✓
```

### 5.2 Árbol 2: Selección de almacén en DOOM

```
Para cada almacén candidato:
│
├── ¿Existe excepción en EDPAR4 para este artículo (TREG=2/C)?
│   ├── SÍ → USAR almacén de la excepción (prioridad máxima)
│   └── NO → continuar evaluación general
│
├── ¿Existe registro en MSSTOK para este artículo en este almacén?
│   ├── NO → almacén DESCARTADO
│   └── SÍ → continuar
│
├── ¿Hay stock disponible (M1DISP >= cantidad)?
│   ├── SÍ (suficiente) → almacén ELEGIBLE ✓
│   ├── SÍ (parcial) → almacén ELEGIBLE PARCIAL (registrar)
│   └── NO → almacén DESCARTADO
│
├── ¿Check de obsolescencia activo (T3OBSO='1')?
│   ├── SÍ y artículo obsoleto Y no está en MSDDOM12 → DESCARTADO
│   └── NO o artículo no obsoleto → continuar
│
├── ¿Check de sobrestock activo (T3SSTO='1')?
│   ├── SÍ y sobrestock detectado Y no está en MSDDOM13 → DESCARTADO
│   └── NO → continuar
│
├── ¿Existe acuerdo logístico en EDPAR4 (TREG=0/A o superior)?
│   ├── NO → almacén DESCARTADO
│   └── SÍ → continuar
│
├── ¿Supera restricciones físicas EDPAR3 (peso/volumen/importe)?
│   ├── NO → almacén DESCARTADO
│   └── SÍ → continuar
│
├── ¿Está dentro del horizonte de planificación (EDPAR2.T4HORZ)?
│   ├── NO → almacén DESCARTADO
│   └── SÍ → continuar
│
└── ¿Es día hábil para envío según MSDDOM.XADI01-07?
    ├── NO → almacén DESCARTADO
    └── SÍ → almacén ELEGIBLE ✓

De todos los almacenes ELEGIBLES:
→ ORDENAR por prioridad MSDDOM.XAPRIS (descendente)
→ SELECCIONAR el primero
→ ACTUALIZAR CODEPE.S5LOGE
```

### 5.3 Árbol 3: Tratamiento especial de categorías de artículo

```
¿El artículo tiene flag de OVERSTOCK (OB)?
├── SÍ → ¿Existe almacén con prioridad OB en MSDDOM11?
│         ├── SÍ → USAR almacén de prioridad OB
│         └── NO → Continuar con lógica estándar
└── NO

¿El artículo tiene flag de OBSOLETE (OC)?
├── SÍ → ¿Existe almacén con prioridad OC en MSDDOM12?
│         ├── SÍ → USAR almacén de prioridad OC (liquidación)
│         └── NO → Artículo INELEGIBLE para DOM (no liquidar sin almacén configurado)
└── NO

¿El artículo tiene flag de SUPER-STOCK (SSTO)?
├── SÍ → ¿Existe almacén con prioridad SSTO en MSDDOM13?
│         ├── SÍ → USAR almacén de prioridad SSTO
│         └── NO → Continuar con lógica estándar
└── NO

¿El artículo es de categoría B (ArtB)?
├── SÍ → ¿Existe almacén con prioridad ArtB en MSDDOM14?
│         ├── SÍ → USAR almacén de prioridad ArtB (con límites adicionales)
│         └── NO → Continuar con lógica estándar
└── NO

→ Aplicar lógica estándar (árbol 2)
```

### 5.4 Árbol 4: ¿Se genera EDI?

```
Tras confirmación del DOM (usuario acepta en ED3061):
│
├── ¿El proveedor asignado tiene PUCEDI configurado?
│   ├── NO → NO se genera EDI → OC manual
│   └── SÍ → continuar
│
├── ¿PUCEDI.C9EPED = '1' (enviar ORDERS habilitado)?
│   ├── NO → NO se genera EDI → OC manual
│   └── SÍ → continuar
│
├── ¿La OC supera el importe mínimo de la expedición (EDPARM.T3LIMP)?
│   ├── NO y EDPARC.T4NIMF = '0' → NO se genera EDI (importe insuficiente)
│   └── SÍ o T4NIMF='1' → continuar
│
└── GENERAR mensaje ORDERS (CALL ED0060)
    ├── Construir mensaje EDIFACT ORDERS
    ├── Enviar a Edicom (proveedor asignado)
    ├── Registrar en EDTRAU (evento de envío)
    └── Actualizar EDTRLP (flag T0EDSN='1')
```

---

## 6. Flujo Completo de un Pedido por el DOM

### 6.1 Diagrama de flujo detallado

```
INICIO: CO2519 ha lanzado el pedido (número de orden asignado)
   │
   ▼
ED3025CL (CL wrapper)
   │ CPYF COCAPE TO QTEMP/COCAPE
   │ CPYF CODEPE TO QTEMP/CODEPE
   │ CPYF CODEPE20 TO QTEMP/CODEPE20
   │ CPYF CODEPE26 TO QTEMP/CODEPE26
   │ CPYF COHFEN TO QTEMP/COHFEN
   │ CALL ED3025
   ▼
ED3025 (orquestador)
   │ Recibe: CIAS, DLGA, NPCO, TPED, DATP, hora
   │
   ├── ¿S0MOST = '1' (mostrador)?
   │   └── SÍ → saltar DOOM, ir a ED3061 directamente
   │
   ├── ¿S0SPLF (pre-split)?
   │   └── SÍ → saltar DOOM, ir a ED3061 directamente
   │
   └── NO → CALL ED3050 (DOOM algorithm)
           │
           └─── DOOM evalúa línea a línea:
                (ver sección 3 para el detalle)
                │
                ├── Resultado OK → CODEPE.S5LOGE actualizado
                └── Resultado ERROR (ZXERRO='R') → error de riesgo
   │
   ├── ¿ZXERRO = 'R' (error de riesgo)?
   │   └── SÍ → NO llamar a ED3061 (bloqueo)
   │
   └── NO → CALL ED3061 (pantalla de confirmación)
           │
           └── Usuario revisa resultado DOM
               │
               ├── CANCEL (F12/F3) → abortar proceso DOM
               │
               └── CONFIRMAR → CALL ED0060
                               │
                               └── Generar mensajes EDI
                                   ├── ORDERS → proveedor (si PUCEDI habilitado)
                                   ├── actualizar EDTRLP.T0EDSN
                                   │
                                   └── CALL ED0100 (auditoría)
                                       ├── Escribir EDTRAU (eventos)
                                       └── Escribir DOOMHT (histórico decisión)
   │
   └── Limpieza QTEMP
```

### 6.2 Ejemplo concreto: pedido con 3 líneas

**Contexto:**
- Empresa 001, Delegación MADR5
- Pedido nº 12345, tipo VT
- Cliente ABC123 — configurado en EDPARC con T4OBSO='1', T4SSTO='0'

**Línea 1:** Artículo A-001 (normal), cantidad 50
```
Evaluar almacén MAD01:
  ✓ Existe en MSSTOK: 80 unidades disponibles (suficiente)
  ✓ No obsoleto, no sobrestock
  ✓ Existe en EDPAR4 (TREG=0, prioridad empresa)
  ✓ Cumple EDPAR3: peso 25kg < 500kg máx., importe 1200€ > 500€ mín.
  ✓ Día hábil para envío (MSDDOM.XADI02=1, martes habilitado)
→ ELEGIBLE — PRIORIDAD 5
Evaluar almacén BCN02:
  ✓ Existe en MSSTOK: 60 unidades
  ✓ Existe en EDPAR4 (TREG=A, prioridad delegación)
→ ELEGIBLE — PRIORIDAD 7 (mayor que MAD01)
SELECCIÓN: BCN02 (mayor prioridad)
CODEPE.S5LOGE = 'BCN02'
```

**Línea 2:** Artículo B-055 (obsoleto), cantidad 10
```
Evaluar almacén BCN02:
  ✓ Existe en MSSTOK: 30 unidades
  ✗ MSSTOK.M5OBSO='1' Y T4OBSO='1' → artículo OBSOLETO
    ¿Está BCN02 en MSDDOM12 (prioridad OC)?
    ✓ SÍ → almacén habilitado para liquidación obsoletos
→ ELEGIBLE — PRIORIDAD OC
SELECCIÓN: BCN02 (configurado para obsoletos)
CODEPE.S5LOGE = 'BCN02'
```

**Línea 3:** Artículo C-999 (servicio), cantidad 1
```
MSARTI00.M5TIPO = 'SERV'
→ BYPASS DOOM — artículo de servicio
→ No se asigna código logístico
→ Gestión manual
```

---

## 7. Generación de Mensajes EDI tras el DOM

### 7.1 Proceso de generación (ED0060)

Tras la confirmación del usuario en ED3061, `ED3025` llama a `ED0060` que:

1. **Lee la configuración del proveedor seleccionado** (PUCEDI):
   - ¿Está habilitado para ORDERS? (`C9EPED='1'`)
   - Roles EDIFACT (COMP, RECE, CFAC, PAGA)
   - Emisor/receptor Edicom

2. **Lee parámetros de validación** (EDPARM, EDPARC, EDPAR4):
   - ¿Se puede enviar sin alcanzar importe mínimo? (T3NIMF / T4NIMF)
   - ¿Forzar coenvío? (T6COEN)

3. **Construye el mensaje ORDERS** en formato EDIFACT con:
   - Cabecera: número de orden, fecha, proveedor, comprador
   - Líneas: artículo, cantidad, precio, fecha de entrega solicitada
   - Referencias: número de pedido de venta origen, condiciones

4. **Envía a Edicom** para distribución al proveedor

5. **Actualiza EDTRLP**:
   - `T0EDSN = '1'` (flag de EDI enviado)
   - `T0EDFC` (código de función EDI)

### 7.2 Respuesta del proveedor (ORDRSP)

Cuando el proveedor responde a la OC:
1. Edicom recibe el ORDRSP y lo entrega al sistema AMS
2. El evento se registra en `EDTRAU` con código de evento ORDRSP
3. `ED0015` procesa la respuesta: actualiza fechas y cantidades en CODEPE y PUCAPE
4. Si hay diferencias → se activan alertas en la pantalla de seguimiento

### 7.3 Generación de DESADV e INVOIC

Estos mensajes se generan en fases posteriores del ciclo:
- **DESADV**: cuando se confirma la expedición (no en el DOM)
- **INVOIC**: cuando se emite la factura al cliente
Ambos se configuran en COCEDI (tabla del cliente).

---

## 8. Trazabilidad y Auditoría

### 8.1 Los tres niveles de trazabilidad

```
Nivel 1 — DOOMHT (decisión)
  ┌─────────────────────────────────────────────────────┐
  │ Por cada ejecución del DOOM:                         │
  │   - Empresa, delegación, OC, tipo, fecha/hora       │
  │   - Tipo de proceso (DOPROC)                        │
  │   - Datos de la decisión (DODATA — blob/JSON)       │
  │   - Usuario que ejecutó (DOUSUA)                    │
  │   - Timestamp (DOTMSP)                              │
  └─────────────────────────────────────────────────────┘

Nivel 2 — EDTRAU (eventos)
  ┌─────────────────────────────────────────────────────┐
  │ Por cada evento significativo del proceso EDI:       │
  │   - Código de evento (T1EDAU) + descripción         │
  │   - Datos del evento (T1DATO, 600 bytes)            │
  │   - Posibles errores (T1EDER) y advertencias        │
  │   - Usuario, job, programa origen                   │
  │   - Cantidad involucrada (T1CANT)                   │
  │   - Estado final (T1STUL)                           │
  └─────────────────────────────────────────────────────┘

Nivel 3 — EDTRLP (línea)
  ┌─────────────────────────────────────────────────────┐
  │ Por cada línea de pedido en el proceso:              │
  │   - Estado completo: CANT, CANR, CANP, CANS, CAND  │
  │   - Fechas: FLAN, FREP, FURE, FRRE, FRRO           │
  │   - Flags: EDSN, EDFC, AUCP, FOPR, MODI, RISK      │
  │   - Referencias cruzadas: hasta 3 OC paralelas      │
  │     (T0CIA1-3, T0DLG1-3, T0NPC1-3, T0CAN1-3)       │
  │   - Contadores de coenvío (T0NCOE, T0NCOI, T0NCOF) │
  └─────────────────────────────────────────────────────┘
```

### 8.2 Flags de estado en EDTRLP

| Flag | Campo | Valor | Significado |
|------|-------|-------|-------------|
| EDI enviado | `T0EDSN` | `'1'` | Se ha enviado mensaje ORDERS |
| Función EDI | `T0EDFC` | código | Función EDI de la última operación |
| Resultado ED3025 | `T0ED3R` | código | Resultado de la última ejecución DOM |
| Auto-confirmación | `T0AUCP` | `'1'` | La línea se confirmó automáticamente |
| Forzar preparación | `T0FOPR` | `'1'` | Flag de forzar preparación |
| Modificación | `T0MODI` | `'1'` | La línea ha sido modificada |
| Riesgo | `T0RISK` | `'1'` | La línea tiene alerta de riesgo |
| Retención | `T0PHLD` | `'1'` | La línea está retenida |
| Documento X | `T0XDOC` | código | Referencia a documento cruzado |
| Porte | `T0PORT` | `'1'` | Incluye cálculo de portes |

### 8.3 Consultar el histórico de decisiones

Para consultar las decisiones DOM históricas de un pedido:

```
CHAIN DOOMHT con: CIAS + DLGA + NPCO + TPED
READ DOOMHT (secuencial por fecha/hora)
→ Cada registro: una decisión DOM para ese pedido
```

Para consultar el log de eventos EDI:

```
SETLL EDTRAU con: CIAS + DLGA + NPCO + TPED + DATP
READE EDTRAU → cada evento del proceso EDI para ese pedido
```

Para consultar el estado completo de una línea:

```
CHAIN EDTRLP con: CIAS + DLGA + NPCO + TPED + DATP + HORA + LINE
→ Registro completo con ~98 campos de estado
```

---

## 9. Configuración del DOM — Guía Operativa

### 9.1 Configurar un nuevo proveedor para el DOM

**Paso 1:** Crear el proveedor en `PUPROV` (si no existe)

**Paso 2:** Habilitar EDI del proveedor en `PUCEDI`:
```
Empresa + Proveedor
  C9EPED = '1'    ← habilitar envío de ORDERS
  C9RRSP = '1'    ← habilitar recepción de ORDRSP
  Roles EDIFACT: C9COMP, C9RECE, C9CFAC, C9PAGA
  Código Edicom del proveedor
```

**Paso 3:** Crear reglas de prioridad en `EDPAR4` (programa `DO0005`):
```
Para la prioridad general del proveedor:
  CIAS + MAGA (almacén origen) + TREG='1' + PROVEEDOR
  → Prioridad en la lista de almacenes para este proveedor

Para excepciones por artículo:
  CIAS + MAGA + TREG='2' + ARTÍCULO
  → Override específico para ese artículo
```

**Paso 4:** Verificar/crear entrada en `EDPAR2` para el almacén:
```
CIAS + MAGA
  T4MLIN: máximo de líneas por OC
  T4IMPO: importe mínimo de pedido
  T4PESO/T4VOLU: restricciones físicas
  T4HORZ: horizonte de planificación
```

**Paso 5:** Verificar `MSDDOM` para la delegación:
```
CIAS + DLGA_DESTINO + DLGA_ORIGEN
  XADI01-07: días habilitados para envío
  XAPRIS: prioridad del almacén origen
```

### 9.2 Gestionar excepciones para artículos específicos

Para que un artículo siempre se sirva desde un almacén concreto:

```
En DO0005, crear regla EDPAR4:
  TREG = 'C' (excepción DLGA + artículo)
  Campos: CIAS + MAGA + DLGA + T6ARVT (artículo)
  T6PRIO = 1 (máxima prioridad)
```

Para excluir un artículo de la ED:

```
En EDPAR4: no crear registro para este artículo+proveedor
→ El DOM no encontrará acuerdo logístico → bypasa el artículo
```

### 9.3 Configurar tratamiento de artículos obsoletos

**Para permitir servir obsoletos desde un almacén específico:**

```
En MSDDOM para la delegación correspondiente:
  XAPR02 (prioridad OC) = número de prioridad
  XAIR02 (límite importe OC) = importe máximo
→ Ese almacén aparecerá en MSDDOM12 con esa prioridad
```

**Para activar el check de obsolescencia por defecto:**

```
En EDPARM:
  T3OBSO = '1' ← activar para toda la delegación

O en EDPARC por cliente:
  T4OBSO = '1' ← activar solo para ese cliente
```

### 9.4 Configurar límites por cliente (EDPARC)

Para configurar reglas específicas para un cliente prioritario:

```
Crear/modificar registro en EDPARC:
  T4CIAS + T4CUST (empresa + cliente)
  T4NIMF = '1'   ← omitir importe mínimo (sirve aunque no alcance el mínimo)
  T4PEXP = 3     ← máximo 3 puntos de expedición
  T4EDFO = '1'   ← solo servir por ED forzada (nunca desde almacén propio)
  T4LIMP = 5000  ← límite de 5000€ por pedido
```

---

## 10. Casos Especiales y Excepciones

### 10.1 Artículos de Servicio

Los artículos de tipo SERVICIO (campo `M5TIPO='SERV'` o `M5SIOL='S'` en MSARTI00) se **excluyen completamente** del algoritmo DOOM. No se asigna código logístico y se gestionan fuera del flujo automático.

**Identificación:** la evaluación de tipo servicio ocurre en el primer tier del DOOM, antes de cualquier otra validación.

### 10.2 Kits y Conjuntos

Los artículos conjunto (kits) tienen tratamiento especial:

1. `CO2733` determina si el conjunto es "Trace" (todos componentes), "no-Trace", o "combinado"
2. Para conjuntos Trace: el DOOM explota el kit en sus componentes y evalúa cada uno
3. El stock del kit = mínimo de stock de todos sus componentes (multiplicado por la cantidad del componente)
4. La fecha de entrega del kit = la más desfavorable de todos los componentes

**Ficheros implicados:** MSCOJC (cabecera), MSCOJD (componentes), CO2733 (evaluación)

### 10.3 Pedidos con Coenvíos

En pedidos con coenvío, el DOOM debe respetar la integridad del coenvío:
- Todas las líneas del coenvío deben poder ser servidas desde el mismo almacén
- Si no es posible → se registra como error de coenvío
- El flag `T6COEN='1'` en EDPAR4 puede forzar coenvío aunque el cliente no lo requiera

### 10.4 OL (Operative Locations) Múltiples

A partir de la modificación `F-2OL` (sep-2021) y `IT244` (may-2022), el sistema soporta asignación a más de un OL. Esto significa que una orden puede dividirse entre múltiples almacenes lógicos (múltiples códigos LOGE) en lugar de asignar todo a uno solo.

**Implicación en EDTRLP:** los campos `T0CIA1-3`, `T0DLG1-3`, `T0NPC1-3`, `T0CAN1-3` permiten registrar referencias cruzadas a hasta 3 OC paralelas para una misma línea.

### 10.5 Artículos Modificados (S5TPE4='2')

Las líneas de pedido con `S5TPE4='2'` son líneas generadas por una modificación/corrección de pedido (versión modificada). El KORE (CO2610) puede excluirlas del cálculo de pendientes si el parámetro `@POMFD='1'`.

---

## 11. Diagnóstico y Resolución de Problemas

### 11.1 El DOM no asigna almacén a una línea

**Verificar en orden:**

1. ¿El artículo es de servicio? → Ver MSARTI00.M5TIPO
2. ¿Existe registro en EDPAR4 para el proveedor? → Ver DO0001/DO0005
3. ¿Hay stock en algún almacén? → Ver MSSTOK para CIAS + artículo
4. ¿Están todos los almacenes descartados por restricciones? → Ver EDPAR3
5. ¿Está el check de obsolescencia bloqueando? → Ver MSARTI00.M5OBSO y EDPARM.T3OBSO
6. ¿Existe entrada en MSDDOM para la delegación? → Ver MSDDOM

### 11.2 El EDI no se genera después del DOM

**Verificar:**

1. ¿Existe registro en PUCEDI para el proveedor asignado? → Ver PUCEDI
2. ¿C9EPED='1' (habilitado para ORDERS)? → Ver PUCEDI.C9EPED
3. ¿El importe supera el mínimo? → Ver EDPARM.T3LIMP y EDPARC.T4NIMF
4. ¿Hubo error de riesgo (ZXERRO='R') que bloqueó la generación? → Ver EDTRAU
5. ¿El usuario canceló en ED3061? → Ver log de auditoría EDTRAU

### 11.3 El proveedor recibió una OC incorrecta

1. Consultar `EDTRAU` para el pedido → ver todos los eventos del proceso
2. Consultar `EDTRLP` para la línea → ver el estado completo y flags
3. Consultar `DOOMHT` → ver la decisión DOM original
4. Verificar `EDPAR4` → ¿era correcta la prioridad configurada?
5. Si hay que re-lanzar: cancelar la OC anterior y re-ejecutar ED3025

### 11.4 Interpretar los flags de EDTRLP

| Situación | Flags a revisar |
|-----------|----------------|
| Línea procesada por DOM | `T0ED3R <> ' '` |
| EDI enviado | `T0EDSN = '1'` |
| Línea retenida | `T0PHLD = '1'` |
| Línea con riesgo | `T0RISK = '1'` |
| Línea modificada | `T0MODI = '1'` |
| Auto-confirmada | `T0AUCP = '1'` |
| Múltiples OC | `T0CIA2 <> ' '` → hay OC secundaria |

---

## 12. Programas del Motor DOM — Referencia Rápida

| Programa | Tipo | Función | Ficheros principales |
|----------|------|---------|---------------------|
| `ED3025CL` | CL Wrapper | Punto de entrada batch. Aislamiento QTEMP. | — |
| `ED3025` | RPG Orch. | Orquestador: decide flujo, llama subprogramas | COCAPE, CODEPE, COHFEN |
| `ED3050` | RPG Batch | Algoritmo DOOM completo | TODOS los ficheros DOM |
| `DOOM01` | RPG Batch | Núcleo del cubo de decisión DOOM | COCAPE, CODEPE, MSSTOK, EDPAR* |
| `ED3061` | RPG Panel | Pantalla confirmación resultado DOM | DO0050W-053W |
| `ED3062` | RPG Panel | Detalle por línea del resultado DOM | DO0050W-053W |
| `ED3063` | RPG Panel | Excepciones y errores DOM | DO0050W-053W |
| `ED0060` | RPG Batch | Generación mensajes EDI (ORDERS, etc.) | PUCEDI, EDPARM, EDPARC, EDPAR4 |
| `ED0100` | RPG Batch | Auditoría y trazabilidad | EDTRAU, EDTRLP, DOOMHT |
| `ED0015` | RPG Batch | Sincronización eventos venta ↔ compra | COCAPE, CODEPE, PUCAPE, PUDEPE |
| `DO0001` | RPG SFL | Lista prioridades DOM (v1) | EDPAR4, EDPARM, MSALMA |
| `DO0005` | RPG Panel | Mantenimiento prioridad DOM (v1) | EDPAR4, EDHPA4 |
| `DO0010` | RPG SFL+P | Parámetros Fluidra Direct | — |
| `DO0020` | RPG SFL | Lista prioridades DOM (v2) | EDPAR4, EDPARM, MSALMA |
| `DO0025` | RPG Panel | Mantenimiento prioridad DOM (v2) | EDPAR4, EDHPA4 |
| `DO0030` | RPG SFL | Lista prioridades empresa | EDPAR4 |
| `DO0035` | RPG Panel | Mantenimiento prioridades empresa | EDPAR4, EDHPA4 |
| `DO0042` | RPG Panel | Variante mantenimiento DOM | EDPAR4 |
| `DO0043` | RPG Panel | Variante mantenimiento DOM | EDPAR4 |
| `ED1000CL` | CL Wrapper | Punto de entrada simulación ED | — |
| `ED1000` | RPG Orch. | Simulación y confirmación de entregas directas | COCAPE, CODEPE, EDPAR* |
| `EDINCI` | RPG | Inicialización del proceso DOM | — |
| `EDINCI30` | RPG | Variante de inicialización | — |
| `EDMACI` | RPG | Maestro de configuración DOM | EDPARM, EDPARC |
| `EDMAIC` | RPG | Variante maestro DOM | EDPARM, EDPARC |
| `EDMAID` | RPG | Datos de maestro DOM | EDPARM |
| `EDMAII` | RPG | Mantenimiento interno DOM | EDPARM, EDPARC |
| `EDMAIV` | RPG | Vista de maestro DOM | EDPARM |
| `EDRFDG` | RPG | Referencia de datos DOM | — |
| `EDTRAU` | DDS/PF | Fichero físico auditoría eventos | — |
| `EDTRAX` | DDS/PF | Fichero físico auditoría extendida | — |
| `EDTRLP` | DDS/PF | Fichero físico trazabilidad de línea | — |
| `EDTRLX` | DDS/PF | Fichero físico trazabilidad extendida/histórico | — |
| `DOOMHT` | DDS/PF | Fichero físico histórico decisiones DOOM | — |
| `DOM005-008` | DDS/PF+LF | Tablas de trabajo DOOM | — |
| `EDPAR4` | DDS/PF | Prioridades de distribución (MAESTRO) | — |
| `EDPAR401` | DDS/LF | Lógico de EDPAR4 | — |
| `EDPAR2` | DDS/PF | Parámetros por almacén | — |
| `EDPAR3` | DDS/PF | Restricciones capacidad | — |
| `EDPARC` | DDS/PF | Parámetros por cliente | — |
| `EDPARM` | DDS/PF | Parámetros por delegación | — |
| `EDHPA4` | DDS/PF | Histórico cambios EDPAR4 | — |
| `EDHPAR` | DDS/LF | Lógico del histórico | — |
| `MSDDOM` | DDS/PF | Matriz delegaciones origen-destino | — |
| `MSDDOM11-14` | DDS/LF | Lógicos por prioridad OB/OC/SSTO/ArtB | — |

---

*Sistema AMS — Fluidra S.A. | Motor DOM en Profundidad | Versión 1.0 — Marzo 2026*
