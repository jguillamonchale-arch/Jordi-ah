# Índice de Documentación — Sistema AMS Fluidra S.A.

**Versión:** 2.0 — Marzo 2026
**Sistema:** AMS — IBM i (AS/400), RPG IV, 418 programas, ~153.000 líneas

---

## Suite Documental Completa

### Documentos Funcionales

| Doc | Archivo | Contenido | Audiencia |
|-----|---------|-----------|-----------|
| **Doc 0** | [`FUNCIONAL/00_Resumen_Ejecutivo.md`](FUNCIONAL/00_Resumen_Ejecutivo.md) | Visión ejecutiva del sistema, módulos, integraciones, ciclo de negocio, volumetría | Dirección, gestores |
| **Doc 2** | [`FUNCIONAL/02_Manual_Funcional.md`](FUNCIONAL/02_Manual_Funcional.md) | Manual funcional por módulos: Ventas (CO), Compras (PU), DOM (ED/DO), Maestros (MS/PM/MM), EDI (TR/WR), Auxiliares | Funcional + técnico |
| **Doc 3** | [`FUNCIONAL/03_Motor_DOM_Profundo.md`](FUNCIONAL/03_Motor_DOM_Profundo.md) | **Análisis en profundidad del Motor DOM/DOOM**: algoritmo, árboles de decisión, tablas maestras, flujos, trazabilidad, configuración operativa | Técnico senior + funcional clave |

### Documentos Técnicos

| Doc | Archivo | Contenido | Audiencia |
|-----|---------|-----------|-----------|
| **Doc 1** | [`TECNICA/01_Arquitectura_General.md`](TECNICA/01_Arquitectura_General.md) | Arquitectura IBM i, patrones RPG (ZXOPT), mapa de módulos, modelo de datos global, ficheros DDS, flujos de llamadas, integraciones externas, convenciones | Técnico senior, arquitecto |
| **Doc 4** | [`TECNICA/04_Referencia_Tecnica_Programas.md`](TECNICA/04_Referencia_Tecnica_Programas.md) | Fichas técnicas de ~50 programas principales: parámetros, ficheros, llamadas, notas técnicas | AMS técnico, desarrolladores |

### Documentos Legacy (v1.0 — referencia histórica)

| Archivo | Contenido original |
|---------|-------------------|
| [`FUNCIONAL/Resumen_Ejecutivo.md`](FUNCIONAL/Resumen_Ejecutivo.md) | Resumen ejecutivo v1.0 (solo módulos CO/CM y DO) |
| [`FUNCIONAL/Manual_Funcional.md`](FUNCIONAL/Manual_Funcional.md) | Manual funcional v1.0 (solo módulos CO/CM y DO) |
| [`TECNICA/Arquitectura.md`](TECNICA/Arquitectura.md) | Arquitectura técnica v1.0 (solo módulos CO/CM y DO) |
| [`TECNICA/Programas_Referencia.md`](TECNICA/Programas_Referencia.md) | Referencia técnica v1.0 (15 programas) |

---

## Mapa del Sistema

```
SISTEMA AMS — FLUIDRA S.A.
│
├── VENTAS (CO/CM) ~106 pgms
│   Pedidos de cliente → lanzamiento → motor DOM
│
├── COMPRAS (PU) ~37 pgms
│   Órdenes de compra → proveedores → EDI
│
├── MOTOR DOM (ED/DO/DOM) ~61 pgms ← NÚCLEO DEL SISTEMA
│   Algoritmo DOOM: Ventas + Stock + Prioridades = Almacén óptimo
│   Genera EDI al proveedor seleccionado
│
├── MAESTROS (MS/PM/MM) ~93 pgms
│   Artículos, stock, almacenes, parámetros de empresa
│
├── EDI (TR/WR/COCEDI/PUCEDI) ~20 pgms
│   ORDERS, DESADV, INVOIC, ORDRSP, INVRPT via Edicom
│
└── AUXILIARES (CM/GX/CF/MI/ZQ/UT) ~101 pgms
    Configuración, inventarios, gateways, utilidades
```

---

## Guía de Uso para Equipo AMS Externo

| Si necesitas... | Lee este documento |
|----------------|-------------------|
| Entender el sistema en 5 minutos | Doc 0: Resumen Ejecutivo |
| Entender la arquitectura IBM i y los patrones RPG | Doc 1: Arquitectura General |
| Entender un módulo de negocio específico | Doc 2: Manual Funcional (capítulo del módulo) |
| Entender en profundidad el Motor DOM | Doc 3: Motor DOM en Profundidad |
| Modificar un programa específico | Doc 4: Referencia Técnica de Programas |
| Entender las tablas de base de datos | Doc 1 (sección 6) + Doc 4 (fichas DDS) |
| Configurar el DOM para nuevo proveedor | Doc 3 (sección 9) |
| Diagnosticar un problema en el DOM | Doc 3 (sección 11) |
| Entender la trazabilidad EDI | Doc 3 (sección 8) + Doc 2 (capítulo 5) |

---

*Documentación generada a partir del código fuente. Versión 2.0 — Marzo 2026*
