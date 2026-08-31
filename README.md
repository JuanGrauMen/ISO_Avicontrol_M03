# AVICONTROL — Módulo 3: Liquidación de Lote y Análisis de Rentabilidad

Sistema para la consolidación de ventas, cálculo de costos operativos, liquidación definitiva de lotes y análisis de rentabilidad financiera por ciclo productivo avícola.

## Información Académica

- **Institución:** Universidad del Magdalena
- **Programa:** Ingeniería de Sistemas
- **Asignatura:** Ingeniería de Software (Grupo 2)
- **Docente:** Ing. Juan Manuel Rodríguez Pineda

## Integrantes del Grupo

- Juan Grau
- Jorge Meléndez
- Andrés Lara
- Daniela Jiménez
- Juan Muñoz

---

## Alcance del Módulo 3

El Módulo 3 se encarga de determinar la rentabilidad real de cada lote avícola al cierre de su ciclo de producción.

### Casos de Uso del Módulo:

1. **M3-CU01 – Consultar Lista de Galpones (P1):** Consulta del estado operativo y población viva actual de los galpones (integración oficial con Módulo 1).
2. **M3-CU02 – Registrar Matriz de Ventas (P1):** Registro único y definitivo de comercialización de aves (pollos vendidos, peso promedio y precio por kg en COP).
3. **M3-CU03 – Generar Matriz de Ventas / Liquidación (P1):** Cálculo de Venta Bruta, Pérdida por Mortalidad, Costos Operativos (alimento, medicina, compra pollitos) y Utilidad Neta.
4. **M3-CU04 – Desglose de Ventas y Gastos (P2):** Detalle auditado por partida/categoría y exportación a formato Excel (.xlsx).
5. **M3-CU05 – Consultar Historial de Reportes Financieros (P3):** Consulta histórica con filtros por fecha y galpón para análisis comparativo entre ciclos.

---

## Metodología de Desarrollo

Este proyecto sigue el marco de **Specification-Driven Development (SDD)**:
1. **Fase 1 (SPEC):** Definición formal de requerimientos, escenarios *Given/When/Then* y criterios de aceptación sin detalles de implementación (`specs/features/modulo3/`).
2. **Fase 2 (PLAN):** Diseño técnico de arquitectura, modelos y contratos de integración.
3. **Fase 3 (Implementación):** Desarrollo en Java 17, Maven y pruebas unitarias con JUnit 5.
