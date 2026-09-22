# Implementation Plan: AVICONTROL Módulo 3 – Liquidación de Lote y Análisis de Rentabilidad

**Date**: 2026-09-21
**Versión**: 2.0 — ceñida a las secciones de `plan-template.md`; reemplaza a `plan-v1.md`
**Spec**: [spec.md](../spec.md) (índice normativo) y los 9 specs atómicos: [CU01](../m3-cu01-lista-galpones/spec.md), [CU03](../m3-cu03-generar-liquidacion/spec.md), [CU04](../m3-cu04-anular-liquidacion/spec.md), [CU05](../m3-cu05-desglose-ventas-gastos/spec.md), [CU06](../m3-cu06-historial-liquidaciones/spec.md), [CU07](../m3-cu07-consultar-galpon-lote-m1/spec.md), [CU08](../m3-cu08-consultar-resultado-sacrificio-m2/spec.md), [CU09](../m3-cu09-consultar-alimento-requerido-m2/spec.md), [CU10](../m3-cu10-consultar-consumo-medicamento-m2/spec.md)

## Summary

El Módulo 3 genera la **Liquidación** de un lote: toma el resultado final de sacrificio (Módulo 2), el precio por kilogramo que ingresa el Administrador Financiero y los costos del ciclo (Módulo 1 y Módulo 2), y calcula Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta. La Liquidación es inmutable, única `ACTIVA` por lote y solo se corrige por anulación. Cuatro casos de uso de sincronización (CU07–CU10) mantienen una copia local de M1 y M2; cuatro de consulta y operación (CU01, CU04, CU05, CU06) trabajan sobre la Liquidación y esa copia.

Se construye como el `pom.xml` y `AGENTS.md` §3 lo fijan hoy: **Java 17, Maven, JUnit 5, sin framework web, sin base de datos y sin dependencias nuevas**. Las reglas y fórmulas viven en clases Java puras; los accesos a M1/M2 son interfaces con dobles en memoria para las pruebas; la persistencia es en memoria detrás de interfaces. Cada acceptance scenario de los specs es un `@Test` con los valores del spec, y cada decisión abierta D-01..D-07 queda aislada en una única clase con un supuesto explícito.

## Technical Context

**Language/Version**: Java 17 (`maven.compiler.release` 17), Maven, `maven-surefire-plugin` 3.2.5.
**Primary Dependencies**: ninguna en `main`. `org.junit.jupiter:junit-jupiter` 5.10.2 en `test`. No se agregan dependencias (`AGENTS.md` §3).
**Storage**: memoria. Repositorios como interfaces con implementación `InMemory…` en `main`. La persistencia definitiva es una decisión pendiente del equipo.
**Testing**: JUnit 5 con `mvn test`. Sin Mockito: los dobles de M1/M2 son clases en `src/test`.
**Target Platform**: JVM 17, `jar` de librería. Sin interfaz de usuario en este plan.
**Project Type**: single.
**Performance Goals**: no aplican en esta etapa (los SC de tiempo se verifican cuando exista interfaz y persistencia).
**Constraints**: todo valor monetario y todo kilogramo es `BigDecimal`; totales COP con `setScale(0, HALF_UP)` y porcentajes con `setScale(2, HALF_UP)` (RT-01, RT-02), ambos solo en `shared.Rounding`. Ninguna operación funcional consulta en vivo a M1/M2 (solo `sync` conoce los gateways). M3 no escribe en M1/M2 salvo el aviso de utilización (CU08.FR-006). "Ahora" viene de un `java.time.Clock` inyectado. Código en inglés sin comentarios; entidades del glosario y atributos de las Key Entities con su nombre del spec. Sin pruebas de UI, concurrencia, tiempos ni Excel real (`AGENTS.md` §9).
**Scale/Scope**: una granja, decenas de galpones, un usuario fijo, 9 casos de uso, 54 acceptance scenarios.

## Project Structure

### Documentation (this feature)

```text
docs/specs/features/modulo3/
├── spec.md                      # índice normativo
├── plan/
│   ├── plan-v1.md               # versión anterior (superada)
│   └── plan-v2.md               # este archivo (plan general de los 9 CU)
├── CAMBIOS.md
└── m3-cuNN-*/spec.md            # un spec autocontenido por caso de uso
```

### Source Code (repository root)

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── shared/
│   └── Rounding.java                       # copTotal(BigDecimal), percentage(BigDecimal)
├── sync/
│   ├── FuenteSincronizacion.java           # MODULO_1 | MODULO_2
│   ├── RegistroSincronizacion.java         # bitácora (Key Entity CU07–CU10) + incidencias
│   ├── ResultadoSincronizacion.java        # EXITOSA | FALLIDA
│   ├── RegistroSincronizacionRepository.java (+ InMemory…)
│   ├── SyncStatus.java · SyncStatusQuery.java   # última sincronización exitosa por fuente (RT-09)
│   ├── ModuleUnavailableException.java
│   ├── m1/
│   │   ├── Module1Gateway.java             # puerto de lectura hacia M1 (CU07)
│   │   ├── RemoteGalpon.java · RemoteLote.java · RemoteAlertaVaciado.java
│   │   ├── EstadoGalpon.java               # 6 estados, grafía de M1 (D-07)
│   │   ├── Galpon.java · Lote.java · AlertaVaciadoSanitario.java
│   │   ├── CopiaLocalGalponLote.java (+ InMemory…)   # Key Entity CU07/CU01
│   │   └── Module1SyncService.java         # CU07
│   └── m2/
│       ├── Module2Gateway.java             # puerto de lectura + aviso de utilización (CU08–CU10)
│       ├── RemoteResultadoSacrificio.java · RemotePartidaAlimento.java
│       ├── RemoteConsumoMedicamento.java · RemoteTramoRecepcion.java
│       ├── ResultadoFinalSacrificio.java · EstadoUtilizacionLocal.java
│       ├── AvisoUtilizacionResultado.java · EstadoEntrega.java
│       ├── PartidaAlimentoLote.java · EstadoValorizacion.java
│       ├── ConsumoMedicamentoLote.java · TramoRecepcionConsumo.java
│       ├── ResultadoFinalSacrificioRepository.java (+ InMemory…)
│       ├── AvisoUtilizacionResultadoRepository.java (+ InMemory…)
│       ├── PartidaAlimentoLoteRepository.java (+ InMemory…)
│       ├── ConsumoMedicamentoLoteRepository.java (+ InMemory…)
│       ├── SacrificioSyncService.java      # CU08
│       ├── UsageNoticeDispatcher.java      # CU08 FR-006/FR-007
│       ├── AlimentoSyncService.java        # CU09
│       └── MedicamentoSyncService.java     # CU10
├── domain/
│   ├── Liquidacion.java                    # único Documento Financiero; único mutador: anular()
│   ├── EstadoLiquidacion.java              # ACTIVA | ANULADA
│   ├── PartidaCostoLote.java · CategoriaCosto.java    # ALIMENTO | MEDICINA | POBLACION
│   ├── SnapshotDatosOrigen.java · FuenteOrigen.java
│   ├── RegistroAnulacion.java
│   ├── LiquidacionCalculator.java          # fórmulas FR-001..FR-004 de CU03
│   ├── CostValuation.java                  # subtotal de una partida (punto único de D-05)
│   ├── LiquidacionEligibility.java · Eligibility.java   # precondiciones compartidas CU01/CU03
│   └── LiquidacionRepository.java (+ InMemory…)         # garantiza CA-G04 en save
└── application/
    ├── ListGalponesService.java            # CU01  → GalponRow, GalponListResult
    ├── GenerateLiquidacionService.java     # CU03  → GenerateLiquidacionCommand
    ├── AnnulLiquidacionService.java        # CU04  → AnulacionPendiente
    ├── DesgloseQueryService.java           # CU05  → DesgloseView, DesgloseLine, DesgloseExporter (puerto)
    ├── LiquidacionHistoryService.java      # CU06  → HistoryFilter, HistoryRow, HistoryResult, EmptyReason
    └── excepciones: MissingSyncedDataException, LoteNotLiquidableException,
        ActiveLiquidacionExistsException, InvalidPrecioKgException, PendingValuationException,
        NoCostItemsException, InvalidAnulacionMotivoException, LiquidacionAlreadyAnuladaException,
        ExportNotAllowedException, ExportFailedException, InvalidDateRangeException

src/test/java/co/edu/unimagdalena/avicontrol/
├── support/
│   ├── FakeModule1Gateway.java · FakeModule2Gateway.java   # modos COMPLETO | DATOS_FALTANTES | AUSENTE (CA-G07)
│   ├── FakeDesgloseExporter.java
│   ├── CasoDorado.java                     # fixture con los valores exactos de los specs
│   └── TestContext.java                    # cablea reloj fijo, repositorios en memoria, gateways falsos y servicios
├── shared/RoundingTest.java
├── domain/LiquidacionCalculatorTest.java · InMemoryLiquidacionRepositoryTest.java
├── sync/m1/CU07ConsultarGalponLoteM1Test.java
├── sync/m2/CU08ConsultarResultadoSacrificioM2Test.java · CU09ConsultarAlimentoRequeridoM2Test.java · CU10ConsultarConsumoMedicamentoM2Test.java
└── application/CU01ConsultarListaGalponesTest.java · CU03GenerarLiquidacionTest.java · CU04AnularLiquidacionTest.java · CU05ConsultarDesgloseTest.java · CU06ConsultarHistorialTest.java
```

**Structure Decision**: los cuatro paquetes de `AGENTS.md` §9, con `sync` dividido en `sync.m1` y `sync.m2` porque cada uno es la copia local de una fuente con su propio gateway. Las implementaciones `InMemory…` van en `main` porque hoy son el almacenamiento real; los dobles de M1/M2 van en `test` porque simulan sistemas externos. No existen `MatrizVentas`, `Reporte`, `Venta` ni `Desglose` como entidades: la Matriz de Venta Final son los atributos de `Liquidacion` (CU03.FR-013) y el Desglose es el record `DesgloseView` que devuelve un servicio y no se persiste (CU05.FR-007).

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Dejar la estructura de paquetes y el redondeo listos.

- [ ] T001 Crear los paquetes `shared`, `sync`, `sync/m1`, `sync/m2`, `domain`, `application` en `src/main` y `support`, `shared`, `domain`, `sync/m1`, `sync/m2`, `application` en `src/test`; eliminar los `.gitkeep`
- [ ] T002 [P] Crear `shared/Rounding.java` con `copTotal(BigDecimal)` → `setScale(0, HALF_UP)` y `percentage(BigDecimal)` → `setScale(2, HALF_UP)`; ningún otro código llama a `setScale`
- [ ] T003 [P] Escribir `RoundingTest`: `.5` hacia arriba, negativos, cocientes no exactos (500 / 9.000 × 100 → 5,56)
- [ ] T004 Verificar que `mvn test` compila y ejecuta

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Bitácora, puertos hacia M1/M2, estado de sincronización y soporte de pruebas que todos los CU usan.

**⚠️ CRITICAL**: Ningún CU empieza hasta terminar esta fase.

- [ ] T005 [P] Crear en `sync/` `FuenteSincronizacion`, `ResultadoSincronizacion`, `RegistroSincronizacion` (record con `idSincronizacion`, `fuente`, `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados` y `incidencias: List<String>` para los rechazos que los specs piden "registrar como incidencia"), `RegistroSincronizacionRepository` e `InMemoryRegistroSincronizacionRepository`
- [ ] T006 [P] Crear `sync/ModuleUnavailableException.java` (checked, con fuente y causa)
- [ ] T007 [P] Crear `sync/m1/Module1Gateway.java` con `fetchGalpones()`, `fetchLotesActivos()`, `fetchAlertasVaciadoSanitario()`, todos `throws ModuleUnavailableException`, y los records `RemoteGalpon`, `RemoteLote`, `RemoteAlertaVaciado` con los campos de CU07.FR-002..FR-004 como tipos nulos permitidos (`Long`, `BigDecimal`, `String`) para simular datos faltantes; el estado llega como `String` con la grafía de M1
- [ ] T008 [P] Crear `sync/m2/Module2Gateway.java` con `fetchResultadosSacrificio()`, `fetchPartidasAlimento(UUID idLote)`, `fetchConsumosMedicamento(UUID idLote)` y `notifyResultUsage(UUID idResultado, UUID idLiquidacion)`, todos `throws ModuleUnavailableException`, y los records `RemoteResultadoSacrificio`, `RemotePartidaAlimento`, `RemoteConsumoMedicamento`, `RemoteTramoRecepcion` con los campos de CU08–CU10.FR-002/FR-003 y nulos permitidos
- [ ] T009 Crear `sync/SyncStatus.java` (última sincronización exitosa de M1 y de M2, y si la última ejecución de cada una falló) y `SyncStatusQuery.current()` a partir de la bitácora (RT-09, CU07.FR-005) (depende de T005)
- [ ] T010 [P] Crear `test/support/FakeModule1Gateway` y `FakeModule2Gateway` con los tres modos de CA-G07 (`COMPLETO`, `DATOS_FALTANTES`, `AUSENTE` → lanza `ModuleUnavailableException`), listas mutables de datos remotos y, en M2, la lista de avisos recibidos y un modo de fallo del aviso
- [ ] T011 [P] Crear `test/support/CasoDorado.java` y `TestContext.java`. `CasoDorado` construye los valores exactos de los specs: población inicial 9.000 y actual 8.500 (CU03 AS-01); resultado 8.500 pollos y 23.800 kg (CU03 AS-01, CU08 AS-01); precio $4.500; y el desglose de costos que suma los **$85.000.000** del spec: costo total del lote $27.000.000; alimento Pre-inicio 4.000 kg × $2.500, Inicio 8.000 kg × $2.400, Engorde 12.000 kg × $2.300; un consumo de medicamento abastecido por dos recepciones con precios distintos, 400 ml × $1.800 y 300 ml × $1.600 (CU10 AS-02). Variante siniestro total (CU03 AS-06): población actual 0, sin resultado ni precio, costo del lote $27.000.000 + alimento 5.000 kg × $2.600 = **$40.000.000**. Variantes con partida sin precio y sin partidas. El desglose lo fija este plan (los specs solo dan totales) y debe validarse con el stakeholder (CA-G03)

**Checkpoint**: la bitácora registra ejecuciones, los puertos existen y los dobles simulan los tres modos. Los CU pueden empezar.

---

## Phase 3: CU07 – Consultar Galpón y Lote al Módulo 1 (Priority: P1)

**Goal**: Mantener la copia local de galpones, lotes activos y alertas de vaciado sanitario con fecha y hora de sincronización, sin transformar valores, tolerando fallos de M1. Cubre CU07.FR-001..FR-011.

**Independent Test**: `Module1SyncService.synchronize()` contra `FakeModule1Gateway` en modo `COMPLETO` deja la copia local poblada; en modo `AUSENTE` la copia previa queda intacta y la bitácora registra `FALLIDA`.

**Supuestos sobre decisiones abiertas**: D-02 → el contrato con M1 es `Module1Gateway`; se asume que los campos de FR-002..FR-004 existen tal cual. D-04 y D-06 → no se implementa nada; la sincronización almacena poblaciones y estados tal como M1 los informa (FR-009). D-07 → `EstadoGalpon.fromLabel` acepta la grafía de M1 (`"En cosecha"`, `"Vaciado sanitario"`) sin distinguir mayúsculas; cualquier otra cadena es incidencia (FR-011).

### Tests for CU07

- [ ] T012 [CU07] `CU07ConsultarGalponLoteM1Test` con un `@Test` por scenario, en orden: `CU07-AS-01` sincronización inicial exitosa · `CU07-AS-02` incremental con cambio de población (inicial intacta) · `CU07-AS-03` alerta de vaciado con lote desvinculado que sigue en la copia · `CU07-AS-04` M1 no responde con copia previa (copia íntegra, bitácora `FALLIDA`) · `CU07-AS-05` M1 no responde sin copia previa (`findLote` vacío; el bloqueo de la operación se prueba en CU03 vía `MissingSyncedDataException`) · `CU07-AS-06` idempotencia (misma alerta dos veces → un registro, fecha actualizada). Más un test de FR-010/FR-011: lote sin población inicial, costo no positivo, alerta sin lote y estado desconocido generan incidencia sin sobrescribir el registro válido previo

### Implementation for CU07

- [ ] T013 [P] [CU07] Crear `EstadoGalpon` (`DISPONIBLE`, `PRODUCTIVO`, `EN_COSECHA`, `VACIADO_SANITARIO`, `MANTENIMIENTO`, `AISLAMIENTO`, cada uno con su etiqueta de M1) y `fromLabel(String) → Optional<EstadoGalpon>`
- [ ] T014 [P] [CU07] Crear los records `Galpon`, `Lote` (con `costoTotalCop: BigDecimal` entero, `poblacionInicial`/`poblacionActual: long`) y `AlertaVaciadoSanitario`, todos con `fechaHoraSincronizacion`
- [ ] T015 [CU07] Crear `CopiaLocalGalponLote` (`upsertGalpon`, `upsertLote`, `upsertAlerta`, `findGalpon`, `findLote`, `findLoteActivoByGalpon`, `findAlertaByLote`, `allGalpones`, `isEmpty`) e `InMemoryCopiaLocalGalponLote` con `upsert` por id (FR-007) (depende de T014)
- [ ] T016 [CU07] Implementar `Module1SyncService.synchronize()`: llama a los tres `fetch`, valida cada registro, incorpora los válidos, acumula incidencias, escribe la bitácora; ante `ModuleUnavailableException` registra `FALLIDA` sin tocar la copia (FR-006) (depende de T015)

**Checkpoint**: CU01 y CU03 tienen de dónde leer galpones, lotes y alertas.

---

## Phase 4: CU08 – Consultar Resultado Final de Sacrificio al Módulo 2 (Priority: P1)

**Goal**: Sincronizar el resultado final de sacrificio como unidad indivisible, validarlo, admitir correcciones hasta que se use y emitir el aviso de utilización con reintento. Cubre CU08.FR-001..FR-010.

**Independent Test**: `SacrificioSyncService.synchronize()` con un resultado de 8.500 pollos y 23.800 kg lo deja en la copia local `NO_UTILIZADO`; `UsageNoticeDispatcher.dispatchPending()` entrega los avisos `PENDIENTE` y, si el gateway falla, los conserva `PENDIENTE` con `intentos` incrementado.

**Supuestos sobre decisiones abiertas**: D-03 → el aviso es `Module2Gateway.notifyResultUsage(idResultado, idLiquidacion)`, sin confirmación de vuelta; se persiste como `AvisoUtilizacionResultado` y lo reintenta `UsageNoticeDispatcher` en cada ciclo, idempotente por `idAviso`.

### Tests for CU08

- [ ] T017 [CU08] `CU08ConsultarResultadoSacrificioM2Test`: `CU08-AS-01` sincronización exitosa · `CU08-AS-02` rechazo de resultado inválido (cero, negativo, ausente) → incidencia, no habilita · `CU08-AS-03` corrección antes de ser utilizado → copia actualizada · `CU08-AS-04` aviso de utilización tras generar la Liquidación (resultado `UTILIZADO`, el gateway falso recibe `idResultado`) · `CU08-AS-05` fallo del aviso (Liquidación conservada, aviso `PENDIENTE`, reintento en `dispatchPending()`) · `CU08-AS-06` M2 no responde con copia previa (la generación usa la copia). AS-04..AS-06 necesitan `GenerateLiquidacionService`: se escriben aquí con `@Disabled("Fase 8")` y se activan en T036. Más un test de FR-010 (segundo resultado para el mismo lote → incidencia, se conserva el previo)

### Implementation for CU08

- [ ] T018 [P] [CU08] Crear `ResultadoFinalSacrificio` (record con `withEstadoUtilizacion`), `EstadoUtilizacionLocal`, `AvisoUtilizacionResultado`, `EstadoEntrega`
- [ ] T019 [CU08] Crear `ResultadoFinalSacrificioRepository` (`upsert`, `findByLote`, `markUtilizado`) y `AvisoUtilizacionResultadoRepository` (`save`, `findPendientes`) con sus `InMemory…` (depende de T018)
- [ ] T020 [CU08] Implementar `SacrificioSyncService.synchronize()`: valida `cantidadFinalPollos > 0` y `pesoTotalKg > 0` (FR-003), actualiza solo si `NO_UTILIZADO` (FR-004), duplicado por lote → incidencia (FR-010), fallo de M2 → `FALLIDA` sin tocar la copia (FR-008) (depende de T019)
- [ ] T021 [CU08] Implementar `UsageNoticeDispatcher.dispatchPending()`: por cada aviso `PENDIENTE` llama a `notifyResultUsage`; éxito → `ENTREGADO`; fallo → sigue `PENDIENTE`, `intentos + 1` (FR-007) (depende de T019)

**Checkpoint**: la Venta Bruta tiene su insumo físico y el aviso hacia M2 tiene su mecanismo.

---

## Phase 5: CU09 – Consultar Alimento Requerido al Módulo 2 (Priority: P1)

**Goal**: Sincronizar las partidas de alimento con cantidad y precio unitario, sin calcular ni promediar, marcando las que carecen de precio. Cubre CU09.FR-001..FR-009.

**Independent Test**: `AlimentoSyncService.synchronize()` deja las tres partidas del Caso Dorado `VALORIZADA`; una partida sin precio queda `PENDIENTE_PRECIO`; un lote sin partidas no genera registros.

**Supuestos sobre decisiones abiertas**: D-01 → la fuente que M2 decida (compras, consumo real o proyección) se mapea en `Module2Gateway.fetchPartidasAlimento` a `PartidaAlimentoLote` con `cantidadKg` y `precioUnitarioKgCop`; el resto del módulo no distingue el criterio. D-05 → `valorImpuestoCop` se almacena aparte y no se consolida.

### Tests for CU09

- [ ] T022 [CU09] `CU09ConsultarAlimentoRequeridoM2Test`: `CU09-AS-01` sincronización exitosa · `CU09-AS-02` partida sin precio → `PENDIENTE_PRECIO` · `CU09-AS-03` ausencia total (cero partidas, hecho en bitácora) · `CU09-AS-04` M2 no responde con copia previa (`@Disabled("Fase 8")`) · `CU09-AS-05` idempotencia. Más un test de que dos partidas del mismo tipo con precios distintos se conservan por separado (FR-004)

### Implementation for CU09

- [ ] T023 [P] [CU09] Crear `PartidaAlimentoLote` (record con los atributos de la Key Entity) y `EstadoValorizacion` (`VALORIZADA`, `PENDIENTE_PRECIO`; compartido con CU10)
- [ ] T024 [CU09] Crear `PartidaAlimentoLoteRepository` (`upsert`, `findByLote`) e `InMemoryPartidaAlimentoLoteRepository` (depende de T023)
- [ ] T025 [CU09] Implementar `AlimentoSyncService.synchronize()` recorriendo los lotes de `CopiaLocalGalponLote`; sin multiplicar ni promediar (FR-003, FR-004); sin precio → `PENDIENTE_PRECIO` (FR-005); sin partidas → bitácora y cero registros (FR-006) (depende de T024, T015)

**Checkpoint**: el Costo de Alimento tiene sus partidas.

---

## Phase 6: CU10 – Consultar Consumo de Medicamento al Módulo 2 (Priority: P1)

**Goal**: Sincronizar los consumos de medicamento con un tramo por recepción y su precio histórico, sin promediar ni calcular. Cubre CU10.FR-001..FR-010.

**Independent Test**: un consumo abastecido por dos recepciones con precios distintos queda con dos `TramoRecepcionConsumo`, cada uno con su precio; un lote sin consumos no genera registros ni bloquea.

**Supuestos sobre decisiones abiertas**: D-05 → igual que CU09: `valorImpuestoCop` por tramo, sin consolidar.

### Tests for CU10

- [ ] T026 [CU10] `CU10ConsultarConsumoMedicamentoM2Test`: `CU10-AS-01` una recepción · `CU10-AS-02` dos recepciones con precios distintos, sin promediar · `CU10-AS-03` tramo sin precio → `PENDIENTE_PRECIO` · `CU10-AS-04` ausencia total (cero consumos, no bloquea) · `CU10-AS-05` M2 no responde con copia previa (`@Disabled("Fase 8")`) · `CU10-AS-06` idempotencia

### Implementation for CU10

- [ ] T027 [P] [CU10] Crear `ConsumoMedicamentoLote` (con `tramos: List<TramoRecepcionConsumo>`) y `TramoRecepcionConsumo` (records con los atributos de las Key Entities)
- [ ] T028 [CU10] Crear `ConsumoMedicamentoLoteRepository` (`upsert`, `findByLote`) e `InMemoryConsumoMedicamentoLoteRepository` (depende de T027)
- [ ] T029 [CU10] Implementar `MedicamentoSyncService.synchronize()` con las mismas reglas de CU09 aplicadas por tramo (depende de T028, T015)

**Checkpoint**: la copia local está completa.

---

## Phase 7: CU01 – Consultar Lista de Galpones (Priority: P1)

**Goal**: Listar galpones y lotes desde la copia local con las acciones de fila habilitadas o deshabilitadas según la precondición de Liquidación, y con el estado de sincronización visible. Cubre CU01.FR-001..FR-007. Crea `Liquidacion`, su repositorio y `LiquidacionEligibility`, que CU03 reutiliza.

**Independent Test**: con la copia local del Caso Dorado, `list()` devuelve una fila por galpón; solo el galpón en `VACIADO_SANITARIO` con alerta y resultado válido tiene `generarLiquidacionHabilitada = true`; con la copia vacía devuelve "No hay galpones disponibles para consultar".

**Supuestos sobre decisiones abiertas**: D-04 → la elegibilidad exige `VACIADO_SANITARIO` + alerta con el `idLote` también para el siniestro total (CU03.FR-005 tal cual); si M1 abre otra ruta, cambia solo el paso 2 de `LiquidacionEligibility`. D-07 → se compara contra el enum, no contra cadenas.

### Tests for CU01

- [ ] T030 [CU01] `CU01ConsultarListaGalponesTest`: `CU01-AS-01` datos completos con fecha de sincronización · `CU01-AS-02` `EN_COSECHA` → acción deshabilitada, motivo "cierre operativo a cargo de Módulo 2" · `CU01-AS-03` `VACIADO_SANITARIO` con alerta y resultado (o población 0) y sin `ACTIVA` → habilitada · `CU01-AS-04` `DISPONIBLE`/`PRODUCTIVO`/`MANTENIMIENTO`/`AISLAMIENTO` → deshabilitada, informativa · `CU01-AS-05` copia vacía → mensaje literal y todo bloqueado · `CU01-AS-06` M1 indisponible tras una sincronización previa → misma lista, fecha de la última exitosa, `modulo1EnFallo = true`. Más el edge "población 0 en `PRODUCTIVO`/`EN_COSECHA`" deshabilitado
- [ ] T031 [CU01] `InMemoryLiquidacionRepositoryTest` con `@DisplayName("CA-G04 – …")`: guardar una segunda `ACTIVA` para el mismo lote lanza

### Implementation for CU01

- [ ] T032 [P] [CU01] Crear en `domain/` `Liquidacion` (atributos de la Key Entity de CU03 más `partidas: List<PartidaCostoLote>`, `snapshots: List<SnapshotDatosOrigen>` y `registroAnulacion`; todos los campos `final` salvo `estado` y `registroAnulacion`; sin setters; `idLiquidacion` generado con `UUID.randomUUID()`), `EstadoLiquidacion`, `PartidaCostoLote` (record con los atributos de la Key Entity más `impuestoCop` aparte; `subtotalCop` entero), `CategoriaCosto`, `SnapshotDatosOrigen` (record: `fuente`, `fechaHoraSincronizacion`, `referencias: List<UUID>`), `FuenteOrigen`, `RegistroAnulacion` (record)
- [ ] T033 [CU01] Crear `LiquidacionRepository` (`save`, `findById`, `findByLote`, `findActivaByLote`, `findAll`) e `InMemoryLiquidacionRepository` que lanza si se guarda una segunda `ACTIVA` para el mismo lote (CA-G04) (depende de T032)
- [ ] T034 [CU01] Crear `LiquidacionEligibility.evaluate(idLote) → Eligibility(liquidable, siniestroTotal, motivoBloqueo, liquidacionExistente)` aplicando en orden: (1) existe `Lote` en la copia local, si no → "no hay datos sincronizados disponibles para ese lote"; (2) galpón en `VACIADO_SANITARIO` y `AlertaVaciadoSanitario` con ese `idLote`, si no → mensaje de CU03 AS-07; (3) sin `Liquidacion` `ACTIVA` para el lote, si hay → mensaje de CU03 AS-08 con la existente; (4) `poblacionActual == 0` → `siniestroTotal`, liquidable sin resultado; si no, debe existir `ResultadoFinalSacrificio` válido, si no → mensaje de CU03 AS-02 (depende de T033, T015, T019)
- [ ] T035 [CU01] Implementar `ListGalponesService.list() → GalponListResult(rows, syncStatus, emptyMessage)`; cada `GalponRow` trae los campos de FR-001 (sin aforo, edad ni poblaciones), `generarLiquidacionHabilitada`, `consultarDesgloseHabilitada` (si `findByLote` no está vacío) y `motivoBloqueo`; acciones deshabilitadas = bandera en `false`, nunca campo omitido (FR-005) (depende de T034, T009)

**Checkpoint**: el punto de entrada del módulo funciona sobre la copia local y existe la regla de elegibilidad que CU03 reutiliza.

---

## Phase 8: CU03 – Generar Liquidación del Lote (Priority: P1)

**Goal**: Generar la Liquidación `ACTIVA` con los cuatro indicadores exactos, el snapshot por fuente, el marcado del resultado como utilizado y el aviso a M2. Cubre CU03.FR-001..FR-013.

**Independent Test**: con `CasoDorado` completo y precio $4.500, `generate` devuelve Venta Bruta $107.100.000, Mortalidad 500 (5,56 %), Costos Operativos $85.000.000, Utilidad Neta $22.100.000, peso promedio 2,80, estado `ACTIVA`; el resultado queda `UTILIZADO` y el gateway falso de M2 recibió el aviso.

**Supuestos sobre decisiones abiertas**: D-05 → `CostValuation.subtotal` calcula sobre valor neto; el impuesto queda en `PartidaCostoLote.impuestoCop` para poder sumarlo después sin re-sincronizar. D-06 → `mortalidadAves = poblacionInicial − poblacionActual` tal como M1 las informa, sin corrección. D-01 → `GenerateLiquidacionService` valoriza las partidas que haya, sea cual sea su criterio.

### Tests for CU03

- [ ] T036 [CU03] `LiquidacionCalculatorTest`: Venta Bruta 23.800 × 4.500 = 107.100.000 y con precio decimal 23.800 × 4.500,50 = 107.111.900; mortalidad 500 y 5,56 %; siniestro total 9.000 y 100,00 %; utilidad negativa; peso promedio 2,80
- [ ] T037 [CU03] `CU03GenerarLiquidacionTest` con diez `@Test` en el orden del spec: `CU03-AS-01` Caso Dorado · `CU03-AS-02` sin resultado → mensaje literal · `CU03-AS-03` precio ≤ 0 → `InvalidPrecioKgException` con el campo · `CU03-AS-04` partida sin precio → lista de pendientes, nada guardado · `CU03-AS-05` sin partidas → "No hay partidas de costo sincronizadas para este ciclo" · `CU03-AS-06` siniestro total (0 / 9.000 / 100,00 % / $40.000.000 / −$40.000.000) · `CU03-AS-07` `EN_COSECHA` con resultado → bloqueo · `CU03-AS-08` segunda `ACTIVA` → deniega y devuelve la existente · `CU03-AS-09` tras anulación (`@Disabled("Fase 9")`) · `CU03-AS-10` M1/M2 indisponibles con copia local → genera y el snapshot conserva las fechas. Más `@DisplayName("CA-G04 – …")` a nivel de servicio y un test de que un lote sin medicamentos sí liquida (CU10.FR-007)
- [ ] T038 [CU03] Activar los `@Disabled` de CU08 AS-04/05/06, CU09 AS-04 y CU10 AS-05

### Implementation for CU03

- [ ] T039 [P] [CU03] Implementar `LiquidacionCalculator` con `ventaBruta(pesoTotalKg, precioKgCop)` = `Rounding.copTotal(peso × precio)`; `mortalidadAves(inicial, actual)`; `porcentajeMortalidad` = `Rounding.percentage(mortalidad × 100 / inicial)`; `costosOperativos` = suma de `subtotalCop`; `utilidadNeta` = ventaBruta − costos; `pesoPromedio` = `pesoTotalKg / pollosVendidos` a 2 decimales (solo presentación, FR-012)
- [ ] T040 [P] [CU03] Implementar `CostValuation.subtotal(cantidad, precioUnitario)` = `Rounding.copTotal(cantidad × precioUnitario)`. Se redondea por partida y el total es la suma, para que CU05.FR-002 cuadre al peso
- [ ] T041 [CU03] Implementar `GenerateLiquidacionService.generate(GenerateLiquidacionCommand(idLote, precioKgCop nullable, usuarioResponsable))`: (5) tras `LiquidacionEligibility`, valida `precioKgCop > 0` salvo siniestro total (FR-011); (6) reúne partidas: alimento (`PartidaAlimentoLote` del lote), medicina (un `TramoRecepcionConsumo` por línea) y población (una sola `PartidaCostoLote` `POBLACION` con `concepto = "Costo total del lote"`, `cantidad = 1`, `unidadMedida = "lote"`, `precioUnitarioCop = subtotalCop = costoTotalCop`, `referenciaOrigen = idLote`); sin partidas de alimento → `NoCostItemsException` (mensaje de AS-05 si tampoco hay medicina; "No hay partidas de alimento sincronizadas para este ciclo" si solo falta alimento); alguna `PENDIENTE_PRECIO` → `PendingValuationException` con la lista (FR-006); (7) valoriza con `CostValuation`, calcula con `LiquidacionCalculator`, arma un `SnapshotDatosOrigen` por fuente con la fecha de la última sincronización exitosa de esa fuente (bitácora) y las referencias usadas (FR-009); (8) persiste la `Liquidacion` `ACTIVA` con fecha, hora y usuario (FR-008); (9) si consumió resultado, lo marca `UTILIZADO`, crea el `AvisoUtilizacionResultado` `PENDIENTE` y llama a `UsageNoticeDispatcher.dispatchPending()`; un fallo del aviso no revierte nada (FR-010, CU08 AS-05) (depende de T034, T039, T040, T021, T024, T028)

**Checkpoint**: el entregable central del módulo funciona de extremo a extremo con la copia local.

---

## Phase 9: CU04 – Anular Liquidación (Priority: P1)

**Goal**: Anular una Liquidación `ACTIVA` con motivo, fecha, hora y responsable, conservándola íntegra y rehabilitando el lote. Cubre CU04.FR-001..FR-010 (incluido FR-002b: el punto de entrada sigue siendo `ActiveLiquidacionExistsException` de CU03, que devuelve la existente).

**Independent Test**: `prepare(id, "Precio por kg digitado con error")` + `confirm` deja la Liquidación `ANULADA` con su `RegistroAnulacion`, todos los demás campos iguales, y `generate` sobre el mismo lote crea una nueva `ACTIVA`.

### Tests for CU04

- [ ] T042 [CU04] `CU04AnularLiquidacionTest`: `CU04-AS-01` anulación exitosa (estado, registro, campos intactos comparados uno a uno, `findActivaByLote` vacío) · `CU04-AS-02` corrección completa de precio (anular + generar; la anterior sigue `ANULADA`) · `CU04-AS-03` motivo vacío o solo espacios → rechazo sin cambios · `CU04-AS-04` ya anulada → deniega y devuelve el registro existente · `CU04-AS-05` `prepare` sin `confirm` → sin cambios ni registro
- [ ] T043 [CU04] Activar `CU03-AS-09`

### Implementation for CU04

- [ ] T044 [CU04] Implementar `Liquidacion.anular(RegistroAnulacion)`: cambia `estado` y adjunta el registro en una sola operación (FR-005, FR-006, FR-008); lanza si ya está `ANULADA` (FR-002)
- [ ] T045 [CU04] Implementar `AnnulLiquidacionService`: `prepare(idLiquidacion, motivo) → AnulacionPendiente(idLiquidacion, idLote, motivo, consecuencias)` valida el motivo recortado entre 10 y 500 caracteres (FR-003) y que la Liquidación esté `ACTIVA`; `confirm(pendiente, usuarioResponsable)` persiste (FR-004). Cancelar es no llamar a `confirm` (depende de T044)

**Checkpoint**: el ciclo `ACTIVA → ANULADA → nueva ACTIVA` funciona (CA-G08).

---

## Phase 10: CU05 – Consultar Desglose de Ventas y Gastos (Priority: P2)

**Goal**: Proyectar el desglose por categorías desde la Liquidación y sus partidas, con suma exacta a `costosOperativosCop`, y dejar el puerto de exportación. Cubre CU05.FR-001..FR-007; FR-003 (archivo `.xlsx`) solo como puerto, porque implementarlo exige Apache POI y `AGENTS.md` §3 lo prohíbe sin acuerdo.

**Independent Test**: `view(id)` del Caso Dorado devuelve 3 líneas `ALIMENTO`, 2 `MEDICINA`, 1 `POBLACION` cuya suma es $85.000.000; `export` sobre una `ANULADA` lanza `ExportNotAllowedException`.

### Tests for CU05

- [ ] T046 [CU05] `CU05ConsultarDesgloseTest`: `CU05-AS-01` desglose por categorías con suma exacta a `costosOperativosCop` · `CU05-AS-02` exportación: `FakeDesgloseExporter` recibe la misma `DesgloseView` de pantalla (Matriz de Venta Final + desglose); la igualdad con el archivo real (CA-G06) queda para cuando exista implementación · `CU05-AS-03` anulada → solo lectura con `registroAnulacion` y exportación denegada · `CU05-AS-04` exportador falla → `ExportFailedException`, `view` sigue devolviendo lo mismo · `CU05-AS-05` siniestro total → ingresos con Venta Bruta 0 y aviso de que el lote no registró venta

### Implementation for CU05

- [ ] T047 [P] [CU05] Crear `DesgloseView` (ingresos: `pollosVendidos`, `pesoTotalKg`, `pesoPromedioKg`, `precioKgCop`, `ventaBrutaCop`; líneas por categoría; `costosOperativosCop`; `estado`; `registroAnulacion`; `syncStatus`), `DesgloseLine` (`categoria`, `concepto`, `cantidad`, `unidadMedida`, `precioUnitarioCop`, `subtotalCop`, `fuenteOrigen`, `referenciaOrigen`), `DesgloseExporter` (`void export(DesgloseView, String usuario)`), `ExportNotAllowedException`, `ExportFailedException`
- [ ] T048 [P] [CU05] Crear `test/support/FakeDesgloseExporter` (captura la vista; modo de fallo)
- [ ] T049 [CU05] Implementar `DesgloseQueryService.view(idLiquidacion)` (solo desde `LiquidacionRepository`, FR-006; nada se persiste, FR-007) y `export(idLiquidacion, usuario)` (rechaza `ANULADA`, FR-004; envuelve fallos, FR-005) (depende de T047)

**Checkpoint**: cada línea del desglose lleva `fuenteOrigen` y `referenciaOrigen` (CA-G05).

---

## Phase 11: CU06 – Consultar Historial de Liquidaciones (Priority: P3)

**Goal**: Listar Liquidaciones en orden descendente con filtros por galpón y rango de fechas, distinguiendo anuladas y los dos estados vacíos. Cubre CU06.FR-001..FR-008. La paginación (edge "alto volumen") queda para la versión con persistencia.

**Independent Test**: con tres Liquidaciones (dos galpones, una anulada), `query` sin filtros las devuelve en orden descendente; con filtro de galpón devuelve solo las suyas; `desde > hasta` lanza `InvalidDateRangeException`.

### Tests for CU06

- [ ] T050 [CU06] `CU06ConsultarHistorialTest`: `CU06-AS-01` orden descendente y columnas de FR-002 · `CU06-AS-02` filtro por galpón y rango · `CU06-AS-03` anuladas identificables con su registro · `CU06-AS-04` vacío total vs. sin coincidencias (`NO_LIQUIDACIONES` / `NO_MATCHES`) · `CU06-AS-05` rango invertido rechazado. Más el edge "lote desvinculado sigue filtrable por galpón"

### Implementation for CU06

- [ ] T051 [P] [CU06] Crear `HistoryFilter(idGalpon nullable, desde nullable, hasta nullable)`, `HistoryRow` (columnas de FR-002 + `idLiquidacion` para abrir CU05 + `registroAnulacion`), `HistoryResult(rows, emptyReason, syncStatus)`, `EmptyReason` (`NONE`, `NO_LIQUIDACIONES`, `NO_MATCHES`), `InvalidDateRangeException`
- [ ] T052 [CU06] Implementar `LiquidacionHistoryService.query(filter)`: valida `desde <= hasta` (FR-005), filtra y ordena por `fechaHoraGeneracion` descendente (FR-001, FR-003), solo desde `LiquidacionRepository` (FR-008) (depende de T051)

**Checkpoint**: los 9 casos de uso operan con `mvn test` sobre la copia local.

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: Cerrar la trazabilidad y lo diferido.

- [ ] T053 Recorrer los 9 specs y verificar que cada acceptance scenario tiene su `@Test` con el `@DisplayName` correcto y en el orden del spec (CA-G02); que no queda ningún `@Disabled`
- [ ] T054 Implementar el proceso automático con intervalo configurable que piden los FR-001 de CU07–CU10 (`sync/SyncScheduler` sobre `ScheduledExecutorService` encadenando los cuatro `synchronize()` y `dispatchPending()`); sin pruebas, por estar la concurrencia fuera de alcance
- [ ] T055 Pegar el resumen de surefire en el PR de cierre (`AGENTS.md` §12)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup y bloquea todos los CU.
- **CU07 (Phase 3)**: depende de Foundational. Bloquea a CU09 y CU10 (recorren los lotes de la copia local) y a CU01.
- **CU08, CU09, CU10 (Phases 4–6)**: dependen de Foundational (CU09 y CU10 también de CU07). Independientes entre sí; se pueden repartir en paralelo.
- **CU01 (Phase 7)**: depende de CU07 y CU08. Crea `Liquidacion`, su repositorio y `LiquidacionEligibility`, que CU03 reutiliza.
- **CU03 (Phase 8)**: depende de CU01 y de las cuatro sincronizaciones. Activa los tests diferidos de CU08/CU09/CU10.
- **CU04 (Phase 9)**: depende de CU03. Activa `CU03-AS-09`.
- **CU05 y CU06 (Phases 10–11)**: dependen de CU03 y CU04; independientes entre sí, en paralelo.
- **Polish (Phase 12)**: depende de todo lo anterior.

Es el orden de `spec.md` y `AGENTS.md` §5: **CU07 → CU10 → CU01 → CU03 → CU04 → CU05 ∥ CU06**.

### User Story Dependencies

- **CU07**: solo de Foundational.
- **CU08**: solo de Foundational; sus AS-04..AS-06 se completan con CU03.
- **CU09, CU10**: de CU07 (`CopiaLocalGalponLote`); su AS de "M2 no responde con copia previa" se completa con CU03.
- **CU01**: de CU07 y CU08.
- **CU03**: de CU01, CU07, CU08, CU09, CU10.
- **CU04**: de CU03.
- **CU05, CU06**: de CU03 y CU04 (muestran el registro de anulación).

### Within Each User Story

- Entidades y repositorio antes que el servicio.
- Test de acceptance escrito primero con los valores del spec; se ejecuta al terminar la implementación.
- `mvn test` en verde antes de abrir el PR de la rama `feature/cuNN-…` hacia `develop`.
- Un CU por rama; los tests `@Disabled("Fase N")` se activan en la rama del CU que los completa.

## Notes

- `[P]` = puede hacerse en paralelo con otras tareas `[P]` del mismo bloque.
- `[CUNN]` enlaza la tarea con su caso de uso y con la rama `feature/cuNN-…`.
- Cada acceptance scenario es un `@Test` con `@DisplayName("CUNN-AS-XX – <título del scenario>")`, en el orden del spec y con sus valores exactos; los mensajes de las excepciones son los textos literales de los scenarios.
- Ninguna decisión D-01..D-07 se resuelve en código: cada fase declara su supuesto y el punto único donde cambiarlo. Si el equipo resuelve una, se actualizan spec, `spec.md`, `CAMBIOS.md` y `AGENTS.md` §7 (`AGENTS.md` §8).
- El desglose de partidas de `CasoDorado` (T011) lo fija este plan; hay que validarlo con el stakeholder antes de dar por cumplido CA-G03.
- Cada CU debe poder completarse y probarse por separado con los dobles de `test/support`.
- Detenerse en cada checkpoint para validar el CU con el equipo.
- Si al implementar aparece algo que contradice un spec, no se cambia el código hacia el documento raíz: se abre el punto en `CAMBIOS.md` (`AGENTS.md` §2).
