# Resumen Ejecutivo del Sistema AMS
## Sistema de Gestión de Pedidos y Logística — Fluidra S.A.

---

## 1. Visión General

Este sistema es un conjunto de programas RPG para IBM i (AS/400) que dan soporte al ciclo completo de **gestión de pedidos de clientes**, **distribución logística** y **clasificación de artículos** dentro del ERP corporativo de Fluidra S.A. (anteriormente Aquapoint S.A.).

El sistema está organizado en tres módulos funcionales:

| Módulo | Sigla | Propósito Principal |
|--------|-------|---------------------|
| **Compras / Pedidos** | CO / CM | Ciclo de vida de pedidos de cliente: creación, confirmación, lanzamiento y expedición |
| **Logística / Distribución** | DO | Configuración de prioridades DOM y parámetros de distribución Fluido Direct |
| **Utilitarios y KORE** | CO (utilitarios) | Servicios de soporte: cálculo de fechas, rotación de artículos, stock pendiente |

---

## 2. Módulo de Compras (Pedidos de Cliente)

### ¿Qué hace?
Gestiona el proceso completo de un pedido de cliente, desde la captura hasta su expedición:

1. **Consulta de pedidos (CM0085)** — Lista de pedidos activos con filtros por cliente, estado, riesgo, delegación. Permite al usuario navegar y seleccionar pedidos para gestionar.

2. **Cabecera de pedido (CO0120)** — Creación y edición de la cabecera del pedido: cliente, delegación, tipo de pedido, fecha, referencias, condiciones de entrega. Incluye control de riesgo crediticio, integración con Fluidra Direct y soporte a pedidos EDI.

3. **Líneas de pedido (CO0140)** — Entrada y modificación del detalle de líneas: artículo, variante, modelo, cantidad, precio, descuentos, ecotasa, fecha de entrega. Controla compatibilidades entre artículos (ADR, Astralpool) y calcula rendimientos.

4. **Lanzamiento de órdenes (CO2519)** — Genera el número de orden de venta y la lanza al sistema de ejecución. Gestiona la integridad de coenvíos y el nivel de servicio.

5. **Validación de coenvíos (CO2696)** — Antes de expedir, verifica que todas las líneas de un coenvío estén reservadas al 100%.

6. **Parámetros de impresión (CO2593)** — Panel interactivo para configurar parámetros de impresión de documentos (formato, número de copias, impresora). Compatible con ejecución en modo batch (sin pantalla).

### Flujo de un pedido típico

```
[Usuario] → CM0085 (consulta lista pedidos)
              ↓ selecciona pedido
           CO0120 (edita cabecera)
              ↓
           CO0140 (edita líneas / artículos)
              ↓
           CO2519 (lanza orden de venta)
              ↓ [al expedir]
           CO2696 (valida coenvío, si aplica)
```

---

## 3. Módulo de Logística (DOM — Distribución de Órdenes y Materiales)

### ¿Qué hace?
Permite configurar y mantener las **reglas de prioridad de distribución** que el motor DOM utiliza para decidir desde qué almacén y en qué orden se sirven los pedidos.

Las prioridades se organizan en dos dimensiones:
- **Nivel de empresa**: prioridades que aplican a toda la compañía
- **Nivel de delegación**: prioridades específicas por delegación (que sobreescriben las de empresa)

Y en varios tipos de criterio de distribución (TREG):

| TREG | Nivel empresa | TREG | Nivel delegación |
|------|--------------|------|-----------------|
| 0 | Prioridad global de empresa | A | Prioridad global de delegación |
| 1 | Por proveedor | B | Por delegación/proveedor |
| 2 | Por artículo | C | Por delegación/artículo |
| 3 | Por familia/subfamilia | D | Por delegación/familia |
| 4 | Por familia/tarifa | E | Por delegación/fam-subfamilia |
| 5 | Por cliente | F | Por delegación/fam-tarifa |
| — | — | G | Por delegación/cliente |
| — | — | H | Por delegación/procedimiento pedido |

### Programas implicados

| Programa | Tipo de interfaz | Función |
|----------|-----------------|---------|
| **DO0001** | Subfile (lista) | Mantenimiento nivel delegación: vista de prioridades empresa+delegación juntas |
| **DO0005** | Panel (formulario) | Alta/Baja/Copia/Modificación de una prioridad nivel delegación |
| **DO0010** | Panel (formulario) | Parámetros de Fluido Direct: configuración de reglas de distribución directa |
| **DO0020** | Subfile (lista) | Similar a DO0001 — variante de mantenimiento nivel delegación |
| **DO0025** | Panel (formulario) | Similar a DO0005 — variante con opciones extendidas nivel DLGA |

---

## 4. Servicios de Soporte (Utilitarios y KORE)

Programas de servicio invocados por otros módulos, no directamente por usuarios finales.

| Programa | Función |
|----------|---------|
| **CO2611** | Dado una fecha y un número de días laborables, calcula la fecha resultante (excluye sábados y domingos) |
| **CO2733** | Determina la situación de distribución de un conjunto de venta: si todos sus componentes se gestionan en Trace, ninguno, o es combinado |
| **CO2755** | Obtiene la categoría de rotación (MTS/MTO y variantes A/B/C) de un artículo según los parámetros de la empresa |
| **CO2770 / CO2610** | Obtiene los pedidos pendientes de servir a un cliente para un artículo en un rango de fechas (módulo KORE) |

---

## 5. Integrations Externas

El sistema se conecta con:

- **SalesForce** (vía API): CO2593 soporta ejecución batch cuando el usuario es `QUSER` (llamadas desde WS)
- **M3 / Movex**: CO2519 y CO0120 contemplan integración con el sistema Movex para gestión de coenvíos
- **EDI Internacional**: CO0120 y CO0140 gestionan pedidos EDI (tipo PPED=99) cargados desde COCPED
- **Motor DOM externo**: Los datos mantenidos por DO0001-DO0025 alimentan el motor de distribución de órdenes

---

## 6. Datos Clave del Sistema

| Concepto | Descripción |
|----------|-------------|
| **CIAS** | Código de empresa (3 dígitos) |
| **DLGA** | Delegación (5 caracteres) |
| **LOGE** | Código de almacén lógico |
| **TPED** | Tipo de pedido |
| **NPCO** | Número de pedido de cliente |
| **NOPL** | Número de línea de pedido |
| **MTS/MTO** | Make-To-Stock / Make-To-Order (clasificación de artículos) |
| **DOM** | Distribution Order Management |
| **Coenvío** | Envío agrupado de líneas: se expide todo junto o nada |

---

## 7. Historial del Sistema

| Período | Empresa | Observaciones |
|---------|---------|---------------|
| 1997–2005 | Aquapoint S.A. | Versiones originales (CM0085, CO0120, CO0140, CO2519) |
| 2005–2010 | Aquapoint / Transición | Ampliaciones: coenvíos, EDI, Fluidra Direct |
| 2010–presente | Fluidra S.A. | Nuevos programas (DO\*, CO2755, CO2610), mantenimiento continuo |
| 26/11/2025 | Fluidra S.A. | Última modificación registrada (CO2755 — SCE-104) |

---

*Documentación generada a partir del código fuente. Versión 1.0 — Marzo 2026.*
