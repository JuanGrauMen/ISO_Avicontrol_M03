# Implementation Plan: Historial de Liquidaciones y Exportación de Desglose

**Date**: 2026-09-28  
**Specs**:
- [m3-cu05-desglose-ventas-gastos](../m3-cu05-desglose-ventas-gastos/spec.md) – Consultar Desglose de Ventas y Gastos  
- [m3-cu06-historial-liquidaciones](../m3-cu06-historial-liquidaciones/spec.md) – Consultar Historial de Liquidaciones  

## Summary

Este plan aborda la **consulta histórica cronológica** de liquidaciones (`CU06`) y la **visualización y exportación en Excel del desglose pormenorizado de ventas y gastos** (`CU05`). Permite al Administrador Financiero filtrar el historial por galpón y rango de fechas (orden descendente por fecha de generación), consultar el desglose agrupado por categorías de costo (alimento, medicina, población) garantizando que los subtotales cuadran exactamente con los Costos Operativos de la liquidación, y generar y descargar el reporte `.xlsx` formateado mediante Apache POI (`poi-ooxml:5.2.5`).

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Apache POI `poi-ooxml:5.2.5`, JUnit 5, MockMvc
- **Storage**: Proyección y lectura sobre tablas `liquidacion`, `partida_costo_lote`, `registro_anulacion`
- **Testing**: Unit tests de lógica de proyección/agrupación, tests de generación de Excel con Apache POI y tests REST
- **Target Platform**: JVM 21 / REST API / Archivo binario `.xlsx`
- **Project Type**: Backend Service & Report Generator
- **Performance Goals**: Generación y descarga de archivo Excel < 10s (SC-002); consulta de historial < 3s (SC-001)
- **Constraints**: Formato monetario COP sin decimales en Excel; fórmulas de suma automáticas; lectura de liquidaciones anuladas preservando el estado original.
- **Scale/Scope**: Decenas de liquidaciones históricas por año.

## Project Structure

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   └── port/in/                        # ConsultarHistorialUseCase.java, ConsultarDesgloseUseCase.java, ExportarExcelUseCase.java
├── application/
│   ├── service/                        # ConsultarHistorialService.java, ConsultarDesgloseService.java, ExportarExcelService.java
│   └── dto/                            # DesgloseDto.java, HistorialFiltroDto.java
└── infrastructure/
    └── adapter/rest/                   # HistorialController.java, DesgloseController.java
```

## Phase 1: CU06 – Historial Cronológico de Liquidaciones

**Purpose**: Consulta filtrada y ordenada del historial de liquidaciones activas y anuladas.

- [ ] **T001** Crear puerto `ConsultarHistorialUseCase.java` y DTO `HistorialFiltroDto.java`.
- [ ] **T002** Unit Test `ConsultarHistorialServiceTest.java`: verificación de orden descendente por fecha de generación, filtro combinado por galpón y fechas, validación de rango de fechas no invertido y respuesta ante vacío.
- [ ] **T003** Implementar `ConsultarHistorialService.java`.
- [ ] **T004** Integration Test e implementación de `GET /api/v1/liquidaciones` en `HistorialController.java`.

---

## Phase 2: CU05 – Consultar Desglose de Ventas y Gastos

**Purpose**: Proyección detallada de partidas de costo agrupadas por categoría.

- [ ] **T005** Crear puerto `ConsultarDesgloseUseCase.java` y DTO `DesgloseDto.java`.
- [ ] **T006** Unit Test `ConsultarDesgloseServiceTest.java`: verificar que la suma de subtotales agrupados por categoría cuadra al peso con el indicador Costos Operativos de la liquidación y soporte para liquidaciones anuladas.
- [ ] **T007** Implementar `ConsultarDesgloseService.java`.
- [ ] **T008** Integration Test e implementación de `GET /api/v1/liquidaciones/{id}/desglose` en `DesgloseController.java`.

---

## Phase 3: CU05 – Generación y Exportación de Reporte Excel (Apache POI)

**Purpose**: Generar el documento Excel `.xlsx` con la Matriz de Venta Final y el desglose de costos.

- [ ] **T009** Unit Test `ExportarExcelServiceTest.java`: verificar construcción del libro de trabajo Excel, estilos de celda monetarios COP, encabezados y fórmulas de suma de Apache POI.
- [ ] **T010** Implementar `ExportarExcelService.java` en `application/service/`.
- [ ] **T011** Integration Test e implementación de `GET /api/v1/liquidaciones/{id}/desglose/excel` con `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` en `DesgloseController.java`.
