# Implementation Plan: Gestión del Ciclo de Vida de Liquidaciones

**Date**: 2026-09-28  
**Actualizado**: 2026-10-02  
**Specs**:
- [m3-cu03-generar-liquidacion](../m3-cu03-generar-liquidacion/spec.md) – Generar Liquidación del Lote  
- [m3-cu04-anular-liquidacion](../m3-cu04-anular-liquidacion/spec.md) – Anular Liquidación  

## Summary

Este plan cubre el núcleo de dominio del Módulo 3: la **generación inmutable de la Liquidación** de un lote (`CU03`) y su **anulación auditable** (`CU04`). Calcula automáticamente Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta aplicando redondeo `HALF_UP` a enteros COP y 2 decimales para porcentajes. Permite liquidar el Caso Dorado, siniestros totales por mortalidad del 100% y bloquea la creación de segundas liquidaciones activas simultáneas sobre el mismo lote (`CONSTRAINT uq_lote_activa`). La anulación (`PUT /api/v1/liquidaciones/{id}/anulacion`) requiere un motivo justificado (10-500 caracteres), preserva el registro histórico sin borrar filas y rehabilita el lote para correcciones.

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Jakarta Validation, Flyway (`V3__create_liquidacion_tables.sql`), JUnit 5, MockMvc
- **Storage**: PostgreSQL / H2 en memoria (`liquidacion`, `partida_costo_lote`, `snapshot_datos_origen`, `registro_anulacion`)
- **Testing**: Unit tests exhaustivos del cálculo financiero y pruebas de integración REST end-to-end
- **Target Platform**: JVM 21 / REST API
- **Project Type**: Core Domain & Service (motor de cálculo financiero y auditoría)
- **Performance Goals**: Generación de liquidación < 5s (SC-002)
- **Constraints**: Regla `HALF_UP` en enteros COP (RT-01); 2 decimales en porcentajes (RT-02); inmutabilidad absoluta de la liquidación activa; bloqueo de segunda liquidación activa por lote.

---

## Especificación de Endpoints REST y Payloads JSON

### A. Endpoint: `POST /api/v1/liquidaciones` (Generar Liquidación)

#### Request Body (Caso Dorado)
```json
{
  "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
  "idLote": "98765432-e89b-12d3-a456-426614174000",
  "precioKgCop": 4500.00
}
```

#### Response Body (`201 Created` - Caso Dorado)
```json
{
  "idLiquidacion": 1045,
  "idLote": "98765432-e89b-12d3-a456-426614174000",
  "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
  "idResultadoSacrificio": "456e7890-e89b-12d3-a456-426614174000",
  "pollosVendidos": 8500,
  "pesoTotalKg": 23800.00,
  "pesoPromedioKg": 2.8000,
  "precioKgCop": 4500.00,
  "ventaBrutaCop": 107100000,
  "mortalidadAves": 500,
  "porcentajeMortalidad": 5.56,
  "costosOperativosCop": 85000000,
  "utilidadNetaCop": 22100000,
  "estado": "ACTIVA",
  "fechaHoraGeneracion": "2026-10-02T15:30:00Z",
  "usuarioResponsable": "financiero@avicontrol.edu.co"
}
```

#### Response Body (`201 Created` - Siniestro Total / 100% Mortalidad)
```json
{
  "idLiquidacion": 1046,
  "idLote": "77765432-e89b-12d3-a456-426614174000",
  "idGalpon": "323e4567-e89b-12d3-a456-426614174000",
  "idResultadoSacrificio": null,
  "pollosVendidos": 0,
  "pesoTotalKg": 0.00,
  "pesoPromedioKg": 0.0000,
  "precioKgCop": 0.00,
  "ventaBrutaCop": 0,
  "mortalidadAves": 9000,
  "porcentajeMortalidad": 100.00,
  "costosOperativosCop": 40000000,
  "utilidadNetaCop": -40000000,
  "estado": "ACTIVA",
  "fechaHoraGeneracion": "2026-10-02T15:45:00Z",
  "usuarioResponsable": "financiero@avicontrol.edu.co"
}
```

#### Response Error (`409 Conflict` - Intento de Segunda Liquidación Activa)
```json
{
  "type": "https://avicontrol.edu.co/errors/liquidacion-duplicada",
  "title": "Conflicto de Liquidación",
  "status": 409,
  "detail": "El lote 98765432-e89b-12d3-a456-426614174000 ya cuenta con una liquidación activa (ID: 1045).",
  "instance": "/api/v1/liquidaciones",
  "timestamp": "2026-10-02T16:00:00Z"
}
```

---

### B. Endpoint: `PUT /api/v1/liquidaciones/{id}/anulacion` (Anular Liquidación)

#### Request Body
```json
{
  "motivo": "Precio por kilogramo mal ingresado ($450 COP en vez de $4500 COP). Se requiere reliquidación."
}
```

#### Response Body (`200 OK`)
```json
{
  "idAnulacion": 201,
  "idLiquidacion": 1045,
  "idLote": "98765432-e89b-12d3-a456-426614174000",
  "motivo": "Precio por kilogramo mal ingresado ($450 COP en vez de $4500 COP). Se requiere reliquidación.",
  "fechaHoraAnulacion": "2026-10-02T16:15:00Z",
  "usuarioResponsable": "financiero@avicontrol.edu.co",
  "estadoLiquidacion": "ANULADA"
}
```

#### Response Error (`400 Bad Request` - Motivo Corto)
```json
{
  "type": "https://avicontrol.edu.co/errors/validacion-incorrecta",
  "title": "Error de Validación",
  "status": 400,
  "detail": "El motivo de anulación debe tener al menos 10 caracteres.",
  "instance": "/api/v1/liquidaciones/1045/anulacion",
  "timestamp": "2026-10-02T16:16:00Z"
}
```

---

## Phase 1: Foundational – Esquema SQL de Liquidación y Dominio

**Purpose**: Crear tablas Flyway V3 y modelos del núcleo financiero.

- [ ] **T001** Crear migración Flyway `V3__create_liquidacion_tables.sql` (`liquidacion`, `partida_costo_lote`, `snapshot_datos_origen`, `registro_anulacion`).
- [ ] **T002** Crear modelos de dominio `Liquidacion`, `PartidaCostoLote`, `SnapshotDatosOrigen`, `RegistroAnulacion`.
- [ ] **T003** Crear entidades JPA y repositorios Spring Data con la restricción de unicidad para liquidación activa.
- [ ] **T004** Implementar `Rounding.java` en `domain/shared/` con pruebas unitarias de redondeo `HALF_UP`.

---

## Phase 2: CU03 – Generar Liquidación del Lote

**Purpose**: Cálculo inmutable de Venta Bruta, Costos Operativos y Utilidad Neta.

- [ ] **T005** Unit Test `GenerarLiquidacionServiceTest.java`: verificar Caso Dorado (8500 pollos, $4500/kg → VB $107.100.000, Mort 5.56%, CO $85.000.000, UN $22.100.000), siniestro total (100% mortalidad), rechazo de precio ≤ 0, bloqueo por partidas sin precio y prevención de segunda liquidación activa.
- [ ] **T006** Implementar `GenerarLiquidacionService.java` y `GenerarLiquidacionUseCase.java`.
- [ ] **T007** Controller Test e implementación de `POST /api/v1/liquidaciones` en `LiquidacionController.java`.

---

## Phase 3: CU04 – Anular Liquidación

**Purpose**: Anulación auditable con motivo y rehabilitación del lote.

- [ ] **T008** Unit Test `AnularLiquidacionServiceTest.java`: anulación exitosa de liquidación `ACTIVA`, rechazo de motivos < 10 caracteres, rechazo de anulaciones duplicadas y verificación de atrocidad transaccional.
- [ ] **T009** Implementar `AnularLiquidacionService.java` y `AnularLiquidacionUseCase.java`.
- [ ] **T010** Controller Test e implementación de `PUT /api/v1/liquidaciones/{id}/anulacion` en `LiquidacionController.java`.
