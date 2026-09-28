# Implementation Plan: Sincronización de Datos Locales (M1 y M2)

**Date**: 2026-09-28  
**Specs**:
- [m3-cu07-consultar-galpon-lote-m1](../m3-cu07-consultar-galpon-lote-m1/spec.md) – Consultar Galpón y Lote al M1  
- [m3-cu08-consultar-resultado-sacrificio-m2](../m3-cu08-consultar-resultado-sacrificio-m2/spec.md) – Consultar Resultado Final de Sacrificio al M2  
- [m3-cu09-consultar-alimento-requerido-m2](../m3-cu09-consultar-alimento-requerido-m2/spec.md) – Consultar Alimento Requerido al M2  
- [m3-cu10-consultar-consumo-medicamento-m2](../m3-cu10-consultar-consumo-medicamento-m2/spec.md) – Consultar Consumo de Medicamento al M2  

## Summary

Este subsistema se encarga de la **sincronización e ingesta de datos locales** desde Módulo 1 (Galpones, Lotes y Alertas de Vaciado Sanitario) y Módulo 2 (Resultados Finales de Sacrificio, Partidas de Alimento y Consumos de Medicamentos con sus tramos por recepción). Mantiene la copia local aislada en la base de datos de Módulo 3 de forma incremental e idempotente, registra la bitácora de sincronización (`registro_sincronizacion`) y emite el aviso de utilización de resultados de sacrificio (`aviso_utilizacion_resultado`) de forma asíncrona reintentable.

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, `@Scheduled`, Kafka / RestClient, Jakarta Validation, JUnit 5, MockRestServiceServer, Testcontainers
- **Storage**: PostgreSQL (producción) / H2 en memoria (dev/test), gestionado mediante Flyway (`V1__create_sync_tables.sql` y `V2__create_partidas_tables.sql`)
- **Testing**: JUnit 5, Mockito, Spring Boot Test, MockRestServiceServer (verificación de endpoints M1/M2)
- **Target Platform**: JVM 21 / Docker Container
- **Project Type**: Backend service (capa de sincronización e integración)
- **Performance Goals**: Sincronización periódica eficiente (< 3s por ciclo), reintentos con backoff ante fallos de conexión
- **Constraints**: Operación autónoma en M3 sobre copia local previa si M1 o M2 están indisponibles; actualización idempotente sin duplicar registros ni alterar datos de liquidaciones generadas.
- **Scale/Scope**: Base de datos local para decenas de galpones y lotes por granja.

## Project Structure

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   ├── model/                          # Entidades de dominio (Galpon, Lote, AlertaVaciadoSanitario, etc.)
│   └── port/out/                       # Puertos de salida (Modulo1Port, Modulo2SacrificioPort, AvisoUtilizacionPort, etc.)
├── application/
│   └── service/sync/                   # Servicios de sincronización (SyncModulo1Service, SyncSacrificioService, etc.)
└── infrastructure/
    ├── adapter/
    │   ├── client/                     # Adaptadores REST / Kafka hacia M1 y M2
    │   └── persistence/                # Entidades JPA, Flyway V1 y V2, Repositorios Spring Data
    └── config/                         # SyncSchedulerConfig (@Scheduled)
```

## Phase 1: Foundational – Esquema SQL e Infraestructura de Sincronización

**Purpose**: Crear tablas de copia local y repositorios base.

- [ ] **T001** Crear migración Flyway `V1__create_sync_tables.sql` (`registro_sincronizacion`, `galpon`, `lote`, `alerta_vaciado_sanitario`, `resultado_final_sacrificio`, `aviso_utilizacion_resultado`).
- [ ] **T002** Crear migración Flyway `V2__create_partidas_tables.sql` (`partida_alimento_lote`, `consumo_medicamento_lote`, `tramo_recepcion_consumo`).
- [ ] **T003** Crear entidades JPA y repositorios Spring Data para las tablas de copia local y bitácora.
- [ ] **T004** Crear modelos de dominio y mappers Entity ↔ Domain.

---

## Phase 2: CU07 – Sincronización M1 (Galpones, Lotes y Alertas)

**Purpose**: Sincronización periódica idempotente desde Módulo 1.

- [ ] **T005** Crear puerto `Modulo1Port` y su adaptador `Modulo1RestAdapter`.
- [ ] **T006** Unit Test `SyncModulo1ServiceTest.java`: casos de sync exitosa, actualización incremental, fallo M1 con/sin copia previa y datos inválidos.
- [ ] **T007** Crear `SyncModulo1Service.java`: consulta M1, persiste copia local y registra bitácora en `registro_sincronizacion`.

---

## Phase 3: CU08 – Sincronización M2 Sacrificio y Aviso de Utilización

**Purpose**: Ingesta de resultados finales de sacrificio y emisión de aviso saliente.

- [ ] **T008** Crear puertos `Modulo2SacrificioPort` y `AvisoUtilizacionPort` con sus adaptadores REST/Kafka.
- [ ] **T009** Unit Test `SyncSacrificioServiceTest.java`: verificación de cantidad > 0, peso > 0, actualización pre-utilización y rechazo de duplicados.
- [ ] **T010** Crear `SyncSacrificioService.java` y `AvisoUtilizacionService.java`: recepción de sacrificios y despacho asíncrono reintentable del aviso de utilización.

---

## Phase 4: CU09 y CU10 – Sincronización M2 (Alimento y Medicamentos)

**Purpose**: Ingesta de partidas de alimento y consumos de medicina por tramos.

- [ ] **T011** Crear puertos `Modulo2AlimentoPort` y `Modulo2MedicamentoPort` con sus adaptadores.
- [ ] **T012** Unit Test `SyncAlimentoServiceTest.java` y `SyncMedicamentoServiceTest.java`: verificación de marcas de valorización, preservación de tramos de recepción y no promediación prematura de precios.
- [ ] **T013** Implementar `SyncAlimentoService.java` y `SyncMedicamentoService.java`.
- [ ] **T014** Configurar `@Scheduled` en `SyncSchedulerConfig.java` para disparar periódicamente las 4 tareas de sincronización.
