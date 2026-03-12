# Resumen Ejecutivo del Sistema AMS — Fluidra S.A.
## Sistema Integral de Gestión Logística, Pedidos y Distribución

**Versión:** 2.0 — Marzo 2026
**Audiencia:** Dirección, gestores funcionales, responsables de proyecto
**Clasificación:** Interno

---

## 1. ¿Qué es este sistema?

El **Sistema AMS de Fluidra S.A.** es el núcleo operativo de la cadena de suministro de Fluidra sobre plataforma IBM i (AS/400). Gestiona el ciclo completo de negocio desde la recepción de un pedido de cliente hasta la expedición del producto, incluyendo la optimización automática de fuente de suministro, la coordinación con proveedores mediante EDI, y la gestión de los maestros de artículos, stocks y almacenes.

El sistema está compuesto por **418 programas fuente** (RPG IV + CL), más de **153.000 líneas de código**, y lleva en producción desde 1997 con ampliaciones y modificaciones continuas hasta la actualidad (última modificación registrada: diciembre 2025).

---

## 2. Módulos Principales

El sistema se organiza en **seis dominios funcionales** más un motor de decisión centralizado:

| # | Módulo | Prefijo | Programas | Función principal |
|---|--------|---------|-----------|-------------------|
| 1 | **Pedidos de Venta** | CO / CM | ~106 | Ciclo completo del pedido de cliente: creación, validación, lanzamiento, expedición |
| 2 | **Compras** | PU | ~37 | Gestión de órdenes de compra a proveedores: creación, seguimiento, recepción |
| 3 | **Motor DOM / Distribución Directa** | ED / DO / DOM | ~61 | Algoritmo de optimización de fuente de suministro (DOOM); entregas directas; configuración de prioridades |
| 4 | **Maestros y Parametrización** | MS / PM / MM | ~93 | Artículos, stocks, almacenes, unidades de medida, parámetros de empresa |
| 5 | **EDI e Intercambio Electrónico** | TR / WR / COCEDI / PUCEDI | ~20 | Mensajes EDI (ORDERS, DESADV, INVOIC) con clientes y proveedores |
| 6 | **Configuración y Auxiliares** | CM / GX / CF / CC / MI / ZQ | ~101 | Setup del sistema, inventarios, balances, gateways logísticos |

---

## 3. El Motor DOM — Elemento Diferencial del Sistema

El **Motor DOM (Distribution Order Management)** es el núcleo de inteligencia del sistema. Cuando se confirma un pedido de venta, el motor DOM analiza automáticamente y en tiempo real qué almacén o proveedor debe servir cada línea, aplicando un **algoritmo de optimización multidimensional (DOOM)** que evalúa:

- Stock disponible en cada almacén
- Capacidad y límites del almacén (peso, volumen, importe mínimo)
- Acuerdos logísticos entre proveedor y almacén
- Prioridades configuradas por empresa, delegación, proveedor, artículo o cliente
- Tratamiento especial de artículos obsoletos, sobrestock y categoría B
- Restricciones de transporte y calendarios de envío

**Resultado:** cada línea de pedido recibe automáticamente el código logístico del almacén/proveedor óptimo, y el sistema genera los mensajes EDI correspondientes al proveedor seleccionado.

El motor DOM conecta directamente los módulos de Ventas, Compras y Stock en un único flujo automatizado.

---

## 4. Integración con Sistemas Externos

El sistema AMS no opera de forma aislada. Se integra con:

| Sistema externo | Tipo de integración | Programas implicados |
|----------------|--------------------|--------------------|
| **EDI / Edicom** (UN-EDIFACT) | Mensajes ORDERS, DESADV, INVOIC, ORDRSP, INVRPT con clientes y proveedores | ED0060, COCEDI, PUCEDI, EDTRAU, EDTRLP |
| **M3 / Movex** | Coordinación de coenvíos y albaranes a terceros | CO2519, CO0120 |
| **SalesForce** | Ejecución batch de programas vía API (sin pantalla) | CO2593 |
| **Motor KORE** | Cálculo de pedidos pendientes de servir por artículo | CO2610 / CO2770 |
| **Fluidra Direct (FD)** | Canal de distribución directa empresa productiva → cliente final | ED1000, ED3025, CO0120 |

---

## 5. Volumetría del Sistema

| Indicador | Valor |
|-----------|-------|
| Programas totales | 418 |
| Líneas de código | ~153.000 |
| Programas interactivos (con pantalla) | ~120 |
| Subprogramas de servicio (sin pantalla) | ~80 |
| Ficheros DDS (PF + LF + Display) | ~227 |
| Ficheros físicos principales | ~80 |
| Wrappers CL (puntos de entrada batch) | 4 |
| Módulos de negocio identificados | 6 |
| Integraciones externas activas | 5 |
| Período activo en producción | 1997 – actualidad |

---

## 6. Ciclo de Negocio de Punta a Punta

```
CLIENTE
  │
  ▼
[1] PEDIDO DE VENTA
    CM0085 → CO0120 → CO0140
    (consulta → cabecera → líneas)
  │
  ▼
[2] LANZAMIENTO
    CO2519 (asigna número de orden, valida coenvíos, fecha de entrega)
  │
  ▼
[3] MOTOR DOM / DOOM
    ED3025CL → ED3050/DOOM → ED3061 (confirmación)
    ┌─ Lee stock disponible (MSSTOK)
    ├─ Evalúa prioridades almacén/proveedor (EDPAR4, MSDDOM)
    ├─ Aplica reglas de negocio (obsoletos, sobrestock, capacidad)
    └─ Selecciona almacén óptimo (CODEPE.S5LOGE)
  │
  ▼
[4] GENERACIÓN EDI
    ED0060 → mensajes ORDERS al proveedor (vía PUCEDI/Edicom)
  │
  ▼
[5] PROVEEDOR / ALMACÉN
    Recibe PO, confirma (ORDRSP), prepara y envía (DESADV)
  │
  ▼
[6] RECEPCIÓN / FACTURACIÓN
    Módulo PU (recepción OC)
    Módulo INVOIC (factura al cliente vía COCEDI)
  │
  ▼
CLIENTE (recibe producto + factura)
```

---

## 7. Datos Clave del Sistema (Glosario Ejecutivo)

| Término | Significado operativo |
|---------|----------------------|
| **CIAS** | Código de empresa (3 dígitos) — el sistema es multiempresa |
| **DLGA** | Delegación — unidad organizativa de ventas/compras (5 caracteres) |
| **LOGE / MAGA** | Almacén lógico / físico — código de ubicación del stock |
| **DOM** | Distribution Order Management — motor de asignación automática de almacén |
| **DOOM** | Algoritmo central del DOM: cubo de decisión 3D (validaciones × almacenes × artículos) |
| **Coenvío** | Grupo de líneas de pedido que deben expedirse conjuntamente al 100% |
| **ED (Entrega Directa)** | Fluido Direct: envío directo del proveedor al cliente, sin pasar por almacén propio |
| **MTS / MTO** | Make-To-Stock / Make-To-Order — clasificación de artículos por política de fabricación |
| **EDI** | Electronic Data Interchange: intercambio electrónico de documentos comerciales |
| **EDPAR4** | Fichero maestro de prioridades de distribución del DOM |
| **TREG** | Tipo de regla de distribución (0-5 nivel empresa, A-H nivel delegación) |
| **TPED** | Tipo de pedido (determina comportamiento: normal, EDI, SC, etc.) |
| **SFL / ZXOPT** | Tecnología de pantallas IBM i: subfile (lista paginada); patrón de control de flujo |

---

## 8. Historial Evolutivo

| Período | Empresa | Hitos principales |
|---------|---------|-------------------|
| 1997–2005 | Aquapoint S.A. | Versiones originales: CM0085, CO0120, CO0140, CO2519 |
| 2005–2006 | Aquapoint | Coenvíos (C0031), EDI internacional (C0008), integración Movex |
| 2009–2010 | Transición a Fluidra | Control de riesgo crediticio (FQ365), Fluidra Direct (FQ284) |
| 2010–2014 | Fluidra S.A. | DOM v2: DO0001-DO0025, DOOM algorithm, motor KORE (CO2610) |
| 2014–2021 | Fluidra S.A. | Email confirmaciones (C1997), multi-OL (F-2OL), SalesForce API (B0251) |
| 2022–2025 | Fluidra S.A. | DOM 2ª fase OL (IT244), auditoría digital (PGE21), correcciones MTS/MTO (08835) |

---

## 9. Alcance de Esta Documentación

Esta suite documental cubre en detalle:

| Documento | Contenido | Audiencia |
|-----------|-----------|-----------|
| **Doc 1 — Arquitectura General** | Entorno IBM i, patrones técnicos, integraciones, mapa de módulos, modelo de datos | Técnico senior, arquitecto |
| **Doc 2 — Manual Funcional** | 6 capítulos por módulo de negocio: flujos, reglas, pantallas principales | Funcional + técnico |
| **Doc 3 — Motor DOM en Profundidad** | Algoritmo DOOM, árboles de decisión, tablas maestras, trazabilidad | Técnico + funcional clave |
| **Doc 4 — Referencia Técnica de Programas** | Ficha técnica de ~50 programas principales | AMS técnico |

---

*Sistema AMS — Fluidra S.A. | Documentación generada a partir de código fuente | Versión 2.0 — Marzo 2026*
