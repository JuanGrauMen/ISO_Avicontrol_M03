# Implementation Plan: Consultar Lista de Lotes

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
**Specs**:
- [m3-cu01-lista-lotes](../m3-cu01-lista-lotes/spec.md) – Consultar Lista de Lotes  

## Summary

Permite al Administrador Financiero consultar la **lista de lotes** alojados en galpones, alimentada exclusivamente desde la copia local sincronizada de M3-CU07. Cada fila muestra UUID y nombre del lote, fecha de ingreso, nombre y UUID del galpón, la **etapa** derivada (FR-013) y la fecha de última sincronización. **No se muestra columna de mortalidad** en la lista (la mortalidad se presenta en la Liquidación).

Cuando la población actual sincronizada del lote sea 0 por mortalidad total, la fila muestra el distintivo **"Siniestro total"** junto a su etapa (FR-001). Cada fila expone **un único botón** determinado por la etapa:
- `Por Liquidar` → "Generar liquidación" (habilitado si hay resultado de sacrificio válido o siniestro total confirmado).
- `Liquidado` → "Ver liquidación" (habilitado, abre M3-CU03.FR-014).
- `Productivo`, `En Cosecha`, `Aislamiento` → "Generar liquidación" (deshabilitado).

Incluye filtrado por etapa (una etapa o "Todos"), búsqueda por nombre de lote, nombre de galpón o UUID, paginación a máximo 6 lotes por página e indicador de progreso (ej. "6 de 7 lotes").

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Jakarta Validation, JUnit 5, MockMvc
- **Storage**: Consulta sobre copia local de tablas `galpon` y `lote` (mantenidas por M3-CU07)
- **Testing**: Unit tests con Mockito y pruebas de integración REST con MockMvc
- **Target Platform**: JVM 21 / REST API
- **Project Type**: Backend REST API (caso de uso de consulta)
- **Performance Goals**: Tiempo de respuesta HTTP GET < 3s (SC-001)

---

## Etapas y Regla de Derivación (FR-013)

La **etapa** del lote es un valor derivado por M3 (nunca almacenado ni comunicado a M1), calculada aplicando las siguientes reglas en orden; la primera que se cumple determina la etapa:

| Prioridad | Condición | Etapa derivada |
|---|---|---|
| 1 | Existe alerta de vaciado sanitario **y** Liquidación `ACTIVA` del lote | `Liquidado` |
| 2 | Existe alerta de vaciado sanitario **y** no existe Liquidación `ACTIVA` | `Por Liquidar` |
| 3 | Lote vinculado a galpón en estado `Productivo` | `Productivo` |
| 4 | Lote vinculado a galpón en estado `En Cosecha` | `En Cosecha` |
| 5 | Lote vinculado a galpón en estado `Aislamiento` | `Aislamiento` |

Un galpón sin lote vinculado ni alerta pendiente no genera fila.

---

## JSON Payload de Ejemplo (Respuesta Paginada)

```json
GET /api/v1/lotes?etapa=Por Liquidar&search=lote-norte&page=0&size=6

HTTP/1.1 200 OK
{
  "content": [
    {
      "idLote": "a1b2c3d4-...",
      "nombreLote": "Lote Norte Ciclo 4",
      "fechaIngreso": "2026-06-01",
      "idGalpon": "f1e2d3c4-...",
      "nombreGalpon": "Galpón Norte",
      "etapa": "Por Liquidar",
      "siniestroTotal": false,
      "accion": "GENERAR_LIQUIDACION",
      "accionHabilitada": true,
      "fechaHoraUltimaSincronizacion": "2026-10-03T10:00:00Z"
    },
    {
      "idLote": "b2c3d4e5-...",
      "nombreLote": "Lote Sur Ciclo 3",
      "fechaIngreso": "2026-05-15",
      "idGalpon": "e2d3c4b5-...",
      "nombreGalpon": "Galpón Sur",
      "etapa": "Por Liquidar",
      "siniestroTotal": true,
      "accion": "GENERAR_LIQUIDACION",
      "accionHabilitada": true,
      "fechaHoraUltimaSincronizacion": "2026-10-03T09:50:00Z"
    }
  ],
  "page": 0,
  "size": 6,
  "totalElements": 2,
  "totalPages": 1,
  "mensajePaginacion": "2 de 2 lotes"
}
```

---

## Snippets de Código Java de Puertos, DTOs y Servicio

### Puerto de Entrada: `ListarLotesUseCase.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.in;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LotePageDto;

public interface ListarLotesUseCase {
    /**
     * @param etapa  una de las 5 etapas derivadas o null/"Todos" para todas
     * @param search texto libre: nombre de lote, nombre de galpón o UUID
     * @param page   índice de página (0-based)
     * @param size   tamaño máximo de página (se limita a 6 en el servicio)
     */
    LotePageDto listarLotes(String etapa, String search, int page, int size);
}
```

### Puerto de Salida: `LoteRepository.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.out;

import co.edu.unimagdalena.avicontrol.domain.model.Lote;
import org.springframework.data.domain.Page;

public interface LoteRepository {
    /**
     * Consulta lotes filtrando por etapa derivada (o todas si es null/"Todos")
     * y buscando por nombre de lote, galpón o UUID.
     */
    Page<Lote> buscarYFiltrarPorEtapa(String etapa, String search, int page, int size);
}
```

### DTOs de Respuesta: `LoteResumenDto.java` y `LotePageDto.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.in.dto;

import java.time.Instant;
import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

/**
 * Proyección de solo lectura por fila de la lista de lotes (M3-CU01).
 * No incluye mortalidad (la mortalidad se muestra en la Liquidación).
 */
public record LoteResumenDto(
    UUID idLote,
    String nombreLote,
    LocalDate fechaIngreso,
    UUID idGalpon,
    String nombreGalpon,
    String etapa,              // "Productivo" | "En Cosecha" | "Aislamiento" | "Por Liquidar" | "Liquidado"
    boolean siniestroTotal,    // true cuando poblacionActual == 0 por mortalidad total
    String accion,             // "GENERAR_LIQUIDACION" | "VER_LIQUIDACION"
    boolean accionHabilitada,  // false cuando la etapa no permite la acción
    Instant fechaHoraUltimaSincronizacion
) {}

public record LotePageDto(
    List<LoteResumenDto> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    String mensajePaginacion   // ej. "6 de 7 lotes" o "No se encontraron resultados"
) {}
```

### Servicio de Aplicación: `ListarLotesService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LotePageDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LoteResumenDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.ListarLotesUseCase;
import co.edu.unimagdalena.avicontrol.domain.port.out.LoteRepository;
import co.edu.unimagdalena.avicontrol.domain.port.out.LiquidacionRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ListarLotesService implements ListarLotesUseCase {

    private final LoteRepository loteRepository;
    private final LiquidacionRepository liquidacionRepository;

    public ListarLotesService(LoteRepository loteRepository,
                              LiquidacionRepository liquidacionRepository) {
        this.loteRepository = loteRepository;
        this.liquidacionRepository = liquidacionRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public LotePageDto listarLotes(String etapa, String search, int page, int size) {
        int pageSize = Math.min(size, 6); // FR-011: máximo 6 lotes por página

        var resultadoPage = loteRepository.buscarYFiltrarPorEtapa(etapa, search, page, pageSize);

        if (resultadoPage.getContent().isEmpty()) {
            String msg = (resultadoPage.getTotalElements() == 0
                          && (etapa == null || "Todos".equals(etapa))
                          && (search == null || search.isBlank()))
                ? "No hay lotes registrados"
                : "No se encontraron resultados";
            return new LotePageDto(List.of(), page, pageSize, 0, 0, msg);
        }

        var dtos = resultadoPage.getContent().stream().map(lote -> {
            // Derivar etapa (FR-013)
            String etapaDerivada = derivarEtapa(lote);
            boolean esSiniestro = lote.getPoblacionActual() != null && lote.getPoblacionActual() == 0;

            // Determinar acción única según etapa (FR-005)
            String accion;
            boolean habilitada;
            if ("Liquidado".equals(etapaDerivada)) {
                accion = "VER_LIQUIDACION";
                habilitada = true;
            } else if ("Por Liquidar".equals(etapaDerivada)) {
                accion = "GENERAR_LIQUIDACION";
                // Habilitada si hay resultado de sacrificio válido o es siniestro total
                habilitada = lote.tieneResultadoSacrificio() || esSiniestro;
            } else {
                // Productivo, En Cosecha, Aislamiento: botón presente pero deshabilitado
                accion = "GENERAR_LIQUIDACION";
                habilitada = false;
            }

            return new LoteResumenDto(
                lote.getId(), lote.getNombre(), lote.getFechaIngreso(),
                lote.getIdGalpon(), lote.getNombreGalpon(),
                etapaDerivada, esSiniestro,
                accion, habilitada,
                lote.getFechaHoraUltimaSincronizacion()
            );
        }).toList();

        long total = resultadoPage.getTotalElements();
        int shown = dtos.size();
        String paginacion = shown + " de " + total + " lotes";
        return new LotePageDto(dtos, page, pageSize, total, resultadoPage.getTotalPages(), paginacion);
    }

    /** FR-013: reglas en orden; la primera que se cumple determina la etapa. */
    private String derivarEtapa(Lote lote) {
        boolean tieneAlerta = lote.getAlertaVaciadoSanitario() != null;
        boolean tieneActiva = liquidacionRepository.existeActivaPorLote(lote.getId());

        if (tieneAlerta && tieneActiva) return "Liquidado";
        if (tieneAlerta)               return "Por Liquidar";

        return switch (lote.getEstadoGalpon()) {
            case "Productivo"   -> "Productivo";
            case "En Cosecha"   -> "En Cosecha";
            case "Aislamiento"  -> "Aislamiento";
            default             -> "Productivo"; // fallback seguro
        };
    }
}
```

### Controlador REST: `LoteController.java`
```java
package co.edu.unimagdalena.avicontrol.infrastructure.adapter.rest;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LotePageDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.ListarLotesUseCase;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/lotes")
public class LoteController {

    private final ListarLotesUseCase listarLotesUseCase;

    public LoteController(ListarLotesUseCase listarLotesUseCase) {
        this.listarLotesUseCase = listarLotesUseCase;
    }

    /**
     * GET /api/v1/lotes?etapa=Por Liquidar&search=norte&page=0&size=6
     */
    @GetMapping
    public ResponseEntity<LotePageDto> listarLotes(
            @RequestParam(required = false) String etapa,
            @RequestParam(required = false) String search,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "6") int size) {
        return ResponseEntity.ok(listarLotesUseCase.listarLotes(etapa, search, page, size));
    }
}
```

---

## Phase 1: Puertos y DTOs

- [ ] **T001** Crear puerto `ListarLotesUseCase.java` en `domain/port/in/`.
- [ ] **T002** Crear DTOs `LoteResumenDto.java` y `LotePageDto.java` en `domain/port/in/dto/`.

---

## Phase 2: Lógica de Servicio — Derivación de Etapa y Acción Única

- [ ] **T003** Unit Test `ListarLotesServiceTest.java`: verificar derivación de etapa (FR-013) para los 5 casos, distintivo `siniestroTotal`, botón habilitado/deshabilitado, filtro de etapas, búsqueda por término, paginación a 6 ítems, mensajes de vacío total vs. sin resultados, y mensaje de paginación (ej. "6 de 7 lotes").
- [ ] **T004** Implementar `ListarLotesService.java`.

---

## Phase 3: Adaptador REST (Controlador HTTP)

- [ ] **T005** Integration Test `LoteControllerTest.java` con MockMvc: verificar parámetros de query, respuesta paginada y formato JSON de `LoteResumenDto`.
- [ ] **T006** Implementar `LoteController.java`.
