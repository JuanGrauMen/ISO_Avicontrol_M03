# Implementation Plan: Consultar Lista de Galpones

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
**Specs**:
- [m3-cu01-lista-galpones](../m3-cu01-lista-galpones/spec.md) – Consultar Lista de Galpones  

## Summary

Permite al Administrador Financiero consultar la lista de galpones y lotes alojados sobre la copia local sincronizada desde Módulo 1. Incluye el cálculo en tiempo real del porcentaje de mortalidad acumulada `((inicial - actual) / inicial) * 100`, filtrado por estado del catálogo (o "Todos"), búsqueda textual por nombre de galpón, nombre de lote o UUID, paginación a máximo 6 galpones por página, indicador del progreso de paginación (ej. "6 de 7 galpones") y el mensaje `"No se encontraron resultados"` cuando ninguna coincidencia satisface los filtros aplicados.

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Jakarta Validation, JUnit 5, MockMvc
- **Storage**: Consulta sobre copia local de tablas `galpon` y `lote`
- **Testing**: Unit tests con Mockito y pruebas de integración REST con MockMvc
- **Target Platform**: JVM 21 / REST API
- **Project Type**: Backend REST API (caso de uso de consulta)
- **Performance Goals**: Tiempo de respuesta HTTP GET < 3s (SC-001)

---

## Snippets de Código Java de Puertos, DTOs y Servicio

### Puerto de Entrada: `ListarGalponesUseCase.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.in;

import co.edu.unimagdalena.avicontrol.application.dto.GalponPageDto;

public interface ListarGalponesUseCase {
    GalponPageDto listarGalpones(String estado, String search, int page, int size);
}
```

### DTOs de Respuesta: `GalponResumenDto.java` y `GalponPageDto.java`
```java
package co.edu.unimagdalena.avicontrol.application.dto;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

public record GalponResumenDto(
    UUID idGalpon,
    String nombreGalpon,
    String estado,
    UUID idLote,
    String nombreLote,
    LocalDate fechaIngreso,
    BigDecimal porcentajeMortalidad,
    Instant fechaHoraSync,
    List<String> accionesDisponibles
) {}

public record GalponPageDto(
    List<GalponResumenDto> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    String mensaje
) {}
```

### Servicio de Aplicación: `ListarGalponesService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.application.dto.GalponPageDto;
import co.edu.unimagdalena.avicontrol.application.dto.GalponResumenDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.ListarGalponesUseCase;
import co.edu.unimagdalena.avicontrol.domain.port.out.GalponRepository;
import co.edu.unimagdalena.avicontrol.domain.shared.Rounding;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
public class ListarGalponesService implements ListarGalponesUseCase {

    private final GalponRepository galponRepository;

    public ListarGalponesService(GalponRepository galponRepository) {
        this.galponRepository = galponRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public GalponPageDto listarGalpones(String estado, String search, int page, int size) {
        int pageSize = Math.min(size, 6); // Paginación fija máximo 6 ítems
        var resultadoPage = galponRepository.buscarYFiltrar(estado, search, page, pageSize);

        if (resultadoPage.getContent().isEmpty()) {
            return new GalponPageDto(List.of(), page, pageSize, 0, 0, "No se encontraron resultados");
        }

        var dtos = resultadoPage.getContent().stream().map(g -> {
            var mort = Rounding.percentage(
                BigDecimal.valueOf(g.getLote().getPoblacionInicial() - g.getLote().getPoblacionActual())
                    .multiply(BigDecimal.valueOf(100))
                    .divide(BigDecimal.valueOf(g.getLote().getPoblacionInicial()), 4, Rounding.HALF_UP)
            );
            return new GalponResumenDto(g.getId(), g.getNombre(), g.getEstado().name(),
                g.getLote().getId(), g.getLote().getNombre(), g.getLote().getFechaIngreso(),
                mort, g.getFechaHoraSync(), g.determinarAcciones());
        }).toList();

        return new GalponPageDto(dtos, page, pageSize, resultadoPage.getTotalElements(), resultadoPage.getTotalPages(), null);
    }
}
```

---

## Phase 1: Puertos y DTOs

- [ ] **T001** Crear puerto `ListarGalponesUseCase.java` en `domain/port/in/`.
- [ ] **T002** Crear DTOs `GalponResumenDto.java` y `GalponPageDto.java`.

---

## Phase 2: Lógica de Servicio y Cálculo de Mortalidad %

- [ ] **T003** Unit Test `ListarGalponesServiceTest.java`: verificar cálculo de porcentaje de mortalidad acumulada, filtro de estados, búsqueda por término, paginación a 6 ítems y mensaje `"No se encontraron resultados"`.
- [ ] **T004** Implementar `ListarGalponesService.java`.

---

## Phase 3: Adaptador REST (Controlador HTTP)

- [ ] **T005** Integration Test `GalponControllerTest.java` con MockMvc.
- [ ] **T006** Implementar `GalponController.java`.
