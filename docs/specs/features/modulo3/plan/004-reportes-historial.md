# Implementation Plan: Historial de Liquidaciones y Exportación de Desglose

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
**Specs**:
- [m3-cu05-desglose-ventas-gastos](../m3-cu05-desglose-ventas-gastos/spec.md) – Consultar Desglose de Ventas y Gastos  
- [m3-cu06-historial-liquidaciones](../m3-cu06-historial-liquidaciones/spec.md) – Consultar Historial de Liquidaciones  

## Summary

Este plan aborda la **consulta histórica cronológica** de liquidaciones (`CU06`) y la **visualización y exportación en Excel del desglose pormenorizado de ventas y gastos** (`CU05`).

**Historial (CU06)**: Permite al Administrador Financiero filtrar liquidaciones por galpón y rango de fechas (orden descendente). Al seleccionar cualquier fila, el sistema abre la **vista de la Liquidación** (M3-CU03.FR-014) — no el desglose directamente. El Desglose se consulta desde la propia vista de la Liquidación (CU06.FR-006).

**Desglose (CU05)**: Visualización agrupada por categorías de costo — **Alimento**, **Insumos Médicos** y **Costo de Población** (renombrada desde "Población Inicial" para alinear con CU03.FR-003 y CU07) — garantizando que los subtotales cuadran exactamente con `costosOperativosCop` de la Liquidación (FR-002). Exportación a `.xlsx` con Apache POI (`poi-ooxml:5.2.5`).

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Apache POI `poi-ooxml:5.2.5`, JUnit 5, MockMvc
- **Storage**: Proyección y lectura sobre tablas `liquidacion`, `partida_costo_lote`, `registro_anulacion`
- **Testing**: Unit tests de lógica de proyección/agrupación, tests de generación de Excel con Apache POI y tests REST
- **Performance Goals**: Generación y descarga de archivo Excel < 10s (SC-002); consulta de historial < 3s (SC-001)

---

## Snippets de Código Java de Puertos, DTOs, Servicio Apache POI y Controlador

### Puertos de Entrada
```java
package co.edu.unimagdalena.avicontrol.domain.port.in;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.DesgloseDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.dto.HistorialPageDto;
import java.time.LocalDate;
import java.util.UUID;

/** CU06 — Consultar historial cronológico filtrado */
public interface ConsultarHistorialUseCase {
    HistorialPageDto consultarHistorial(UUID idGalpon, LocalDate fechaInicio, LocalDate fechaFin, int page, int size);
}

/** CU05 — Consultar proyección de desglose de ventas y gastos */
public interface ConsultarDesgloseUseCase {
    DesgloseDto consultarDesglose(Long idLiquidacion);
}

/** CU05 — Generar y descargar reporte Excel (.xlsx) */
public interface ExportarExcelUseCase {
    byte[] generarReporteExcelLiquidacion(Long idLiquidacion);
}
```

### DTOs de Respuesta: `HistorialPageDto.java` y `DesgloseDto.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.in.dto;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record HistorialItemDto(
    Long idLiquidacion,
    Instant fechaHoraLiquidacion,
    UUID idGalpon,
    String nombreGalpon,
    UUID idLote,
    String nombreLote,
    long ventaBrutaCop,
    BigDecimal porcentajeMortalidad,
    long costosOperativosCop,
    long utilidadNetaCop,
    String estado                 // "ACTIVA" | "ANULADA"
) {}

public record HistorialPageDto(
    List<HistorialItemDto> content,
    int page,
    int size,
    long totalElements,
    int totalPages
) {}

public record PartidaDesgloseDto(
    String concepto,
    BigDecimal cantidad,
    String unidadMedida,
    BigDecimal precioUnitarioCop,
    long subtotalCop,
    String referenciaOrigen
) {}

public record DesgloseDto(
    Long idLiquidacion,
    String estado,
    boolean esSiniestroTotal,
    List<PartidaDesgloseDto> partidasAlimento,
    List<PartidaDesgloseDto> partidasInsumosMedicos,
    List<PartidaDesgloseDto> partidasCostoPoblacion,
    long totalCostosOperativosCop,
    long ventaBrutaCop,
    long utilidadNetaCop
) {}
```

### Servicio Apache POI: `ExportarExcelService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.domain.port.in.ExportarExcelUseCase;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.stereotype.Service;

import java.io.ByteArrayOutputStream;

@Service
public class ExportarExcelService implements ExportarExcelUseCase {

    @Override
    public byte[] generarReporteExcelLiquidacion(Long idLiquidacion) {
        try (Workbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream out = new ByteArrayOutputStream()) {

            Sheet sheet = workbook.createSheet("Liquidacion_Lote");

            // Estilo de celda para moneda COP ($#,##0)
            CellStyle copStyle = workbook.createCellStyle();
            DataFormat format = workbook.createDataFormat();
            copStyle.setDataFormat(format.getFormat("$#,##0"));

            // Encabezados y Matriz de Venta
            Row headerRow = sheet.createRow(0);
            headerRow.createCell(0).setCellValue("Concepto");
            headerRow.createCell(1).setCellValue("Subtotal (COP)");

            // Partidas y fórmula de suma nativa Excel
            Row totalRow = sheet.createRow(10);
            totalRow.createCell(0).setCellValue("TOTAL COSTOS OPERATIVOS");
            Cell totalCell = totalRow.createCell(1);
            totalCell.setCellFormula("SUM(B2:B9)");
            totalCell.setCellStyle(copStyle);

            workbook.write(out);
            return out.toByteArray();
        } catch (Exception e) {
            throw new RuntimeException("Error al generar reporte Excel POI", e);
        }
    }
}
```

### Adaptador REST con Cabeceras `.xlsx`: `DesgloseController.java`
```java
package co.edu.unimagdalena.avicontrol.infrastructure.adapter.rest;

import co.edu.unimagdalena.avicontrol.domain.port.in.ConsultarDesgloseUseCase;
import co.edu.unimagdalena.avicontrol.domain.port.in.ExportarExcelUseCase;
import co.edu.unimagdalena.avicontrol.domain.port.in.dto.DesgloseDto;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/liquidaciones")
public class DesgloseController {

    private final ConsultarDesgloseUseCase consultarDesgloseUseCase;
    private final ExportarExcelUseCase exportarExcelUseCase;

    public DesgloseController(ConsultarDesgloseUseCase consultarDesgloseUseCase,
                              ExportarExcelUseCase exportarExcelUseCase) {
        this.consultarDesgloseUseCase = consultarDesgloseUseCase;
        this.exportarExcelUseCase = exportarExcelUseCase;
    }

    @GetMapping("/{id}/desglose")
    public ResponseEntity<DesgloseDto> consultarDesglose(@PathVariable Long id) {
        return ResponseEntity.ok(consultarDesgloseUseCase.consultarDesglose(id));
    }

    @GetMapping("/{id}/desglose/excel")
    public ResponseEntity<byte[]> descargarExcelDesglose(@PathVariable Long id) {
        byte[] bytes = exportarExcelUseCase.generarReporteExcelLiquidacion(id);

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"Liquidacion_Lote_" + id + ".xlsx\"")
            .contentType(MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .body(bytes);
    }
}
```

---

## Phase 1: CU06 – Historial Cronológico de Liquidaciones

- [ ] **T001** Crear puerto `ConsultarHistorialUseCase.java` en `domain/port/in/` y DTOs `HistorialItemDto.java` y `HistorialPageDto.java` en `domain/port/in/dto/`.
- [ ] **T002** Unit Test `ConsultarHistorialServiceTest.java`.
- [ ] **T003** Implementar `ConsultarHistorialService.java`.
- [ ] **T004** Integration Test e implementación de `GET /api/v1/liquidaciones` en `HistorialController.java`; verificar que cada fila incluye el `idLiquidacion` que sirve de enlace a la vista de Liquidación (M3-CU03.FR-014), no al desglose directamente.

---

## Phase 2: CU05 – Consultar Desglose de Ventas y Gastos

- [ ] **T005** Crear puerto `ConsultarDesgloseUseCase.java` en `domain/port/in/` y DTO `DesgloseDto.java` en `domain/port/in/dto/`.
- [ ] **T006** Unit Test `ConsultarDesgloseServiceTest.java`.
- [ ] **T007** Implementar `ConsultarDesgloseService.java`.
- [ ] **T008** Integration Test e implementación de `GET /api/v1/liquidaciones/{id}/desglose` en `DesgloseController.java`.

---

## Phase 3: CU05 – Generación y Exportación de Reporte Excel (Apache POI)

- [ ] **T009** Unit Test `ExportarExcelServiceTest.java`.
- [ ] **T010** Implementar `ExportarExcelService.java`.
- [ ] **T011** Integration Test e implementación de `GET /api/v1/liquidaciones/{id}/desglose/excel`.
