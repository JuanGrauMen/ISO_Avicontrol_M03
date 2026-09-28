# Implementation Plan: AVICONTROL Módulo 3 – Liquidación de Lote y Análisis de Rentabilidad

**Date**: 28/09/2026
**Versión**: 2.0
**Arquitectura y tecnologías**: [General.md](#technical-context)
**Specs**:

- [spec.md](../spec.md) (índice normativo)
- [CU01](../m3-cu01-lista-galpones/spec.md) – Consultar Lista de Galpones
- [CU03](../m3-cu03-generar-liquidacion/spec.md) – Generar Liquidación del Lote
- [CU04](../m3-cu04-anular-liquidacion/spec.md) – Anular Liquidación
- [CU05](../m3-cu05-desglose-ventas-gastos/spec.md) – Consultar Desglose de Ventas y Gastos
- [CU06](../m3-cu06-historial-liquidaciones/spec.md) – Consultar Historial de Liquidaciones
- [CU07](../m3-cu07-consultar-galpon-lote-m1/spec.md) – Consultar Galpón y Lote al M1
- [CU08](../m3-cu08-consultar-resultado-sacrificio-m2/spec.md) – Consultar Resultado Final de Sacrificio al M2
- [CU09](../m3-cu09-consultar-alimento-requerido-m2/spec.md) – Consultar Alimento Requerido al M2
- [CU10](../m3-cu10-consultar-consumo-medicamento-m2/spec.md) – Consultar Consumo de Medicamento al M2

## Summary

El Módulo 3 genera la **Liquidación** de un lote avícola: toma el resultado final de sacrificio (Módulo 2), el precio por kilogramo que ingresa el Administrador Financiero y los costos del ciclo (Módulo 1 y Módulo 2), y calcula Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta. La Liquidación es inmutable, única `ACTIVA` por lote y solo se corrige por anulación. Cuatro casos de uso de sincronización (CU07–CU10) mantienen una copia local de M1 y M2; cinco de consulta y operación (CU01, CU03, CU04, CU05, CU06) trabajan sobre la Liquidación y esa copia.

Se construye con **Java 21, Spring Boot 3.3.x, PostgreSQL (producción) / H2 (dev/test), Flyway, Apache POI** y una **arquitectura limpia por capas** (domain → application → infrastructure).

## Technical Context

**Language/Version**: Java 21 (LTS)
**Framework**: Spring Boot 3.3.x (Web, Data JPA, Validation)
**Primary Dependencies**:
- Spring Boot Starter Web (REST API)
- Spring Boot Starter Data JPA (persistencia)
- Spring Boot Starter Validation (Bean Validation)
- Apache POI `poi-ooxml:5.2.5` (exportación Excel M3-CU05)
- Flyway (migraciones de esquema)
- Jackson (serialización JSON, incluido en Spring Boot)

**Storage**: PostgreSQL (producción) + H2 en memoria (dev/test), gestionado con Spring Data JPA
**Testing**: JUnit 5, AssertJ, Mockito, MockRestServiceServer (Spring Boot Starter Test)
**Target Platform**: JVM 21, aplicación Spring Boot (JAR ejecutable con REST API)
**Project Type**: single (backend REST API)
**Performance Goals**: SC-002 lista < 3s, SC-002 liquidación < 5s, SC-002 Excel < 10s
**Constraints**:
- Todo valor monetario es `BigDecimal`; totales COP con `setScale(0, HALF_UP)` y porcentajes con `setScale(2, HALF_UP)` (RT-01, RT-02)
- Ninguna operación funcional consulta en vivo a M1/M2 (solo `@Scheduled` sync conoce los gateways)
- M3 no escribe en M1/M2 salvo el aviso de utilización (CU08.FR-006)
- `java.time.Clock` inyectado para "ahora"
- Código en inglés; entidades del glosario y atributos con nombre del spec

**Scale/Scope**: una granja, decenas de galpones, un usuario fijo, 9 casos de uso

## Esquema de Base de Datos (Flyway Migrations)

Las migraciones se ubican en `src/main/resources/db/migration/` y siguen la convención Flyway `V{N}__{descripcion}.sql`.

### V1__create_sync_tables.sql

```sql
-- Bitácora de sincronización (compartida CU07-CU10)
CREATE TABLE registro_sincronizacion (
    id                BIGSERIAL PRIMARY KEY,
    fuente            VARCHAR(20) NOT NULL,  -- MODULO_1 | MODULO_2
    fecha_hora_inicio TIMESTAMP NOT NULL,
    fecha_hora_fin    TIMESTAMP,
    resultado         VARCHAR(20),           -- EXITOSA | FALLIDA
    descripcion_error TEXT,
    registros_actualizados INT DEFAULT 0
);

-- Copia local de galpones (CU07)
CREATE TABLE galpon (
    id_galpon     UUID PRIMARY KEY,
    nombre        VARCHAR(100) NOT NULL,
    aforo_maximo  INT NOT NULL,
    estado        VARCHAR(30) NOT NULL,
    fecha_hora_sync TIMESTAMP NOT NULL
);

-- Copia local de lotes (CU07)
CREATE TABLE lote (
    id_lote            UUID PRIMARY KEY,
    id_galpon          UUID NOT NULL REFERENCES galpon(id_galpon),
    nombre             VARCHAR(100) NOT NULL,
    fecha_ingreso      DATE NOT NULL,
    poblacion_inicial  INT NOT NULL,
    poblacion_actual   INT NOT NULL,
    costo_total_cop    BIGINT NOT NULL,
    fecha_hora_sync    TIMESTAMP NOT NULL
);

-- Alertas de vaciado sanitario (CU07)
CREATE TABLE alerta_vaciado_sanitario (
    id_alerta        UUID PRIMARY KEY,
    id_galpon        UUID NOT NULL,
    id_lote          UUID NOT NULL,
    fecha_hora_evento TIMESTAMP NOT NULL,
    fecha_hora_sync   TIMESTAMP NOT NULL
);

-- Resultado final de sacrificio (CU08)
CREATE TABLE resultado_final_sacrificio (
    id_resultado       UUID PRIMARY KEY,
    id_lote            UUID NOT NULL,
    id_galpon          UUID NOT NULL,
    cantidad_final_pollos INT NOT NULL,
    peso_total_kg      NUMERIC(12,2) NOT NULL,
    fecha_registro     TIMESTAMP NOT NULL,
    fecha_hora_sync    TIMESTAMP NOT NULL,
    estado_utilizacion VARCHAR(20) NOT NULL DEFAULT 'NO_UTILIZADO'
);

-- Aviso de utilización (CU08)
CREATE TABLE aviso_utilizacion_resultado (
    id_aviso          BIGSERIAL PRIMARY KEY,
    id_resultado      UUID NOT NULL REFERENCES resultado_final_sacrificio(id_resultado),
    id_liquidacion    BIGINT,
    fecha_hora_emision TIMESTAMP NOT NULL,
    estado_entrega    VARCHAR(20) NOT NULL DEFAULT 'PENDIENTE',
    intentos          INT NOT NULL DEFAULT 0
);
```

### V2__create_partidas_tables.sql

```sql
-- Partidas de alimento (CU09)
CREATE TABLE partida_alimento_lote (
    id_partida_origen    UUID PRIMARY KEY,
    id_lote              UUID NOT NULL,
    id_galpon            UUID NOT NULL,
    tipo_alimento        VARCHAR(50) NOT NULL,
    cantidad_kg          NUMERIC(12,3) NOT NULL,
    precio_unitario_kg_cop NUMERIC(14,2),
    valor_impuesto_cop   NUMERIC(14,2),
    estado_valorizacion  VARCHAR(20) NOT NULL DEFAULT 'VALORIZADA',
    referencia_origen    VARCHAR(200),
    fecha_hora_sync      TIMESTAMP NOT NULL
);

-- Consumos de medicamento (CU10)
CREATE TABLE consumo_medicamento_lote (
    id_consumo_origen    UUID PRIMARY KEY,
    id_lote              UUID NOT NULL,
    id_galpon            UUID NOT NULL,
    medicamento          VARCHAR(150) NOT NULL,
    fecha_consumo        DATE NOT NULL,
    cantidad_unidad_base NUMERIC(12,3) NOT NULL,
    unidad_base          VARCHAR(30) NOT NULL,
    fecha_hora_sync      TIMESTAMP NOT NULL
);

-- Tramos por recepción (CU10)
CREATE TABLE tramo_recepcion_consumo (
    id_tramo             BIGSERIAL PRIMARY KEY,
    id_consumo_origen    UUID NOT NULL REFERENCES consumo_medicamento_lote(id_consumo_origen),
    referencia_recepcion VARCHAR(200),
    cantidad_unidad_base NUMERIC(12,3) NOT NULL,
    precio_unitario_base_cop NUMERIC(14,2),
    valor_impuesto_cop   NUMERIC(14,2),
    estado_valorizacion  VARCHAR(20) NOT NULL DEFAULT 'VALORIZADO'
);
```

### V3__create_liquidacion_tables.sql

```sql
-- Liquidación (CU03)
CREATE TABLE liquidacion (
    id_liquidacion      BIGSERIAL PRIMARY KEY,
    id_lote             UUID NOT NULL,
    id_galpon           UUID NOT NULL,
    id_resultado_sacrificio UUID,
    pollos_vendidos     INT,
    peso_total_kg       NUMERIC(12,2),
    peso_promedio_kg    NUMERIC(8,4),
    precio_kg_cop       NUMERIC(14,2),
    venta_bruta_cop     BIGINT NOT NULL,
    mortalidad_aves     INT NOT NULL,
    porcentaje_mortalidad NUMERIC(5,2) NOT NULL,
    costos_operativos_cop BIGINT NOT NULL,
    utilidad_neta_cop   BIGINT NOT NULL,
    estado              VARCHAR(10) NOT NULL DEFAULT 'ACTIVA',
    fecha_hora_generacion TIMESTAMP NOT NULL,
    usuario_responsable VARCHAR(100) NOT NULL,
    CONSTRAINT uq_lote_activa UNIQUE (id_lote) -- parcial: solo ACTIVA (se maneja a nivel aplicación)
);

-- Partidas de costo congeladas en la liquidación (CU03/CU05)
CREATE TABLE partida_costo_lote (
    id_partida          BIGSERIAL PRIMARY KEY,
    id_liquidacion      BIGINT NOT NULL REFERENCES liquidacion(id_liquidacion),
    id_lote             UUID NOT NULL,
    categoria           VARCHAR(20) NOT NULL, -- ALIMENTO | MEDICINA | POBLACION
    concepto            VARCHAR(200) NOT NULL,
    cantidad            NUMERIC(12,3) NOT NULL,
    unidad_medida       VARCHAR(30) NOT NULL,
    precio_unitario_cop NUMERIC(14,2) NOT NULL,
    subtotal_cop        BIGINT NOT NULL,
    fuente_origen       VARCHAR(20) NOT NULL,
    referencia_origen   VARCHAR(200)
);

-- Snapshot de datos de origen (CU03)
CREATE TABLE snapshot_datos_origen (
    id                    BIGSERIAL PRIMARY KEY,
    id_liquidacion        BIGINT NOT NULL REFERENCES liquidacion(id_liquidacion),
    id_lote               UUID NOT NULL,
    fuente                VARCHAR(20) NOT NULL, -- MODULO_1 | MODULO_2
    fecha_hora_sync       TIMESTAMP NOT NULL,
    poblacion_inicial     INT,
    poblacion_actual      INT,
    costo_total_lote_cop  BIGINT
);

-- Registro de anulación (CU04)
CREATE TABLE registro_anulacion (
    id_anulacion          BIGSERIAL PRIMARY KEY,
    id_liquidacion        BIGINT NOT NULL UNIQUE REFERENCES liquidacion(id_liquidacion),
    motivo                VARCHAR(500) NOT NULL,
    fecha_hora_anulacion  TIMESTAMP NOT NULL,
    usuario_responsable   VARCHAR(100) NOT NULL
);
```

## Contratos de Integración con Módulos Externos

```
┌─────────────┐         REST GET          ┌─────────────┐
│  Módulo 1   │ ◄──────────────────────── │             │
│  (Galpones) │   /api/v1/galpones        │             │
│             │   /api/v1/alertas-vaciado  │             │
└─────────────┘                           │  Módulo 3   │
                                          │  (Liquidac.)│
┌─────────────┐         REST GET          │             │
│  Módulo 2   │ ◄──────────────────────── │             │
│  (Operac.)  │   /api/v1/sacrificios     │             │
│             │   /api/v1/alimento        │             │
│             │   /api/v1/medicamentos    │             │
│             │ ────────────────────────► │             │
│             │   POST aviso utilización  │             │
└─────────────┘                           └─────────────┘
```

### Puertos (Interfaces en dominio/application)

| Puerto               | Tipo    | CU      | Descripción                                            |
|:----------------------|:--------|:--------|:-------------------------------------------------------|
| `Modulo1Port`         | Lectura | CU07    | Obtener galpones, lotes y alertas de vaciado sanitario |
| `Modulo2SacrificioPort` | Lectura | CU08 | Obtener resultados finales de sacrificio               |
| `Modulo2AlimentoPort` | Lectura | CU09    | Obtener partidas de alimento del ciclo                 |
| `Modulo2MedicamentoPort` | Lectura | CU10 | Obtener consumos de medicamento del ciclo              |
| `AvisoUtilizacionPort` | Escritura | CU08 | Emitir aviso de utilización del resultado de sacrificio |

### Adaptadores (Implementaciones en infrastructure)

Cada puerto se implementa con un adaptador REST (`RestClient` de Spring Boot) que invoca el endpoint del módulo correspondiente. La URL base es configurable vía `application.yml`.

> **Decisión pendiente (D-02, D-03)**: los endpoints exactos de M1 y M2 deben acordarse con los equipos de cada módulo. Los adaptadores se diseñan para desacoplarse del transporte: si se acuerda mensajería (RabbitMQ/Kafka), se reemplaza el adaptador sin tocar el puerto.

## Project Structure

### Documentation (this feature)

```text
docs/specs/features/modulo3/
├── spec.md                      # índice normativo
├── plan/
│   └── plan-v2.md               # este archivo
├── CAMBIOS.md
└── m3-cuNN-*/spec.md            # un spec por caso de uso
```

### Source Code (repository root)

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   ├── model/
│   │   ├── EstadoGalpon.java                  # enum: 6 estados
│   │   ├── Galpon.java                        # entidad de dominio
│   │   ├── Lote.java                          # entidad de dominio
│   │   ├── AlertaVaciadoSanitario.java        # entidad de dominio
│   │   ├── ResultadoFinalSacrificio.java      # entidad de dominio
│   │   ├── EstadoUtilizacion.java             # enum: NO_UTILIZADO | UTILIZADO
│   │   ├── PartidaAlimentoLote.java           # entidad de dominio
│   │   ├── ConsumoMedicamentoLote.java        # entidad de dominio
│   │   ├── TramoRecepcionConsumo.java         # entidad de dominio
│   │   ├── Liquidacion.java                   # entidad de dominio (core)
│   │   ├── EstadoLiquidacion.java             # enum: ACTIVA | ANULADA
│   │   ├── PartidaCostoLote.java              # entidad de dominio
│   │   ├── CategoriaCosto.java                # enum: ALIMENTO | MEDICINA | POBLACION
│   │   ├── SnapshotDatosOrigen.java           # entidad de dominio
│   │   ├── RegistroAnulacion.java             # entidad de dominio
│   │   ├── RegistroSincronizacion.java        # entidad de dominio
│   │   ├── FuenteSincronizacion.java          # enum: MODULO_1 | MODULO_2
│   │   ├── ResultadoSincronizacion.java       # enum: EXITOSA | FALLIDA
│   │   └── AvisoUtilizacionResultado.java     # entidad de dominio
│   ├── port/
│   │   ├── in/                                # puertos de entrada (use cases)
│   │   │   ├── ListarGalponesUseCase.java
│   │   │   ├── GenerarLiquidacionUseCase.java
│   │   │   ├── AnularLiquidacionUseCase.java
│   │   │   ├── ConsultarDesgloseUseCase.java
│   │   │   ├── ConsultarHistorialUseCase.java
│   │   │   └── ExportarExcelUseCase.java
│   │   └── out/                               # puertos de salida (dependencias externas)
│   │       ├── GalponRepository.java
│   │       ├── LoteRepository.java
│   │       ├── AlertaVaciadoRepository.java
│   │       ├── ResultadoSacrificioRepository.java
│   │       ├── PartidaAlimentoRepository.java
│   │       ├── ConsumoMedicamentoRepository.java
│   │       ├── LiquidacionRepository.java
│   │       ├── RegistroAnulacionRepository.java
│   │       ├── RegistroSincronizacionRepository.java
│   │       ├── Modulo1Port.java
│   │       ├── Modulo2SacrificioPort.java
│   │       ├── Modulo2AlimentoPort.java
│   │       ├── Modulo2MedicamentoPort.java
│   │       └── AvisoUtilizacionPort.java
│   └── shared/
│       └── Rounding.java                      # copTotal(BigDecimal), percentage(BigDecimal)
│
├── application/
│   ├── service/
│   │   ├── ListarGalponesService.java         # CU01
│   │   ├── GenerarLiquidacionService.java     # CU03
│   │   ├── AnularLiquidacionService.java      # CU04
│   │   ├── ConsultarDesgloseService.java      # CU05
│   │   ├── ConsultarHistorialService.java     # CU06
│   │   ├── ExportarExcelService.java          # CU05 (Excel)
│   │   └── sync/
│   │       ├── SyncModulo1Service.java        # CU07
│   │       ├── SyncSacrificioService.java     # CU08
│   │       ├── SyncAlimentoService.java       # CU09
│   │       ├── SyncMedicamentoService.java    # CU10
│   │       └── AvisoUtilizacionService.java   # CU08 (aviso saliente)
│   └── dto/
│       ├── GalponResumenDto.java
│       ├── LiquidacionDto.java
│       ├── DesgloseDto.java
│       ├── HistorialFiltroDto.java
│       └── AnulacionDto.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── rest/                              # adaptadores REST de entrada
│   │   │   ├── GalponController.java          # CU01
│   │   │   ├── LiquidacionController.java     # CU03, CU04
│   │   │   ├── DesgloseController.java        # CU05
│   │   │   └── HistorialController.java       # CU06
│   │   ├── client/                            # adaptadores REST de salida (M1/M2)
│   │   │   ├── Modulo1RestAdapter.java
│   │   │   ├── Modulo2SacrificioRestAdapter.java
│   │   │   ├── Modulo2AlimentoRestAdapter.java
│   │   │   ├── Modulo2MedicamentoRestAdapter.java
│   │   │   └── AvisoUtilizacionRestAdapter.java
│   │   └── persistence/                       # adaptadores JPA de salida
│   │       ├── entity/                        # entidades JPA (@Entity)
│   │       │   ├── GalponEntity.java
│   │       │   ├── LoteEntity.java
│   │       │   ├── AlertaVaciadoEntity.java
│   │       │   ├── ResultadoSacrificioEntity.java
│   │       │   ├── PartidaAlimentoEntity.java
│   │       │   ├── ConsumoMedicamentoEntity.java
│   │       │   ├── TramoRecepcionEntity.java
│   │       │   ├── LiquidacionEntity.java
│   │       │   ├── PartidaCostoEntity.java
│   │       │   ├── SnapshotOrigenEntity.java
│   │       │   ├── RegistroAnulacionEntity.java
│   │       │   ├── RegistroSincronizacionEntity.java
│   │       │   └── AvisoUtilizacionEntity.java
│   │       ├── jpa/                           # Spring Data JPA repositories
│   │       │   ├── JpaGalponRepository.java
│   │       │   ├── JpaLoteRepository.java
│   │       │   ├── JpaAlertaVaciadoRepository.java
│   │       │   ├── JpaResultadoSacrificioRepository.java
│   │       │   ├── JpaPartidaAlimentoRepository.java
│   │       │   ├── JpaConsumoMedicamentoRepository.java
│   │       │   ├── JpaLiquidacionRepository.java
│   │       │   ├── JpaRegistroAnulacionRepository.java
│   │       │   └── JpaRegistroSincronizacionRepository.java
│   │       └── mapper/                        # Entity ↔ Domain mappers
│   │           ├── GalponMapper.java
│   │           ├── LoteMapper.java
│   │           ├── LiquidacionMapper.java
│   │           └── ...
│   ├── config/
│   │   ├── SyncSchedulerConfig.java           # @Scheduled para CU07-CU10
│   │   └── ClockConfig.java                   # @Bean Clock
│   └── exception/
│       └── GlobalExceptionHandler.java        # @ControllerAdvice
│
└── AvicontrolModulo3Application.java          # @SpringBootApplication

src/main/resources/
├── application.yml                            # datasource, sync intervals, M1/M2 base URLs
├── application-dev.yml                        # H2 in-memory
├── application-prod.yml                       # PostgreSQL
└── db/migration/
    ├── V1__create_sync_tables.sql
    ├── V2__create_partidas_tables.sql
    └── V3__create_liquidacion_tables.sql

src/test/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   └── model/
│       ├── LiquidacionTest.java
│       └── RoundingTest.java
├── application/
│   └── service/
│       ├── GenerarLiquidacionServiceTest.java
│       ├── AnularLiquidacionServiceTest.java
│       └── ...
└── infrastructure/
    ├── adapter/rest/
    │   ├── GalponControllerTest.java
    │   ├── LiquidacionControllerTest.java
    │   └── ...
    └── adapter/client/
        ├── Modulo1RestAdapterTest.java
        └── ...
```

**Structure Decision**: Arquitectura limpia por capas con paquetes `domain` → `application` → `infrastructure`. Las dependencias siempre apuntan hacia adentro (infrastructure depende de application, application depende de domain). Los puertos de entrada definen los use cases; los puertos de salida definen las interfaces de repositorio y clientes externos.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Inicialización del proyecto con Spring Boot, configuración base y estructura de paquetes.

- [ ] T001 Actualizar `pom.xml`: Java 21, Spring Boot 3.3.x (parent), agregar dependencias: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-test`, `h2` (test/runtime), `postgresql` (runtime), `flyway-core`, `poi-ooxml:5.2.5`
- [ ] T002 Crear `AvicontrolModulo3Application.java` con `@SpringBootApplication`
- [ ] T003 Crear `application.yml`, `application-dev.yml` (H2), `application-prod.yml` (PostgreSQL) con datasource, sync intervals y URLs base de M1/M2
- [ ] T004 Crear `ClockConfig.java` con `@Bean Clock`
- [ ] T005 Crear `GlobalExceptionHandler.java` con `@ControllerAdvice`
- [ ] T006 Crear `Rounding.java` en `domain/shared/` (RT-01, RT-02)
- [ ] T007 Crear todos los enums: `EstadoGalpon`, `EstadoLiquidacion`, `EstadoUtilizacion`, `CategoriaCosto`, `FuenteSincronizacion`, `ResultadoSincronizacion`
- [ ] T008 Test: `RoundingTest.java` — verificar `copTotal()` con HALF_UP a enteros y `percentage()` con 2 decimales

**Checkpoint**: Proyecto arranca con `mvn spring-boot:run`, H2 accesible, tests de Rounding pasan.

---

## Phase 2: Foundational – Migraciones y Entidades Base (Blocking Prerequisites)

**Purpose**: Esquema de base de datos y entidades JPA que TODAS las historias necesitan.

**⚠️ CRITICAL**: No se puede implementar ningún CU sin que las migraciones Flyway y entidades JPA estén listas.

- [ ] T009 Crear `V1__create_sync_tables.sql` (tablas: `registro_sincronizacion`, `galpon`, `lote`, `alerta_vaciado_sanitario`, `resultado_final_sacrificio`, `aviso_utilizacion_resultado`)
- [ ] T010 Crear `V2__create_partidas_tables.sql` (tablas: `partida_alimento_lote`, `consumo_medicamento_lote`, `tramo_recepcion_consumo`)
- [ ] T011 Crear `V3__create_liquidacion_tables.sql` (tablas: `liquidacion`, `partida_costo_lote`, `snapshot_datos_origen`, `registro_anulacion`)
- [ ] T012 Crear entidades JPA en `infrastructure/adapter/persistence/entity/`: `GalponEntity`, `LoteEntity`, `AlertaVaciadoEntity`, `ResultadoSacrificioEntity`, `PartidaAlimentoEntity`, `ConsumoMedicamentoEntity`, `TramoRecepcionEntity`, `LiquidacionEntity`, `PartidaCostoEntity`, `SnapshotOrigenEntity`, `RegistroAnulacionEntity`, `RegistroSincronizacionEntity`, `AvisoUtilizacionEntity`
- [ ] T013 Crear modelos de dominio en `domain/model/`: `Galpon`, `Lote`, `AlertaVaciadoSanitario`, `ResultadoFinalSacrificio`, `PartidaAlimentoLote`, `ConsumoMedicamentoLote`, `TramoRecepcionConsumo`, `Liquidacion`, `PartidaCostoLote`, `SnapshotDatosOrigen`, `RegistroAnulacion`, `RegistroSincronizacion`, `AvisoUtilizacionResultado`
- [ ] T014 Crear interfaces JPA en `infrastructure/adapter/persistence/jpa/`: todas las interfaces `extends JpaRepository`
- [ ] T015 Crear interfaces de repositorio (puertos de salida) en `domain/port/out/`
- [ ] T016 Crear mappers Entity ↔ Domain en `infrastructure/adapter/persistence/mapper/`
- [ ] T017 Crear implementaciones de repositorio que adaptan JPA a puertos de dominio
- [ ] T018 Test de integración: verificar que Flyway aplica las 3 migraciones y el contexto Spring levanta con H2

**Checkpoint**: `mvn test` pasa. Flyway crea tablas. Contexto Spring arranca. Repositorios inyectables.

---

## Phase 3: CU07 – Sincronización Galpones/Lotes M1 (Priority: P1)

**Goal**: M3 consulta periódicamente a M1 y mantiene copia local de galpones, lotes y alertas de vaciado sanitario.

**Independent Test**: Sincronizar contra un mock de M1, verificar copia local poblada e idempotencia.

### Tests para CU07

- [ ] T019 [P] [CU07] Unit test `SyncModulo1ServiceTest.java`: sincronización exitosa, incremental, idempotente, fallo de M1 con copia previa, sin copia previa, datos inválidos, estado desconocido
- [ ] T020 [P] [CU07] Integration test con MockRestServiceServer: verificar polling → copia local → bitácora

### Implementation para CU07

- [ ] T021 [CU07] Crear `Modulo1Port.java` (interfaz en `domain/port/out/`)
- [ ] T022 [CU07] Crear `Modulo1RestAdapter.java` en `infrastructure/adapter/client/` (implementa `Modulo1Port` con `RestClient`)
- [ ] T023 [CU07] Crear `SyncModulo1Service.java` en `application/service/sync/`: lógica de sincronización idempotente (FR-001 a FR-011)
- [ ] T024 [CU07] Crear `SyncSchedulerConfig.java` con `@Scheduled` configurable para CU07

**Checkpoint**: Sincronización M1 funcional. Copia local poblada. Reintentos ante fallo.

---

## Phase 4: CU08, CU09, CU10 – Sincronización M2 (Priority: P1)

**Goal**: M3 consulta periódicamente a M2: resultado de sacrificio, partidas de alimento y consumos de medicamento.

**Independent Test**: Sincronizar cada tipo contra mock de M2, verificar copia local, validaciones e idempotencia.

### Tests para CU08-CU10

- [ ] T025 [P] [CU08] Unit test `SyncSacrificioServiceTest.java`: exitosa, resultado inválido (cero/negativo), duplicado, corrección pre-utilización, fallo M2
- [ ] T026 [P] [CU09] Unit test `SyncAlimentoServiceTest.java`: exitosa, partida sin precio, ausencia total, idempotente
- [ ] T027 [P] [CU10] Unit test `SyncMedicamentoServiceTest.java`: exitosa, tramos con precios distintos, tramo sin precio, ausencia total (no bloquea), idempotente

### Implementation para CU08-CU10

- [ ] T028 [CU08] Crear `Modulo2SacrificioPort.java`, `Modulo2SacrificioRestAdapter.java`
- [ ] T029 [CU08] Crear `SyncSacrificioService.java`: sync idempotente, validación (cantidad > 0, peso > 0), rechazo duplicados, actualización pre-utilización
- [ ] T030 [CU09] Crear `Modulo2AlimentoPort.java`, `Modulo2AlimentoRestAdapter.java`
- [ ] T031 [CU09] Crear `SyncAlimentoService.java`: sync idempotente, marcar no valorizadas, no calcular subtotales, no promediar precios
- [ ] T032 [CU10] Crear `Modulo2MedicamentoPort.java`, `Modulo2MedicamentoRestAdapter.java`
- [ ] T033 [CU10] Crear `SyncMedicamentoService.java`: sync idempotente con tramos por recepción, no promediar, marcar no valorizados
- [ ] T034 [CU08] Crear `AvisoUtilizacionPort.java`, `AvisoUtilizacionRestAdapter.java`, `AvisoUtilizacionService.java`: aviso asíncrono, idempotente, reintentable (FR-006, FR-007)
- [ ] T035 Agregar `@Scheduled` en `SyncSchedulerConfig.java` para CU08, CU09, CU10

**Checkpoint**: Las 4 sincronizaciones funcionan. Copia local completa de M1 y M2. Aviso de utilización configurable.

---

## Phase 5: CU01 – Lista de Galpones (Priority: P1)

**Goal**: El administrador financiero consulta la lista de galpones con mortalidad % y filtra por estado. Acciones habilitadas/deshabilitadas según estado.

**Independent Test**: GET lista → verificar campos, mortalidad %, filtro por estado, acciones según estado.

### Tests para CU01

- [ ] T036 [P] [CU01] Unit test `ListarGalponesServiceTest.java`: datos completos con mortalidad %, filtro por estado, estado vacío, acciones habilitadas/deshabilitadas por estado
- [ ] T037 [P] [CU01] Integration test `GalponControllerTest.java`: GET `/api/v1/galpones`, filtro `?estado=VACIADO_SANITARIO`, respuesta JSON con campos esperados

### Implementation para CU01

- [ ] T038 [CU01] Crear `ListarGalponesUseCase.java` (interfaz en `domain/port/in/`)
- [ ] T039 [CU01] Crear `ListarGalponesService.java`: consulta copia local, calcula mortalidad %, determina acciones según estado y existencia de resultado/liquidación
- [ ] T040 [CU01] Crear `GalponResumenDto.java` con campos: idGalpon, nombre, estado, idLote, nombreLote, fechaIngreso, porcentajeMortalidad, fechaHoraSync, accionesDisponibles
- [ ] T041 [CU01] Crear `GalponController.java`: GET `/api/v1/galpones?estado={estado}`

**Checkpoint**: Lista de galpones funcional. Mortalidad % visible. Filtro por estado operativo.

---

## Phase 6: CU03 – Generar Liquidación (Priority: P1)

**Goal**: Generar la Liquidación definitiva con Venta Bruta, Mortalidad, Costos Operativos y Utilidad Neta. Incluye siniestro total.

**Independent Test**: POST con precio/kg → verificar los 4 indicadores contra cálculo aritmético manual.

### Tests para CU03

- [ ] T042 [P] [CU03] Unit test `GenerarLiquidacionServiceTest.java`: Caso Dorado (8500 pollos, 23800 kg, $4500/kg → VB $107.100.000, Mort 5.56%, CO $85.000.000, UN $22.100.000), siniestro total, bloqueo sin resultado, rechazo precio ≤ 0, bloqueo partidas sin precio, bloqueo segunda ACTIVA, generación tras anulación
- [ ] T043 [P] [CU03] Integration test `LiquidacionControllerTest.java`: POST `/api/v1/liquidaciones`, validaciones, snapshot financiero persistido

### Implementation para CU03

- [ ] T044 [CU03] Crear `GenerarLiquidacionUseCase.java` (interfaz)
- [ ] T045 [CU03] Crear `GenerarLiquidacionService.java`: validaciones (vaciado sanitario, resultado válido o mortalidad total, precio > 0, partidas valorizadas, no segunda ACTIVA), cálculos (VB, Mortalidad, CO, UN), snapshot, marcado de utilización + aviso M2
- [ ] T046 [CU03] Crear `LiquidacionController.java`: POST `/api/v1/liquidaciones`
- [ ] T047 [CU03] Crear `LiquidacionDto.java` con los 4 indicadores + datos de venta + estado + auditoría

**Checkpoint**: Liquidación funcional end-to-end. Caso Dorado verificado. Siniestro total verificado.

---

## Phase 7: CU04 – Anular Liquidación (Priority: P1)

**Goal**: Anulación auditable de una Liquidación ACTIVA. Conservación íntegra. Rehabilitación del lote.

**Independent Test**: POST anulación → verificar estado ANULADA, registro auditoría, lote rehabilitado.

### Tests para CU04

- [ ] T048 [P] [CU04] Unit test `AnularLiquidacionServiceTest.java`: anulación exitosa, motivo vacío rechazado, motivo < 10 chars rechazado, ya anulada rechazada, atomicidad
- [ ] T049 [P] [CU04] Integration test: POST `/api/v1/liquidaciones/{id}/anulacion`, flujo completo anulación → re-generación

### Implementation para CU04

- [ ] T050 [CU04] Crear `AnularLiquidacionUseCase.java` (interfaz)
- [ ] T051 [CU04] Crear `AnularLiquidacionService.java`: validación estado ACTIVA, motivo 10-500 chars, transacción atómica, registro auditoría
- [ ] T052 [CU04] Agregar endpoint PUT `/api/v1/liquidaciones/{id}/anulacion` en `LiquidacionController.java`

**Checkpoint**: Anulación funcional. Auditoría completa. Corrección end-to-end (anular + re-generar).

---

## Phase 8: CU05 – Desglose de Ventas y Gastos + Excel (Priority: P2)

**Goal**: Consultar desglose pormenorizado y exportar a Excel. Solo lectura para anuladas.

**Independent Test**: GET desglose → verificar que subtotales suman igual a Costos Operativos. Descargar .xlsx.

### Tests para CU05

- [ ] T053 [P] [CU05] Unit test `ConsultarDesgloseServiceTest.java`: desglose completo, suma subtotales = CO, siniestro total, anulada con aviso
- [ ] T054 [P] [CU05] Unit test `ExportarExcelServiceTest.java`: generación .xlsx con Apache POI, formato monetario, fórmulas de suma
- [ ] T055 [P] [CU05] Integration test `DesgloseControllerTest.java`: GET `/api/v1/liquidaciones/{id}/desglose`, GET `/api/v1/liquidaciones/{id}/desglose/excel`

### Implementation para CU05

- [ ] T056 [CU05] Crear `ConsultarDesgloseUseCase.java`, `ExportarExcelUseCase.java` (interfaces)
- [ ] T057 [CU05] Crear `ConsultarDesgloseService.java`: proyección derivada (no almacenada), agrupación por categoría
- [ ] T058 [CU05] Crear `ExportarExcelService.java`: Apache POI, Matriz de Venta Final + desglose, formato COP, fórmulas
- [ ] T059 [CU05] Crear `DesgloseController.java`: GET desglose JSON + GET descarga Excel

**Checkpoint**: Desglose visible. Subtotales cuadran al peso. Excel generado correctamente.

---

## Phase 9: CU06 – Historial de Liquidaciones (Priority: P3)

**Goal**: Historial cronológico filtrable por galpón y rango de fechas. Navegación a desglose.

**Independent Test**: GET historial con filtros → verificar orden descendente, filtrado correcto, estados diferenciados.

### Tests para CU06

- [ ] T060 [P] [CU06] Unit test `ConsultarHistorialServiceTest.java`: orden descendente, filtro galpón + fechas, fecha invertida rechazada, vacío total vs. sin coincidencias
- [ ] T061 [P] [CU06] Integration test `HistorialControllerTest.java`: GET `/api/v1/liquidaciones?galpon={id}&desde={fecha}&hasta={fecha}`

### Implementation para CU06

- [ ] T062 [CU06] Crear `ConsultarHistorialUseCase.java` (interfaz)
- [ ] T063 [CU06] Crear `ConsultarHistorialService.java`: consulta filtrada ordenada, validación de fechas
- [ ] T064 [CU06] Crear `HistorialController.java`: GET `/api/v1/liquidaciones` con query params

**Checkpoint**: Historial funcional. Filtros precisos. Navegación a desglose.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Mejoras transversales que afectan a múltiples CUs.

- [ ] T065 Indicador RT-09: toda respuesta de la API incluye `fechaHoraUltimaSyncM1` y `fechaHoraUltimaSyncM2`
- [ ] T066 Documentar API con OpenAPI/Swagger annotations
- [ ] T067 Tests de integración end-to-end: flujo completo sync → lista → liquidación → anulación → re-liquidación → desglose → historial
- [ ] T068 Configurar logging estructurado (SLF4J + Logback)
- [ ] T069 Revisar manejo de errores y respuestas HTTP (400, 404, 409, 500)
- [ ] T070 Performance: verificar SC-001 (lista < 3s), SC-002 (liquidación < 5s), SC-002 (Excel < 10s)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — se inicia de inmediato
- **Foundational (Phase 2)**: Depende de Phase 1 — BLOQUEA todas las historias
- **CU07 (Phase 3)**: Depende de Phase 2 — copia local M1
- **CU08-CU10 (Phase 4)**: Depende de Phase 2 — puede ejecutarse en paralelo con Phase 3
- **CU01 (Phase 5)**: Depende de Phase 3 (necesita copia local de galpones)
- **CU03 (Phase 6)**: Depende de Phase 3, 4 y 5 (necesita toda la copia local + lista)
- **CU04 (Phase 7)**: Depende de Phase 6 (necesita Liquidación existente)
- **CU05 (Phase 8)**: Depende de Phase 6 (necesita Liquidación con partidas)
- **CU06 (Phase 9)**: Depende de Phase 6 (necesita Liquidaciones generadas)
- **Polish (Phase 10)**: Depende de todas las fases anteriores

### Diagrama de Ejecución

```text
Phase 1 (Setup)
    │
    ▼
Phase 2 (Foundational: Flyway + Entities)
    │
    ├─────────────────┐
    ▼                 ▼
Phase 3 (CU07/M1)  Phase 4 (CU08-10/M2)
    │                 │
    ├─────────────────┘
    ▼
Phase 5 (CU01: Lista)
    │
    ▼
Phase 6 (CU03: Liquidación)
    │
    ├───────────┬───────────┐
    ▼           ▼           ▼
Phase 7     Phase 8     Phase 9
(CU04)      (CU05)      (CU06)
    │           │           │
    └───────────┴───────────┘
              │
              ▼
        Phase 10 (Polish)
```

### Within Each Phase

- Modelos de dominio antes que servicios
- Servicios antes que controladores
- Tests paralelos a la implementación
- Commit después de cada tarea o grupo lógico

## Open Questions (Decisiones Pendientes)

| #    | Decisión                                                                                                      | Afecta        | Contraparte | Supuesto Actual                                            |
|:-----|:--------------------------------------------------------------------------------------------------------------|:--------------|:------------|:-----------------------------------------------------------|
| D-01 | Criterio de costeo del alimento: compras, consumo real o requerimiento proyectado                             | CU03, CU09    | Módulo 2    | Se almacena la "partida" en términos neutros               |
| D-02 | Exposición formal de población inicial, población actual, costo del lote, estado y alerta                     | CU07          | Módulo 1    | REST GET con contrato mínimo asumido                       |
| D-03 | Mecanismo de aviso de utilización del resultado final de sacrificio                                            | CU08          | Módulo 2    | POST REST asíncrono, reintentable                          |
| D-04 | Ruta de estado para liquidar un lote con mortalidad total                                                     | CU03, CU07    | Módulo 1    | Solo se liquida si llega a `Vaciado Sanitario`             |
| D-05 | Tratamiento del impuesto informado por M2: ¿costos sobre valor neto o con impuesto?                           | CU03, CU09, CU10 | Módulo 2 | Se almacenan ambos, no se consolidan hasta definir         |
| D-06 | Registro de mortalidad ocurrida fuera del estado `Productivo`                                                 | CU03, CU07    | Módulo 1    | Se acepta la mortalidad tal como la informa M1             |
| D-07 | Grafía exacta del catálogo de estados compartido                                                              | CU01, CU07    | Módulo 1    | Enum con 6 valores, mapping configurable                   |

## Notes

- `[CU##]` mapea tarea a caso de uso para trazabilidad
- `[P]` indica tarea parallelizable dentro de su fase
- Cada fase se verifica independientemente antes de avanzar
- Verificar tests con `mvn test` tras cada tarea
- Commit después de cada tarea o grupo lógico
- Stop at any checkpoint to validate story independently
