# Implementation Plan: Historial de Liquidaciones y Exportación de Desglose

**Date**: 2026-09-28  
**Actualizado**: 2026-10-02  
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

---

## Especificación de Endpoints REST y Payloads JSON

### A. Endpoint: `GET /api/v1/liquidaciones/{id}/desglose` (Desglose Pormenorizado)

#### Ejemplo de Response Body (`200 OK`)
```json
{
  "idLiquidacion": 1045,
  "idLote": "98765432-e89b-12d3-a456-426614174000",
  "nombreLote": "Lote L-2026-A",
  "estadoLiquidacion": "ACTIVA",
  "resumenMatrizVenta": {
    "pollosVendidos": 8500,
    "pesoTotalKg": 23800.00,
    "precioKgCop": 4500.00,
    "ventaBrutaCop": 107100000,
    "costosOperativosCop": 85000000,
    "utilidadNetaCop": 22100000
  },
  "partidasCategorizadas": [
    {
      "categoria": "POBLACION",
      "subtotalCategoriaCop": 18000000,
      "items": [
        {
          "concepto": "Costo Inicial Lote Pollitos BB (9000 aves)",
          "cantidad": 9000.000,
          "unidadMedida": "AVES",
          "precioUnitarioCop": 2000.00,
          "subtotalCop": 18000000,
          "fuenteOrigen": "MODULO_1"
        }
      ]
    },
    {
      "categoria": "ALIMENTO",
      "subtotalCategoriaCop": 55000000,
      "items": [
        {
          "concepto": "Alimento Iniciador Fase 1",
          "cantidad": 12000.000,
          "unidadMedida": "KG",
          "precioUnitarioCop": 2500.00,
          "subtotalCop": 30000000,
          "fuenteOrigen": "MODULO_2"
        },
        {
          "concepto": "Alimento Engorde Fase 2",
          "cantidad": 10000.000,
          "unidadMedida": "KG",
          "precioUnitarioCop": 2500.00,
          "subtotalCop": 25000000,
          "fuenteOrigen": "MODULO_2"
        }
      ]
    },
    {
      "categoria": "MEDICINA",
      "subtotalCategoriaCop": 12000000,
      "items": [
        {
          "concepto": "Vacuna Gumboro + Newcastle",
          "cantidad": 9000.000,
          "unidadMedida": "DOSIS",
          "precioUnitarioCop": 1333.33,
          "subtotalCop": 12000000,
          "fuenteOrigen": "MODULO_2"
        }
      ]
    }
  ],
  "totalCostosCalculadoCop": 85000000
}
```

---

### B. Endpoint: `GET /api/v1/liquidaciones/{id}/desglose/excel` (Descarga Excel)

#### Cabeceras HTTP de Respuesta
- **`Content-Type`**: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- **`Content-Disposition`**: `attachment; filename="Liquidacion_Lote_L-2026-A.xlsx"`

---

### C. Endpoint: `GET /api/v1/liquidaciones` (Historial Cronológico)

#### Query Params
- **`galpon`** (`UUID`, Opcional): Filtrar por galpón.
- **`desde`** (`DATE`, Opcional): Fecha inicio (`YYYY-MM-DD`).
- **`hasta`** (`DATE`, Opcional): Fecha fin (`YYYY-MM-DD`).

#### Ejemplo de Response Body (`200 OK`)
```json
{
  "content": [
    {
      "idLiquidacion": 1045,
      "idLote": "98765432-e89b-12d3-a456-426614174000",
      "nombreLote": "Lote L-2026-A",
      "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
      "nombreGalpon": "Galpón 1",
      "ventaBrutaCop": 107100000,
      "costosOperativosCop": 85000000,
      "utilidadNetaCop": 22100000,
      "estado": "ACTIVA",
      "fechaHoraGeneracion": "2026-10-02T15:30:00Z",
      "usuarioResponsable": "financiero@avicontrol.edu.co"
    }
  ],
  "totalElements": 1
}
```

---

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
