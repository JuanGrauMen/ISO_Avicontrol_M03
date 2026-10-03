# Implementation Plan: Consultar Lista de Galpones

**Date**: 2026-09-28  
**Actualizado**: 2026-10-02  
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
- **Constraints**: No consulta en vivo a Módulo 1; datos extraídos estrictamente de la copia local sincronizada. Paginación fija a máximo 6 ítems por página.

---

## Especificación del Endpoint REST y Payloads JSON

### Endpoint: `GET /api/v1/galpones`

#### Parámetros de Consulta (Query Params)
- **`estado`** (`String`, Opcional): Filtrar por catálogo (`VACIADO_SANITARIO`, `PRODUCTIVO`, `DISPONIBLE`, etc. o vacíos para todos).
- **`search`** (`String`, Opcional): Término de búsqueda por nombre de galpón, nombre de lote o UUID.
- **`page`** (`int`, Opcional, Default: `0`): Número de página (0-indexed).
- **`size`** (`int`, Opcional, Default: `6`): Tamaño fijo de página (máximo 6 galpones por página).

#### Ejemplo de Respuesta Exitosa (`200 OK`)
```json
{
  "content": [
    {
      "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
      "nombreGalpon": "Galpón 1",
      "estado": "VACIADO_SANITARIO",
      "idLote": "98765432-e89b-12d3-a456-426614174000",
      "nombreLote": "Lote L-2026-A",
      "fechaIngreso": "2026-08-01",
      "porcentajeMortalidad": 5.56,
      "fechaHoraSync": "2026-10-02T14:00:00Z",
      "accionesDisponibles": ["LIQUIDAR", "VER_HISTORIAL"]
    },
    {
      "idGalpon": "223e4567-e89b-12d3-a456-426614174001",
      "nombreGalpon": "Galpón 2",
      "estado": "PRODUCTIVO",
      "idLote": "88765432-e89b-12d3-a456-426614174001",
      "nombreLote": "Lote L-2026-B",
      "fechaIngreso": "2026-08-15",
      "porcentajeMortalidad": 2.10,
      "fechaHoraSync": "2026-10-02T14:00:00Z",
      "accionesDisponibles": ["VER_HISTORIAL"]
    }
  ],
  "page": 0,
  "size": 6,
  "totalElements": 7,
  "totalPages": 2,
  "mensaje": null
}
```

#### Ejemplo de Respuesta Sin Coincidencias (`200 OK`)
```json
{
  "content": [],
  "page": 0,
  "size": 6,
  "totalElements": 0,
  "totalPages": 0,
  "mensaje": "No se encontraron resultados"
}
```

---

## Phase 1: Puertos y DTOs

**Purpose**: Definir la interfaz del caso de uso y la estructura de datos de respuesta paginada.

- [ ] **T001** Crear puerto `ListarGalponesUseCase.java` en `domain/port/in/`.
- [ ] **T002** Crear DTOs `GalponResumenDto.java` (campos id, nombre, estado, porcentajeMortalidad, accionesDisponibles) y `GalponPageDto.java` (content, page, size, totalElements, totalPages, mensaje).

---

## Phase 2: Lógica de Servicio y Cálculo de Mortalidad %

**Purpose**: Implementar el filtrado, búsqueda, paginación (6 por pág) y cálculo de porcentaje de mortalidad.

- [ ] **T003** Unit Test `ListarGalponesServiceTest.java`: verificar cálculo de porcentaje de mortalidad acumulada `((inicial - actual) / inicial) * 100`, filtrado por catálogo de estados, búsqueda por término (nombre/UUID), paginación a 6 ítems, reinicio de página al cambiar filtro y mensaje `"No se encontraron resultados"` si no existen coincidencias.
- [ ] **T004** Implementar `ListarGalponesService.java` en `application/service/`.

---

## Phase 3: Adaptador REST (Controlador HTTP)

**Purpose**: Exponer el endpoint GET `/api/v1/galpones` con parámetros de consulta.

- [ ] **T005** Integration Test `GalponControllerTest.java`: probar `GET /api/v1/galpones?estado={estado}&search={q}&page=0&size=6` con MockMvc.
- [ ] **T006** Implementar `GalponController.java` en `infrastructure/adapter/rest/`.
