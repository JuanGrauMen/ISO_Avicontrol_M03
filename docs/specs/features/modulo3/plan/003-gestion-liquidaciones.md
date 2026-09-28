# Implementation Plan: Gestión del Ciclo de Vida de Liquidaciones

**Date**: 2026-09-28  
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
- **Scale/Scope**: Una liquidación por lote cerrado por ciclo.

## Project Structure

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   ├── model/                          # Liquidacion, PartidaCostoLote, SnapshotDatosOrigen, RegistroAnulacion
│   ├── port/in/                        # GenerarLiquidacionUseCase.java, AnularLiquidacionUseCase.java
│   └── shared/                         # Rounding.java (copTotal, percentage)
├── application/
│   ├── service/                        # GenerarLiquidacionService.java, AnularLiquidacionService.java
│   └── dto/                            # LiquidacionDto.java, AnulacionDto.java
└── infrastructure/
    └── adapter/rest/                   # LiquidacionController.java (POST y PUT /api/v1/liquidaciones)
```

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
