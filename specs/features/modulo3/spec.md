# Épica: Liquidación de Lote y Análisis de Rentabilidad (Módulo 3 – AVICONTROL)

**Created**: 2026-08-31  
**Origen**: Casos de uso – Diagrama "Diagrama 123"  
**Modelo**: 1 especificación = 1 caso de uso (autocontenidas según `spec-template.md`)  

Este documento es el **índice del módulo**: coordina las especificaciones atómicas, sus prioridades y dependencias. Cada especificación dentro de su subcarpeta es 100% autocontenida y autónoma.

## Mapa de Especificaciones

| ID | Nombre | Prioridad | Depende de | Especificación |
| :--- | :--- | :--- | :--- | :--- |
| M3-CU01 | Consultar Lista de Galpones | P1 | — (punto de entrada) | [m3-cu01-lista-galpones/spec.md](m3-cu01-lista-galpones/spec.md) |
| M3-CU02 | Registrar Matriz de Ventas | P1 | M3-CU01 · Módulo 1 (vaciado) · Módulo 2 (resultado final) | [m3-cu02-registrar-matriz-ventas/spec.md](m3-cu02-registrar-matriz-ventas/spec.md) |
| M3-CU03 | Generar Matriz de Ventas (liquidación) | P1 | M3-CU02 · Módulo 1 (población) · Módulo 2 (consumos) | [m3-cu03-generar-matriz-ventas/spec.md](m3-cu03-generar-matriz-ventas/spec.md) |
| M3-CU04 | Desglose de Ventas y Gastos | P2 | M3-CU03 (liquidación ACTIVA) | [m3-cu04-desglose-ventas-gastos/spec.md](m3-cu04-desglose-ventas-gastos/spec.md) |
| M3-CU05 | Consultar Historial de Reportes Financieros | P3 | M3-CU03 (≥ 1 liquidación) | [m3-cu05-historial-reportes/spec.md](m3-cu05-historial-reportes/spec.md) |

## Cadena de Dependencias

```text
CU01 ──► CU02 ──► CU03 ──► CU04
                    │
                    └─────► CU05
```

Orden sugerido de implementación: CU01 → CU02 → CU03 → (CU04 y CU05 en paralelo).

## Criterios de Aceptación Globales del Módulo

El módulo se considera ACEPTADO cuando se cumplen TODOS:

- **CA-G01**: Los 5 casos de uso operan de extremo a extremo con el rol Administrador financiero.
- **CA-G02**: Cada escenario de aceptación de los specs CU01–CU05 pasa tal cual está escrito.
- **CA-G03**: Los cálculos de al menos 3 lotes de prueba coinciden 100% con liquidación manual firmada por el stakeholder.
- **CA-G04**: Es imposible generar una segunda venta o liquidación activa para el mismo lote.
- **CA-G05**: Todo número en pantalla puede explicarse navegando hasta su registro origen.
- **CA-G06**: La exportación a Excel refleja exactamente lo mostrado en pantalla.
- **CA-G07**: Los flujos funcionan correctamente con Módulos 1 y 2 simulando: datos completos, datos faltantes y ausencia total de datos.

## Criterios de Éxito Medibles del Módulo

- **SC-001**: 100% de liquidaciones de prueba coinciden exactamente con cálculo manual verificado.
- **SC-002**: Del clic en "lista de galpones" a la liquidación en pantalla transcurren menos de 30 segundos.
- **SC-003**: Cada cifra es trazable a su origen en ≤ 3 interacciones de navegación.
- **SC-004**: 10 usuarios concurrentes completan consultas dentro de los tiempos NFR-004 sin errores.
- **SC-005**: En el primer ciclo real, el 90% de los lotes vendidos quedan liquidados sin reprocesos manuales.
- **SC-006**: Cero casos de liquidaciones modificables tras su generación; toda corrección pasa por anulación registrada.

## Trazabilidad con la Especificación Consolidada Anterior

| Elemento anterior | Ubicación actual |
| :--- | :--- |
| US1 → US5 | Specs atómicos CU01 → CU05 |
| FR-001 | CU01.RF-001 |
| FR-002 | CU05.RF-001 |
| FR-003 a FR-005 | CU02.RF-001 a RF-004 |
| FR-006 a FR-012 | CU03.RF-001 a RF-006 |
| FR-013 (anulación) | CU02.RF-006 (matriz) y CU03.RF-007 (liquidación) |
| FR-014, FR-015 | CU04.RF-001 a RF-003 |
| FR-016 | CU02.RF-005 y CU03.RF-008 |
| FR-017 (moneda) | Cada spec (COP, redondeo HALF_UP en CU03, decimales en precio/kg en CU02) |
| NFR-001 a NFR-005 | Criterios de Éxito y FRs en cada spec |
| E-01 a E-09 | Manejo de Errores y Escenarios integrados en cada spec |
| CA-01 a CA-07 | CA-G01 a CA-G07 (este documento) |
