# Implementation Plan: AVICONTROL Módulo 3 – Liquidación de Lote y Análisis de Rentabilidad

**Date**: 2026-09-21
**Versión**: 1.0 (primera versión completa; sin implementación aún)
**Spec**: [spec.md](../spec.md) (índice normativo) y los 9 specs atómicos: [CU01](../m3-cu01-lista-galpones/spec.md), [CU03](../m3-cu03-generar-liquidacion/spec.md), [CU04](../m3-cu04-anular-liquidacion/spec.md), [CU05](../m3-cu05-desglose-ventas-gastos/spec.md), [CU06](../m3-cu06-historial-liquidaciones/spec.md), [CU07](../m3-cu07-consultar-galpon-lote-m1/spec.md), [CU08](../m3-cu08-consultar-resultado-sacrificio-m2/spec.md), [CU09](../m3-cu09-consultar-alimento-requerido-m2/spec.md), [CU10](../m3-cu10-consultar-consumo-medicamento-m2/spec.md)

## Summary

El Módulo 3 genera la **Liquidación** de un lote avícola: toma el resultado final de sacrificio (Módulo 2), el precio por kilogramo que ingresa el Administrador Financiero y los costos del ciclo (Módulo 1 y Módulo 2), y calcula Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta. La Liquidación es inmutable, única `ACTIVA` por lote y solo se corrige por anulación. Alrededor de ella hay cuatro casos de uso de sincronización (CU07–CU10) que mantienen una copia local de los datos de M1 y M2, y cuatro de consulta y operación (CU01, CU04, CU05, CU06).

Este plan define **cómo** se construye en el estado actual del repositorio: una **librería de dominio en Java 17** con Maven y JUnit 5, **sin framework web, sin base de datos y sin dependencias nuevas**. Las decisiones que lo sostienen:

- **Dominio puro y probable con `mvn test`**: todas las reglas de negocio y las fórmulas viven en clases Java sin infraestructura, de modo que cada acceptance scenario de los specs se ejecuta tal cual está escrito con los valores del Caso Dorado.
- **Puertos hacia M1 y M2**: los accesos externos son interfaces (`Module1Gateway`, `Module2Gateway`). En esta etapa solo existen dobles en memoria para pruebas; los adaptadores reales se escriben cuando se cierren D-02 y D-03.
- **Copia local sincronizada en memoria**: repositorios en memoria detrás de interfaces, para que la persistencia definitiva (si el equipo la decide) no toque el dominio.
- **Un punto de código por decisión abierta**: cada `[NEEDS CLARIFICATION – D-0X]` queda aislado en una única clase o método con un supuesto explícito (sección *Open Questions*), sin resolverlo.
- **Cuatro paquetes**: `shared`, `sync`, `domain`, `application`, según `AGENTS.md` §9, refinados en *Project Structure*.

## Technical Context

**Language/Version**: Java 17 (`maven.compiler.release` 17), Maven con `maven-surefire-plugin` 3.2.5, tal como está en el `pom.xml`.
**Primary Dependencies**: ninguna en `main`. Solo `org.junit.jupiter:junit-jupiter` 5.10.2 en `test`. No se agregan dependencias (`AGENTS.md` §3); la exportación a Excel y cualquier adaptador HTTP quedan fuera de este plan por esa razón.
**Storage**: memoria. Repositorios definidos como interfaces con una implementación `InMemory…` en `main`. No hay base de datos ni migraciones; la persistencia definitiva es una decisión pendiente del equipo (alineación con M1/M2, que usan PostgreSQL).
**Testing**: JUnit 5 (Jupiter) ejecutado por surefire con `mvn test`. Sin Mockito: los dobles de M1/M2 son clases escritas a mano en `src/test`.
**Target Platform**: JVM 17, empaquetado como `jar` de librería. Sin interfaz de usuario en este plan; los specs describen pantallas (P1–P5, ver `PENDIENTES.md`) que consumirán los servicios de `application`.
**Project Type**: single. Un único módulo Maven.
**Performance Goals**: no aplican en esta etapa. SC-002 (30 s), CU01.SC-001 (3 s), CU03.SC-002 (5 s), CU04.SC-003 (2 s), CU05.SC-002 (10 s) y CU06.SC-001 (3 s) se verifican cuando exista interfaz y persistencia.
**Constraints**:
- Todo valor monetario y todo kilogramo es `BigDecimal`; nunca `double`. Totales en COP con `setScale(0, HALF_UP)`; porcentajes con `setScale(2, HALF_UP)` (RT-01, RT-02).
- Ninguna operación funcional consulta en vivo a M1 o M2 (Modelo de Integración §3, SC-007). Solo `sync` conoce los gateways.
- M3 no escribe en M1 ni M2, salvo el aviso de utilización (CU08.FR-006).
- "Ahora" se obtiene de un `java.time.Clock` inyectado; en pruebas es un reloj fijo.
- Código en inglés sin comentarios; nombres de entidades del glosario y de atributos de las Key Entities tal cual están en los specs (`Liquidacion`, `pesoTotalKg`, `precioKgCop`).
- Sin pruebas de UI, concurrencia (SC-004, CU04 edge "Concurrencia"), tiempos ni exportación real a Excel (CA-G06), por decisión de `AGENTS.md` §9.

**Scale/Scope**: una granja, decenas de galpones, un Administrador Financiero (identidad fija provista por el contexto externo, `spec.md` supuestos), dos integraciones de lectura (M1, M2), una notificación saliente (M2), 9 casos de uso, 54 acceptance scenarios.

## Project Structure

### Documentation (this feature)

```text
docs/specs/features/modulo3/
├── spec.md                      # índice normativo
├── plan/
│   └── plan-v1.md               # este archivo (plan general de los 9 CU, versión 1.0)
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
│   ├── RegistroSincronizacion.java         # bitácora (Key Entity CU07–CU10)
│   ├── ResultadoSincronizacion.java        # EXITOSA | FALLIDA
│   ├── RegistroSincronizacionRepository.java
│   ├── InMemoryRegistroSincronizacionRepository.java
│   ├── SyncStatus.java                     # última sincronización exitosa por fuente + fallo reciente (RT-09)
│   ├── SyncStatusQuery.java
│   ├── ModuleUnavailableException.java
│   ├── SyncScheduler.java                  # ScheduledExecutorService con intervalo configurable
│   ├── m1/
│   │   ├── Module1Gateway.java             # puerto de lectura hacia M1 (CU07)
│   │   ├── RemoteGalpon.java · RemoteLote.java · RemoteAlertaVaciado.java
│   │   ├── EstadoGalpon.java               # 6 estados, grafía de M1 (D-07)
│   │   ├── Galpon.java · Lote.java · AlertaVaciadoSanitario.java
│   │   ├── CopiaLocalGalponLote.java       # repositorio de la copia local de M1 (Key Entity)
│   │   ├── InMemoryCopiaLocalGalponLote.java
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
│   ├── Liquidacion.java                    # único Documento Financiero
│   ├── EstadoLiquidacion.java              # ACTIVA | ANULADA
│   ├── PartidaCostoLote.java · CategoriaCosto.java
│   ├── SnapshotDatosOrigen.java · FuenteOrigen.java
│   ├── RegistroAnulacion.java
│   ├── LiquidacionCalculator.java          # fórmulas FR-001..FR-004 de CU03
│   ├── CostValuation.java                  # subtotal de una partida (punto único de D-05)
│   ├── LiquidacionEligibility.java · Eligibility.java   # precondiciones compartidas CU01/CU03
│   ├── LiquidacionRepository.java
│   └── InMemoryLiquidacionRepository.java  # garantiza CA-G04 en la escritura
└── application/
    ├── ListGalponesService.java            # CU01  → GalponRow, GalponListResult
    ├── GenerateLiquidacionService.java     # CU03  → GenerateLiquidacionCommand
    ├── AnnulLiquidacionService.java        # CU04  → AnulacionPendiente
    ├── DesgloseQueryService.java           # CU05  → DesgloseView, DesgloseLine, DesgloseExporter (puerto)
    ├── LiquidacionHistoryService.java      # CU06  → HistoryFilter, HistoryRow, HistoryResult, EmptyReason
    ├── MissingSyncedDataException.java · LoteNotLiquidableException.java
    ├── ActiveLiquidacionExistsException.java · InvalidPrecioKgException.java
    ├── PendingValuationException.java · NoCostItemsException.java
    ├── InvalidAnulacionMotivoException.java · LiquidacionAlreadyAnuladaException.java
    ├── ExportNotAllowedException.java · ExportFailedException.java
    └── InvalidDateRangeException.java

src/test/java/co/edu/unimagdalena/avicontrol/
├── support/
│   ├── FakeModule1Gateway.java             # modos: COMPLETO | DATOS_FALTANTES | AUSENTE (CA-G07)
│   ├── FakeModule2Gateway.java             # ídem + registro de avisos recibidos y modo de fallo del aviso
│   ├── FakeDesgloseExporter.java           # captura lo exportado; modo de fallo
│   ├── CasoDorado.java                     # fixture con los valores exactos de los specs
│   └── TestContext.java                    # cablea repositorios en memoria, reloj fijo y servicios
├── shared/RoundingTest.java
├── domain/LiquidacionCalculatorTest.java
├── domain/InMemoryLiquidacionRepositoryTest.java
├── sync/m1/CU07ConsultarGalponLoteM1Test.java
├── sync/m2/CU08ConsultarResultadoSacrificioM2Test.java
├── sync/m2/CU09ConsultarAlimentoRequeridoM2Test.java
├── sync/m2/CU10ConsultarConsumoMedicamentoM2Test.java
├── application/CU01ConsultarListaGalponesTest.java
├── application/CU03GenerarLiquidacionTest.java
├── application/CU04AnularLiquidacionTest.java
├── application/CU05ConsultarDesgloseTest.java
└── application/CU06ConsultarHistorialTest.java
```

**Structure Decision**: se mantienen los cuatro paquetes de `AGENTS.md` §9 y se refina `sync` en `sync.m1` y `sync.m2` porque cada subpaquete es la copia local de una fuente y tiene su propio gateway. Las implementaciones `InMemory…` viven en `main`, no en `test`, porque hoy son el almacenamiento real de la librería; los dobles de M1/M2 (`Fake…Gateway`) sí viven en `test`, porque simulan sistemas externos. No existen las clases `MatrizVentas`, `Reporte`, `Venta` ni `Desglose` como entidades: la Matriz de Venta Final son los atributos de `Liquidacion` (CU03.FR-013) y el Desglose es el record de solo lectura `DesgloseView` que devuelve un servicio y nunca se persiste (CU05.FR-007).

## Architecture & Design Decisions

### Justificación de las decisiones

| Decisión | Justificación | Alternativa descartada y por qué |
|---|---|---|
| **Librería de dominio sin framework** | Es lo que hay en el `pom.xml` y lo que `AGENTS.md` §3 fija. Permite ejecutar los 54 acceptance scenarios con `mvn test` en segundos y sin infraestructura. El dominio no cambia cuando se agregue UI o persistencia. | Spring Boot + PostgreSQL como M1/M2: es una decisión de alineación de equipo, no del plan; se puede tomar después sin reescribir `domain` ni `application`. |
| **Puertos (`Module1Gateway`, `Module2Gateway`) en `sync`** | El Modelo de Integración exige que solo CU07–CU10 conozcan a M1/M2. Una interfaz por módulo externo deja el contrato pendiente (D-02, D-03) en un solo lugar y permite los tres modos de CA-G07 con un doble en memoria. | Que los servicios de aplicación consulten M1/M2: viola SC-007 y "ninguna operación funcional hace consultas en vivo". |
| **Copia local en repositorios en memoria detrás de interfaces** | Cumple "copia local sincronizada con fecha y hora" sin base de datos. Las interfaces (`CopiaLocalGalponLote`, `…Repository`) son el único punto que cambiaría con una persistencia real. | Estructuras estáticas o `Map` compartidos: no serían reemplazables ni aislables en pruebas. |
| **`Liquidacion` como clase con campos `final` + `anular()`** | RT-03 (inmutabilidad) y CU04.FR-006 (solo cambia el estado) se cumplen por construcción: el único método que muta es `anular(RegistroAnulacion)`, que cambia `estado` y adjunta el registro en una sola operación (CU04.FR-008, atomicidad sin transacciones). | Setters + validación en servicio: la inmutabilidad dependería de disciplina, no del tipo. |
| **`LiquidacionCalculator` con funciones puras** | Las cuatro fórmulas de CU03 (FR-001..FR-004) se prueban con valores exactos sin montar nada. SC-001 (100 % de coincidencia con cálculo manual) se verifica en un test unitario. | Cálculo dentro del servicio: mezcla orquestación con aritmética y dificulta la auditoría. |
| **`LiquidacionEligibility` compartido por CU01 y CU03** | CU01.FR-004/FR-005 y CU03.FR-005 enuncian la misma precondición (estado `Vaciado sanitario` + alerta con UUID del lote + resultado válido o siniestro total + sin `ACTIVA`). Un solo lugar evita que la lista habilite lo que la generación rechaza. | Duplicar la regla en cada servicio: dos fuentes de verdad. |
| **`Rounding` en `shared` como único lugar de redondeo** | RT-01 y RT-02 se aplican en un método cada una. Ningún otro código llama a `setScale`. | Redondear en cada fórmula: fácil de olvidar y de hacer distinto. |
| **Subtotal por partida redondeado a entero, total = suma de subtotales** | CU05.FR-002 exige que la suma de subtotales del desglose sea idéntica al peso a `costosOperativosCop`. Si el total se redondeara aparte, podría diferir en 1 COP. | Sumar exacto y redondear el total: descuadra el desglose. |
| **Aviso de utilización persistido como `AvisoUtilizacionResultado` + `UsageNoticeDispatcher`** | CU08.FR-007 exige asíncrono, idempotente y con reintento sin revertir la Liquidación. El aviso nace `PENDIENTE` al persistir la Liquidación; el dispatcher intenta entregarlo y, si falla, lo deja `PENDIENTE` para el siguiente ciclo de sincronización. | Llamar al gateway dentro de la generación y fallar si no responde: viola SC-004 de CU08. |
| **Anulación en dos pasos (`prepare` / `confirm`)** | CU04.FR-004 y AS-05 describen una confirmación explícita y una cancelación sin efectos. Sin UI, el paso `prepare` valida el motivo y devuelve el resumen (lote afectado, consecuencias); `confirm` persiste. Cancelar es no llamar a `confirm`, y así AS-05 es probable. | Un solo método `anular(id, motivo)`: AS-05 quedaría sin prueba. |
| **`DesgloseExporter` como puerto sin implementación** | CU05.FR-003 exige `.xlsx`, pero `AGENTS.md` §3 prohíbe añadir POI sin acuerdo. El puerto deja lista la costura y permite probar FR-004 (no exportar `ANULADA`) y FR-005 (fallo sin perder la vista) con un doble. | Implementar el Excel ahora: requiere dependencia. Omitir el puerto: CU05 AS-02/AS-04 no tendrían dónde engancharse. |

### Capas y responsabilidades

```text
Administrador Financiero (UI futura)            Proceso automático (SyncScheduler)
            │                                              │
   ┌────────▼──────────────────────────────────────────────▼────────┐
   │ application   servicios por CU, comandos, vistas de solo lectura│
   ├────────────────────────────────────────────────────────────────┤
   │ domain        Liquidacion, partidas, snapshot, fórmulas, reglas │
   ├────────────────────────────────────────────────────────────────┤
   │ sync          copia local M1/M2, bitácora, gateways, servicios  │
   │               de sincronización, dispatcher del aviso           │
   ├────────────────────────────────────────────────────────────────┤
   │ shared        Rounding                                          │
   └────────────────────────────────────────────────────────────────┘
                                   │ Module1Gateway · Module2Gateway (puertos)
                                   ▼
                         Módulo 1 · Módulo 2 (fuera del jar)
```

| Paquete | Responsabilidad | Puede depender de | No debe |
|---|---|---|---|
| `application` | Orquestar un caso de uso: leer la copia local, aplicar reglas de `domain`, persistir, devolver vistas o lanzar excepciones con mensajes del spec. | `domain`, `sync` (solo repositorios, `SyncStatusQuery` y `UsageNoticeDispatcher`), `shared` | Conocer `Module1Gateway` ni `Module2Gateway`; calcular fórmulas; redondear. |
| `domain` | Entidades del glosario, fórmulas, valorización, elegibilidad, repositorio de Liquidaciones. | `sync.m1`/`sync.m2` (entidades de la copia local, como datos de entrada), `shared` | Conocer gateways ni bitácora. |
| `sync` | Consultar M1/M2 por los puertos, validar e incorporar a la copia local, registrar la bitácora, entregar el aviso de utilización. | `shared` | Calcular subtotales, totales o utilidad (CU09.FR-003, CU10.FR-005); interpretar valores (CU07.FR-009). |
| `shared` | Redondeo COP y porcentajes. | Nada del proyecto | |

**Reglas de la arquitectura** (se revisan en code review; no hay ArchUnit):

1. Solo `sync` importa `Module1Gateway` y `Module2Gateway`.
2. Solo `shared.Rounding` llama a `BigDecimal.setScale`.
3. Solo `domain.LiquidacionCalculator` y `domain.CostValuation` multiplican cantidades por precios.
4. `Liquidacion` no tiene setters; su único mutador es `anular`.
5. Ningún servicio de `application` crea partidas con valor cero ni infiere datos ausentes.

### Diseño transversal

- **Reloj.** Todos los servicios reciben `java.time.Clock` por constructor. Fechas y horas son `LocalDateTime` en la zona del reloj; en pruebas, `Clock.fixed(...)`.
- **Usuario responsable.** Se recibe como `String usuarioResponsable` en `GenerateLiquidacionCommand` y en `AnnulLiquidacionService.confirm`. No hay autenticación (supuesto transversal del `spec.md`).
- **Bitácora y estado de sincronización (RT-09, CU07.FR-005).** Cada ejecución de un servicio de sincronización crea un `RegistroSincronizacion` (`fuente`, `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados`, `incidencias`). `SyncStatusQuery.current()` devuelve `SyncStatus` con la última sincronización exitosa por fuente y si la última ejecución falló; `ListGalponesService`, `DesgloseQueryService` y `LiquidacionHistoryService` lo incluyen en sus resultados para que toda pantalla pueda mostrarlo.
- **Incidencias.** Los specs piden "registrar como incidencia" datos incompletos, estados desconocidos, resultados inválidos o duplicados (CU07.FR-010/FR-011, CU08.FR-003/FR-010). Se modelan como `List<String> incidencias` dentro de `RegistroSincronizacion` (mensajes en español con el identificador del dato rechazado). No se crea una entidad nueva.
- **Idempotencia de la sincronización (CU07.FR-007, CU08.FR-009, CU09.FR-008, CU10.FR-009).** Cada repositorio de copia local hace `upsert` por el identificador de origen (`idGalpon`, `idLote`, `idAlerta`, `idResultado`, `idPartidaOrigen`, `idConsumoOrigen`) y actualiza `fechaHoraSincronizacion`. Obtener el mismo dato dos veces nunca crea un segundo registro.
- **Fallo de un módulo (CU07.FR-006, CU08.FR-008, CU09.FR-007, CU10.FR-008).** El gateway lanza `ModuleUnavailableException`; el servicio de sincronización captura, escribe un `RegistroSincronizacion` `FALLIDA` con `descripcionError` y **no toca** la copia local. El siguiente ciclo del `SyncScheduler` reintenta.
- **Datos parciales (CU07 edge "Datos parciales", CA-G07).** Los records `Remote…` admiten campos nulos. El servicio de sincronización valida cada registro: si es inválido (población inicial ausente, costo no positivo, alerta sin `idLote`, estado fuera de catálogo) lo rechaza con incidencia y **no sobrescribe** el registro válido previo; los válidos del mismo lote sí se incorporan.
- **Precondición de Liquidación (`LiquidacionEligibility.evaluate(idLote)`).** Devuelve `Eligibility(liquidable, siniestroTotal, motivoBloqueo, existeLiquidacion)` aplicando, en orden: (1) existe `Lote` en la copia local, si no → "no hay datos sincronizados disponibles para ese lote"; (2) el galpón está en `VACIADO_SANITARIO` y existe `AlertaVaciadoSanitario` con ese `idLote`, si no → mensaje de CU03 AS-07; (3) no existe `Liquidacion` `ACTIVA` para el lote (CA-G04), si existe → mensaje de CU03 AS-08; (4) si `poblacionActual == 0` → `siniestroTotal = true`, liquidable sin resultado; si no, debe existir `ResultadoFinalSacrificio` válido, si no → mensaje de CU03 AS-02.
- **Generación (`GenerateLiquidacionService.generate(command)`).** Tras la elegibilidad: (5) valida `precioKgCop > 0` salvo siniestro total (FR-011); (6) reúne partidas: alimento (`PartidaAlimentoLote` del lote), medicina (un `TramoRecepcionConsumo` por línea) y población (`Lote.costoTotalCop`); si no hay partidas de alimento → `NoCostItemsException` (CU03 AS-05, CU09 AS-03); si alguna está `PENDIENTE_PRECIO` → `PendingValuationException` con la lista (FR-006); (7) valoriza cada partida con `CostValuation`, calcula con `LiquidacionCalculator`, arma `SnapshotDatosOrigen` (uno por fuente, con la fecha y hora de la última sincronización exitosa de esa fuente y las referencias usadas); (8) persiste la `Liquidacion` `ACTIVA` con fecha, hora y usuario; (9) si consumió resultado, lo marca `UTILIZADO` y crea el `AvisoUtilizacionResultado` `PENDIENTE`, que `UsageNoticeDispatcher.dispatchPending()` intenta entregar de inmediato (FR-010, CU08.FR-006). Un fallo del aviso no revierte nada (CU08 AS-05).
- **Partida de población.** `PartidaCostoLote` con `categoria = POBLACION`, `concepto = "Costo total del lote"`, `cantidad = 1`, `unidadMedida = "lote"`, `precioUnitarioCop = subtotalCop = costoTotalCop`, `fuenteOrigen = MODULO_1`, `referenciaOrigen = idLote`. Así el desglose de CU05 la muestra como una fila sin inventar un precio por ave.
- **Matriz de Venta Final.** Son los atributos de `Liquidacion`: `ventaBrutaCop`, `mortalidadAves`, `porcentajeMortalidad`, `costosOperativosCop`, `utilidadNetaCop`, `pollosVendidos`, `pesoTotalKg`, `pesoPromedioKg`, `precioKgCop`. No hay clase aparte (glosario).
- **Desglose (CU05).** `DesgloseQueryService.view(idLiquidacion)` construye `DesgloseView` en el momento: sección de ingresos (datos de venta; Venta Bruta 0 y aviso "El lote no registró venta" en siniestro total), líneas por categoría `ALIMENTO`, `MEDICINA`, `POBLACION` con `concepto`, `cantidad`, `unidadMedida`, `precioUnitarioCop`, `subtotalCop`, `fuenteOrigen`, `referenciaOrigen`, el total de costos, el estado y, si está `ANULADA`, el `RegistroAnulacion`. `export(idLiquidacion, usuario)` rechaza `ANULADA` con `ExportNotAllowedException` y envuelve cualquier fallo del `DesgloseExporter` en `ExportFailedException` sin alterar la vista.
- **Historial (CU06).** `LiquidacionHistoryService.query(HistoryFilter)` valida `desde <= hasta` (`InvalidDateRangeException`), filtra por `idGalpon` opcional y por rango sobre `fechaHoraGeneracion`, ordena descendente y devuelve `HistoryResult(rows, emptyReason)` con `EmptyReason` `NONE | NO_LIQUIDACIONES | NO_MATCHES` para diferenciar los dos estados vacíos de AS-04. La paginación (edge "Rendimiento con alto volumen") se deja para la versión con persistencia.
- **Lista de galpones (CU01).** `ListGalponesService.list()` devuelve `GalponListResult(rows, syncStatus, emptyMessage)`. Cada `GalponRow` trae los campos de FR-001 y las acciones de la fila como banderas (`generarLiquidacionHabilitada`, `consultarDesgloseHabilitada`) más `motivoBloqueo`, calculadas con `LiquidacionEligibility` y `LiquidacionRepository.findByLote`. Las acciones deshabilitadas se representan con la bandera en `false`, nunca omitiendo el campo (FR-005: deshabilitadas, no ocultas).

## Data Model

No hay esquema de base de datos. Las estructuras son clases Java; los nombres de atributos son los de las Key Entities de cada spec.

### `domain`

| Clase | Atributos | Notas |
|---|---|---|
| `Liquidacion` | `idLiquidacion: UUID`, `idLote: UUID`, `idGalpon: UUID`, `idResultadoSacrificio: UUID` (nulo en siniestro total), `pollosVendidos: int`, `pesoTotalKg: BigDecimal`, `pesoPromedioKg: BigDecimal` (2 decimales, presentación), `precioKgCop: BigDecimal` (nulo en siniestro total), `ventaBrutaCop: BigDecimal`, `mortalidadAves: int`, `porcentajeMortalidad: BigDecimal`, `costosOperativosCop: BigDecimal`, `utilidadNetaCop: BigDecimal`, `partidas: List<PartidaCostoLote>`, `snapshots: List<SnapshotDatosOrigen>`, `estado: EstadoLiquidacion`, `fechaHoraGeneracion: LocalDateTime`, `usuarioResponsable: String`, `registroAnulacion: RegistroAnulacion` (nulo si `ACTIVA`) | Todos los campos `final` salvo `estado` y `registroAnulacion`; `anular(RegistroAnulacion)` es el único mutador y lanza si ya está `ANULADA`. `idLiquidacion` lo genera M3 (`UUID.randomUUID()`). |
| `EstadoLiquidacion` | `ACTIVA`, `ANULADA` | |
| `PartidaCostoLote` | `idPartida: UUID`, `idLote`, `categoria: CategoriaCosto`, `concepto`, `cantidad: BigDecimal`, `unidadMedida`, `precioUnitarioCop: BigDecimal`, `impuestoCop: BigDecimal`, `subtotalCop: BigDecimal` (entero), `fuenteOrigen: FuenteOrigen`, `referenciaOrigen: String` | Record. `impuestoCop` se conserva aparte y no entra en `subtotalCop` (D-05). |
| `CategoriaCosto` | `ALIMENTO`, `MEDICINA`, `POBLACION` | |
| `SnapshotDatosOrigen` | `fuente: FuenteOrigen`, `fechaHoraSincronizacion: LocalDateTime`, `referencias: List<UUID>` | Record. `idLiquidacion` e `idLote` están en la `Liquidacion` que lo contiene. |
| `FuenteOrigen` | `MODULO_1`, `MODULO_2` | |
| `RegistroAnulacion` | `idAnulacion: UUID`, `idLiquidacion`, `motivo`, `fechaHoraAnulacion`, `usuarioResponsable` | Record. |
| `Eligibility` | `liquidable: boolean`, `siniestroTotal: boolean`, `motivoBloqueo: String`, `liquidacionExistente: Optional<Liquidacion>` | Record de solo lectura. |

### `sync.m1` (copia local de Módulo 1)

| Clase | Atributos | Notas |
|---|---|---|
| `Galpon` | `idGalpon: UUID`, `nombre`, `aforoMaximo: long`, `estado: EstadoGalpon`, `fechaHoraSincronizacion` | Record. |
| `EstadoGalpon` | `DISPONIBLE("Disponible")`, `PRODUCTIVO("Productivo")`, `EN_COSECHA("En cosecha")`, `VACIADO_SANITARIO("Vaciado sanitario")`, `MANTENIMIENTO("Mantenimiento")`, `AISLAMIENTO("Aislamiento")` | `fromLabel(String)` acepta la grafía de M1 sin distinguir mayúsculas; desconocido → `Optional.empty()` (D-07, CU07.FR-011). |
| `Lote` | `idLote: UUID`, `idGalpon`, `nombre`, `fechaIngreso: LocalDate`, `poblacionInicial: long`, `poblacionActual: long`, `costoTotalCop: BigDecimal` (entero), `fechaHoraSincronizacion` | Record. Se conserva aunque M1 desvincule el lote (CU07.FR-004). |
| `AlertaVaciadoSanitario` | `idAlerta: UUID`, `idGalpon`, `idLote`, `fechaHoraEvento`, `fechaHoraSincronizacion` | Record. |
| `CopiaLocalGalponLote` | interfaz: `upsertGalpon`, `upsertLote`, `upsertAlerta`, `findGalpon`, `findLote`, `findLoteActivoByGalpon`, `findAlertaByLote`, `allGalpones`, `isEmpty` | Key Entity de CU07/CU01 modelada como repositorio. |

### `sync.m2` (copia local de Módulo 2)

| Clase | Atributos | Notas |
|---|---|---|
| `ResultadoFinalSacrificio` | `idResultado: UUID`, `idLote`, `idGalpon`, `cantidadFinalPollos: int`, `pesoTotalKg: BigDecimal`, `fechaRegistro: LocalDate`, `fechaHoraSincronizacion`, `estadoUtilizacionLocal: EstadoUtilizacionLocal` | Record con `withEstadoUtilizacion(...)`. Único por lote (CU08.FR-010). |
| `EstadoUtilizacionLocal` | `NO_UTILIZADO`, `UTILIZADO` | |
| `AvisoUtilizacionResultado` | `idAviso: UUID`, `idResultado`, `idLiquidacion`, `fechaHoraEmision`, `estadoEntrega: EstadoEntrega`, `intentos: int` | Record. |
| `EstadoEntrega` | `PENDIENTE`, `ENTREGADO`, `FALLIDO` | |
| `PartidaAlimentoLote` | `idPartidaOrigen: UUID`, `idLote`, `idGalpon`, `tipoAlimento`, `cantidadKg: BigDecimal`, `precioUnitarioKgCop: BigDecimal` (nulo si pendiente), `valorImpuestoCop: BigDecimal`, `estadoValorizacion: EstadoValorizacion`, `referenciaOrigen`, `fechaHoraSincronizacion` | Record. |
| `EstadoValorizacion` | `VALORIZADA`, `PENDIENTE_PRECIO` | Compartido por alimento y tramos de medicamento. |
| `ConsumoMedicamentoLote` | `idConsumoOrigen: UUID`, `idLote`, `idGalpon`, `medicamento`, `fechaConsumo: LocalDate`, `cantidadUnidadBase: BigDecimal`, `unidadBase`, `tramos: List<TramoRecepcionConsumo>`, `fechaHoraSincronizacion` | Record. |
| `TramoRecepcionConsumo` | `idTramo: UUID`, `idConsumoOrigen`, `referenciaRecepcion`, `cantidadUnidadBase: BigDecimal`, `precioUnitarioBaseCop: BigDecimal` (nulo si pendiente), `valorImpuestoCop`, `estadoValorizacion` | Record. |

### `sync` (común)

| Clase | Atributos | Notas |
|---|---|---|
| `RegistroSincronizacion` | `idSincronizacion: UUID`, `fuente: FuenteSincronizacion`, `fechaHoraInicio`, `fechaHoraFin`, `resultado: ResultadoSincronizacion`, `descripcionError`, `cantidadRegistrosActualizados: int`, `incidencias: List<String>` | Record. Compartido por CU07–CU10. |
| `SyncStatus` | `ultimaExitosaModulo1: Optional<LocalDateTime>`, `ultimaExitosaModulo2: Optional<LocalDateTime>`, `modulo1EnFallo: boolean`, `modulo2EnFallo: boolean` | Record para RT-09. |

## API Contracts

No hay HTTP. Los contratos son las firmas de los **puertos** (lo que M3 necesita de M1 y M2, insumo para cerrar D-02 y D-03) y de los **servicios de aplicación** (lo que consumirán las pantallas P1–P5).

### Puertos hacia los módulos de origen (`sync`)

```java
public interface Module1Gateway {
    List<RemoteGalpon> fetchGalpones() throws ModuleUnavailableException;
    List<RemoteLote> fetchLotesActivos() throws ModuleUnavailableException;
    List<RemoteAlertaVaciado> fetchAlertasVaciadoSanitario() throws ModuleUnavailableException;
}

public interface Module2Gateway {
    List<RemoteResultadoSacrificio> fetchResultadosSacrificio() throws ModuleUnavailableException;
    List<RemotePartidaAlimento> fetchPartidasAlimento(UUID idLote) throws ModuleUnavailableException;
    List<RemoteConsumoMedicamento> fetchConsumosMedicamento(UUID idLote) throws ModuleUnavailableException;
    void notifyResultUsage(UUID idResultado, UUID idLiquidacion) throws ModuleUnavailableException;
}
```

Los records `Remote…` replican los campos de FR-002/FR-003/FR-004 de cada spec con tipos nulos permitidos (`Integer`, `Long`, `BigDecimal`, `String`), para que el doble pueda simular datos faltantes. `EstadoGalpon` llega como `String` con la grafía de M1.

### Servicios de sincronización (`sync`)

| Servicio | Método | CU |
|---|---|---|
| `Module1SyncService` | `RegistroSincronizacion synchronize()` | CU07 |
| `SacrificioSyncService` | `RegistroSincronizacion synchronize()` | CU08 |
| `AlimentoSyncService` | `RegistroSincronizacion synchronize()` (recorre los lotes de la copia local) | CU09 |
| `MedicamentoSyncService` | `RegistroSincronizacion synchronize()` (ídem) | CU10 |
| `UsageNoticeDispatcher` | `void dispatchPending()` | CU08 |
| `SyncStatusQuery` | `SyncStatus current()` | RT-09 |
| `SyncScheduler` | `SyncScheduler(Duration interval, Runnable... jobs)`, `start()`, `stop()` | Modelo de Integración §1 |

### Servicios de aplicación (`application`)

| Servicio | Firma | Devuelve / lanza | CU |
|---|---|---|---|
| `ListGalponesService` | `GalponListResult list()` | `rows: List<GalponRow>`, `syncStatus`, `emptyMessage` ("No hay galpones disponibles para consultar" o nulo) | CU01 |
| `GenerateLiquidacionService` | `Liquidacion generate(GenerateLiquidacionCommand cmd)` con `cmd = (idLote, precioKgCop nullable, usuarioResponsable)` | `Liquidacion` `ACTIVA`; lanza `MissingSyncedDataException`, `LoteNotLiquidableException`, `ActiveLiquidacionExistsException` (con la existente), `InvalidPrecioKgException`, `NoCostItemsException`, `PendingValuationException` (con la lista) | CU03 |
| `AnnulLiquidacionService` | `AnulacionPendiente prepare(UUID idLiquidacion, String motivo)`; `Liquidacion confirm(AnulacionPendiente pendiente, String usuarioResponsable)` | `AnulacionPendiente(idLiquidacion, idLote, motivo, consecuencias)`; lanza `LiquidacionAlreadyAnuladaException` (con el `RegistroAnulacion`), `InvalidAnulacionMotivoException` | CU04 |
| `DesgloseQueryService` | `DesgloseView view(UUID idLiquidacion)`; `void export(UUID idLiquidacion, String usuario)` | `DesgloseView`; lanza `ExportNotAllowedException`, `ExportFailedException` | CU05 |
| `LiquidacionHistoryService` | `HistoryResult query(HistoryFilter filter)` con `filter = (idGalpon nullable, desde nullable, hasta nullable)` | `HistoryResult(rows, emptyReason, syncStatus)`; lanza `InvalidDateRangeException` | CU06 |

Todos los mensajes de las excepciones son los textos literales de los acceptance scenarios (por ejemplo, "El lote no cuenta con resultado final de sacrificio sincronizado desde Módulo 2. No es posible liquidar").

## Testing Strategy

| Tipo | Qué cubre | Dónde |
|---|---|---|
| Acceptance (un `@Test` por scenario) | Los 54 acceptance scenarios de los 9 specs, con `@DisplayName("CUNN-AS-XX – <título>")` en el mismo orden que el spec y con los valores exactos del spec. Verifican CA-G02. | `test/.../CUNNXxxTest.java` |
| Unitarias de dominio | Fórmulas FR-001..FR-004 de CU03 con el Caso Dorado y el siniestro total; redondeos RT-01/RT-02 en casos límite (`.5`, cocientes no exactos); `CostValuation` con precios decimales. | `LiquidacionCalculatorTest`, `RoundingTest` |
| Invariantes | CA-G04 en el repositorio (`save` de una segunda `ACTIVA` para el mismo lote lanza) y en el servicio (CU03 AS-08). CA-G08 por construcción (`Liquidacion` sin setters). | `InMemoryLiquidacionRepositoryTest`, `CU03GenerarLiquidacionTest` |
| CA-G07 (tres modos) | `FakeModule1Gateway` y `FakeModule2Gateway` con modos `COMPLETO`, `DATOS_FALTANTES` y `AUSENTE` (lanza `ModuleUnavailableException`). Cada test de sincronización ejercita al menos dos modos. | `support/` |

Fuera de alcance en esta etapa (`AGENTS.md` §9): UI, concurrencia, tiempos, archivo `.xlsx` real. CU05 AS-02 se cubre en la parte que sí es verificable (el `DesgloseExporter` recibe exactamente la `DesgloseView` mostrada); la igualdad con el archivo (CA-G06) queda para cuando exista implementación.

### Fixture única: `CasoDorado`

Los specs fijan los totales; el plan fija el desglose que los produce, para que CU03 y CU05 usen los mismos datos. Debe validarse con el stakeholder para CA-G03.

| Dato | Valor | Origen |
|---|---|---|
| Población inicial / actual | 9.000 / 8.500 aves | CU03 AS-01 (M1) |
| Resultado final de sacrificio | 8.500 pollos, 23.800 kg | CU03 AS-01, CU08 AS-01 (M2) |
| Precio por kg | $4.500 | CU03 AS-01 |
| Costo total del lote (POBLACION) | $27.000.000 | plan |
| Alimento Pre-inicio | 4.000 kg × $2.500 = $10.000.000 | plan |
| Alimento Inicio | 8.000 kg × $2.400 = $19.200.000 | plan |
| Alimento Engorde | 12.000 kg × $2.300 = $27.600.000 | plan |
| Medicamento (un consumo, dos recepciones) | 400 ml × $1.800 = $720.000 y 300 ml × $1.600 = $480.000 | plan; precios distintos para CU10 AS-02 |
| **Costos Operativos** | **$85.000.000** | CU03 AS-01 |
| Resultado esperado | Venta Bruta $107.100.000 · Mortalidad 500 (5,56 %) · Utilidad Neta $22.100.000 · peso promedio 2,80 kg | CU03 AS-01 |

Siniestro total (CU03 AS-06): 9.000 iniciales, población actual 0, sin resultado ni precio, POBLACION $27.000.000 + Alimento 5.000 kg × $2.600 = $13.000.000 → Costos Operativos $40.000.000, Utilidad Neta −$40.000.000, mortalidad 9.000 (100,00 %).

## Tareas

Rutas relativas a `src/main/java/co/edu/unimagdalena/avicontrol/` (abreviado `main/`) y `src/test/java/co/edu/unimagdalena/avicontrol/` (abreviado `test/`). `[P]` = se puede hacer en paralelo con las demás `[P]` de su bloque porque toca archivos distintos. `[CUNN]` enlaza la tarea con su caso de uso. Cada CU se implementa en su propia rama `feature/cuNN-…` (`AGENTS.md` §10).

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Dejar la estructura de paquetes y el redondeo listos.

- [ ] T001 Crear los paquetes `shared`, `sync`, `sync/m1`, `sync/m2`, `domain`, `application` en `main/` y `support`, `shared`, `domain`, `sync/m1`, `sync/m2`, `application` en `test/`; eliminar los `.gitkeep`
- [ ] T002 [P] Crear `main/shared/Rounding.java` con `copTotal(BigDecimal)` (`setScale(0, HALF_UP)`) y `percentage(BigDecimal)` (`setScale(2, HALF_UP)`)
- [ ] T003 [P] Escribir `test/shared/RoundingTest.java`: `.5` hacia arriba, negativos, cocientes no exactos (500/9.000 × 100 → 5,56)
- [ ] T004 Verificar que `mvn test` compila y ejecuta con la estructura vacía

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Bitácora, puertos, excepción de indisponibilidad, planificador y soporte de pruebas que todos los CU usan.

**⚠️ CRITICAL**: Ningún CU empieza hasta terminar esta fase.

- [ ] T005 [P] Crear en `main/sync/` `FuenteSincronizacion`, `ResultadoSincronizacion`, `RegistroSincronizacion` (record con `incidencias`), `RegistroSincronizacionRepository` e `InMemoryRegistroSincronizacionRepository`
- [ ] T006 [P] Crear `main/sync/ModuleUnavailableException.java` (checked, con la fuente y la causa)
- [ ] T007 [P] Crear `main/sync/m1/Module1Gateway.java` y los records `RemoteGalpon`, `RemoteLote`, `RemoteAlertaVaciado` con campos nulos permitidos
- [ ] T008 [P] Crear `main/sync/m2/Module2Gateway.java` y los records `RemoteResultadoSacrificio`, `RemotePartidaAlimento`, `RemoteConsumoMedicamento`, `RemoteTramoRecepcion`
- [ ] T009 Crear `main/sync/SyncStatus.java` y `SyncStatusQuery.java` (última exitosa por fuente y bandera de fallo, a partir de la bitácora) (depende de T005)
- [ ] T010 [P] Crear `main/sync/SyncScheduler.java` sobre `ScheduledExecutorService` con `Duration` configurable; sin pruebas (concurrencia fuera de alcance)
- [ ] T011 [P] Crear en `test/support/` `FakeModule1Gateway` y `FakeModule2Gateway` con los tres modos de CA-G07, listas mutables de datos remotos y, en M2, la lista de avisos recibidos y un modo de fallo del aviso
- [ ] T012 [P] Crear `test/support/CasoDorado.java` con los valores de la tabla *Fixture única* (constructores para el caso completo, el siniestro total y variantes con partida sin precio) y `TestContext.java` que cablea reloj fijo, repositorios en memoria, gateways falsos y servicios

**Checkpoint**: la bitácora registra ejecuciones, los puertos existen y los dobles simulan los tres modos. Los CU pueden empezar.

---

## Phase 3: CU07 – Consultar Galpón y Lote al Módulo 1 (Priority: P1)

**Goal**: Mantener la copia local de galpones, lotes activos y alertas de vaciado sanitario con fecha y hora de sincronización, sin transformar valores, tolerando fallos de M1.

**Independent Test**: `Module1SyncService.synchronize()` contra `FakeModule1Gateway` en modo `COMPLETO` deja la copia local poblada; en modo `AUSENTE` la copia previa queda intacta y la bitácora registra `FALLIDA`.

**Cobertura de FR**: FR-001 → `SyncScheduler` + `synchronize()` · FR-002/FR-003/FR-004 → `Galpon`, `Lote`, `AlertaVaciadoSanitario` + `upsert` · FR-005 → `RegistroSincronizacion` + `SyncStatusQuery` · FR-006 → captura de `ModuleUnavailableException` · FR-007 → `upsert` por id · FR-008 → `Module1Gateway` solo tiene métodos `fetch` · FR-009 → los records copian el valor sin `setScale` ni conversión · FR-010/FR-011 → validación por registro con incidencia.

### Tests for CU07

- [ ] T013 [CU07] `test/sync/m1/CU07ConsultarGalponLoteM1Test.java` con seis `@Test`: `CU07-AS-01` sincronización inicial exitosa · `CU07-AS-02` incremental con cambio de población (inicial intacta) · `CU07-AS-03` alerta de vaciado con lote desvinculado que sigue en la copia · `CU07-AS-04` M1 no responde con copia previa (copia íntegra, bitácora `FALLIDA`) · `CU07-AS-05` M1 no responde sin copia previa (operar sobre el lote lanza `MissingSyncedDataException`, se verifica vía `LiquidacionEligibility` en la Fase 8; aquí se verifica `findLote` vacío y bitácora) · `CU07-AS-06` idempotencia (misma alerta dos veces, un registro, fecha actualizada). Más un test de FR-010/FR-011: lote sin población inicial, costo no positivo, alerta sin lote y estado desconocido generan incidencia sin sobrescribir.

### Implementation for CU07

- [ ] T014 [P] [CU07] Crear `main/sync/m1/EstadoGalpon.java` con etiqueta de M1 y `fromLabel`
- [ ] T015 [P] [CU07] Crear los records `Galpon`, `Lote`, `AlertaVaciadoSanitario` en `main/sync/m1/`
- [ ] T016 [CU07] Crear `CopiaLocalGalponLote` (interfaz) e `InMemoryCopiaLocalGalponLote` con `upsert` por id (depende de T015)
- [ ] T017 [CU07] Implementar `Module1SyncService.synchronize()`: consulta los tres `fetch`, valida cada registro, incorpora los válidos, acumula incidencias, escribe la bitácora; ante `ModuleUnavailableException` registra `FALLIDA` sin tocar la copia (depende de T016)

**Checkpoint**: CU01 y CU03 tienen de dónde leer galpones, lotes y alertas.

---

## Phase 4: CU08 – Consultar Resultado Final de Sacrificio al Módulo 2 (Priority: P1)

**Goal**: Sincronizar el resultado final de sacrificio como unidad indivisible, validarlo, admitir correcciones hasta que se use y emitir el aviso de utilización con reintento.

**Independent Test**: `SacrificioSyncService.synchronize()` con un resultado de 8.500 pollos y 23.800 kg lo deja en la copia local `NO_UTILIZADO`; `UsageNoticeDispatcher.dispatchPending()` entrega los avisos `PENDIENTE` y, si el gateway falla, los conserva `PENDIENTE` con `intentos` incrementado.

**Cobertura de FR**: FR-001 → `synchronize()` · FR-002 → `ResultadoFinalSacrificio` · FR-003 → validación `cantidadFinalPollos > 0`, `pesoTotalKg > 0` con incidencia · FR-004 → `upsert` solo si `NO_UTILIZADO` · FR-005 → `Liquidacion` inmutable; la sincronización nunca toca `LiquidacionRepository` · FR-006 → `GenerateLiquidacionService` marca `UTILIZADO` y crea el aviso (Fase 8) · FR-007 → `UsageNoticeDispatcher` · FR-008 → captura de `ModuleUnavailableException` · FR-009 → `upsert` por `idResultado` · FR-010 → segundo resultado para el mismo lote → incidencia, se conserva el previo.

### Tests for CU08

- [ ] T018 [CU08] `test/sync/m2/CU08ConsultarResultadoSacrificioM2Test.java`: `CU08-AS-01` sincronización exitosa · `CU08-AS-02` rechazo de resultado inválido (cero, negativo, ausente) · `CU08-AS-03` corrección antes de ser utilizado · `CU08-AS-04` aviso de utilización tras generar la Liquidación (marca `UTILIZADO`, el gateway falso recibe `idResultado`) · `CU08-AS-05` fallo del aviso (Liquidación conservada, aviso `PENDIENTE`, reintento en `dispatchPending()`) · `CU08-AS-06` M2 no responde con copia previa (la generación usa la copia). AS-04 a AS-06 usan `GenerateLiquidacionService`, por lo que este test se completa en la Fase 8; se deja escrito y marcado `@Disabled("Fase 8")` hasta entonces. Más un test de FR-010 (dos resultados para el mismo lote).

### Implementation for CU08

- [ ] T019 [P] [CU08] Crear `ResultadoFinalSacrificio`, `EstadoUtilizacionLocal`, `AvisoUtilizacionResultado`, `EstadoEntrega` en `main/sync/m2/`
- [ ] T020 [CU08] Crear `ResultadoFinalSacrificioRepository` y `AvisoUtilizacionResultadoRepository` con sus `InMemory…` (depende de T019)
- [ ] T021 [CU08] Implementar `SacrificioSyncService.synchronize()` con validación FR-003, corrección FR-004 y duplicado FR-010 (depende de T020)
- [ ] T022 [CU08] Implementar `UsageNoticeDispatcher.dispatchPending()`: por cada aviso `PENDIENTE`, llama a `notifyResultUsage`; éxito → `ENTREGADO`; fallo → sigue `PENDIENTE`, `intentos + 1` (depende de T020)

**Checkpoint**: la Venta Bruta tiene su insumo físico y el aviso hacia M2 tiene su mecanismo.

---

## Phase 5: CU09 – Consultar Alimento Requerido al Módulo 2 (Priority: P1)

**Goal**: Sincronizar las partidas de alimento con cantidad y precio unitario, sin calcular ni promediar, marcando las que carecen de precio.

**Independent Test**: `AlimentoSyncService.synchronize()` deja tres partidas del Caso Dorado con `VALORIZADA`; una partida sin precio queda `PENDIENTE_PRECIO`; un lote sin partidas no genera registros.

**Cobertura de FR**: FR-001 → `synchronize()` por cada lote de `CopiaLocalGalponLote` · FR-002 → `PartidaAlimentoLote` · FR-003/FR-004 → el servicio no multiplica ni promedia (regla de arquitectura 3) · FR-005 → `estadoValorizacion` · FR-006 → ausencia registrada en bitácora, cero partidas · FR-007 → captura de `ModuleUnavailableException` · FR-008 → `upsert` por `idPartidaOrigen` · FR-009 → el gateway solo lee.

### Tests for CU09

- [ ] T023 [CU09] `test/sync/m2/CU09ConsultarAlimentoRequeridoM2Test.java`: `CU09-AS-01` sincronización exitosa · `CU09-AS-02` partida sin precio → `PENDIENTE_PRECIO` · `CU09-AS-03` ausencia total (cero partidas, hecho registrado) · `CU09-AS-04` M2 no responde con copia previa (Fase 8, `@Disabled` hasta entonces) · `CU09-AS-05` idempotencia. Más un test de que dos partidas del mismo tipo con precios distintos se conservan por separado (edge "Precios distintos").

### Implementation for CU09

- [ ] T024 [P] [CU09] Crear `PartidaAlimentoLote` y `EstadoValorizacion` en `main/sync/m2/`
- [ ] T025 [CU09] Crear `PartidaAlimentoLoteRepository` e `InMemoryPartidaAlimentoLoteRepository` con `findByLote` (depende de T024)
- [ ] T026 [CU09] Implementar `AlimentoSyncService.synchronize()` (depende de T025, T016)

**Checkpoint**: el Costo de Alimento tiene sus partidas.

---

## Phase 6: CU10 – Consultar Consumo de Medicamento al Módulo 2 (Priority: P1)

**Goal**: Sincronizar los consumos de medicamento con un tramo por recepción y su precio histórico, sin promediar ni calcular.

**Independent Test**: un consumo abastecido por dos recepciones con precios distintos queda con dos `TramoRecepcionConsumo`, cada uno con su precio; un lote sin consumos no genera registros ni bloquea.

**Cobertura de FR**: FR-001 → `synchronize()` · FR-002/FR-003 → `ConsumoMedicamentoLote` + `tramos` · FR-004/FR-005 → sin promedios ni totales · FR-006 → `estadoValorizacion` por tramo · FR-007 → ausencia registrada, no bloquea (lo verifica CU03 en la Fase 8) · FR-008 → captura · FR-009 → `upsert` por `idConsumoOrigen` · FR-010 → solo lectura.

### Tests for CU10

- [ ] T027 [CU10] `test/sync/m2/CU10ConsultarConsumoMedicamentoM2Test.java`: `CU10-AS-01` una recepción · `CU10-AS-02` dos recepciones con precios distintos, sin promediar · `CU10-AS-03` tramo sin precio → `PENDIENTE_PRECIO` · `CU10-AS-04` ausencia total (cero consumos) · `CU10-AS-05` M2 no responde con copia previa (Fase 8, `@Disabled` hasta entonces) · `CU10-AS-06` idempotencia

### Implementation for CU10

- [ ] T028 [P] [CU10] Crear `ConsumoMedicamentoLote` y `TramoRecepcionConsumo` en `main/sync/m2/`
- [ ] T029 [CU10] Crear `ConsumoMedicamentoLoteRepository` e `InMemoryConsumoMedicamentoLoteRepository` con `findByLote` (depende de T028)
- [ ] T030 [CU10] Implementar `MedicamentoSyncService.synchronize()` (depende de T029, T016)

**Checkpoint**: la copia local está completa. Los cuatro servicios de sincronización se pueden encadenar en `SyncScheduler`.

---

## Phase 7: CU01 – Consultar Lista de Galpones (Priority: P1)

**Goal**: Listar galpones y lotes desde la copia local con las acciones de fila habilitadas o deshabilitadas según la precondición de Liquidación, y con el estado de sincronización visible.

**Independent Test**: con la copia local del Caso Dorado, `list()` devuelve una fila por galpón; solo el galpón en `VACIADO_SANITARIO` con alerta y resultado válido tiene `generarLiquidacionHabilitada = true`; con la copia vacía devuelve el mensaje de estado vacío.

**Cobertura de FR**: FR-001 → `GalponRow` (sin aforo, edad ni poblaciones) + `syncStatus` · FR-002 → lee solo `CopiaLocalGalponLote` · FR-003 → `EstadoGalpon` · FR-004/FR-005 → banderas de acción calculadas con `LiquidacionEligibility` y `LiquidacionRepository.findByLote` · FR-006 → `emptyMessage` · FR-007 → el servicio no conoce `Module1Gateway`.

### Tests for CU01

- [ ] T031 [CU01] `test/application/CU01ConsultarListaGalponesTest.java`: `CU01-AS-01` visualización con datos completos y fecha de sincronización · `CU01-AS-02` galpón `EN_COSECHA` → acción deshabilitada con motivo "cierre operativo a cargo de Módulo 2" · `CU01-AS-03` `VACIADO_SANITARIO` con alerta y resultado (o población 0) y sin `ACTIVA` → habilitada · `CU01-AS-04` `DISPONIBLE`/`PRODUCTIVO`/`MANTENIMIENTO`/`AISLAMIENTO` → deshabilitada, informativa · `CU01-AS-05` copia vacía → "No hay galpones disponibles para consultar" · `CU01-AS-06` M1 indisponible tras una sincronización previa → misma lista, fecha de la última exitosa, `modulo1EnFallo = true`. Más el edge "población 0 en `PRODUCTIVO`/`EN_COSECHA`" deshabilitado.

### Implementation for CU01

- [ ] T032 [P] [CU01] Crear en `main/domain/` `Liquidacion`, `EstadoLiquidacion`, `PartidaCostoLote`, `CategoriaCosto`, `SnapshotDatosOrigen`, `FuenteOrigen`, `RegistroAnulacion` (las necesita `LiquidacionRepository`)
- [ ] T033 [CU01] Crear `LiquidacionRepository` (`save`, `findById`, `findByLote`, `findActivaByLote`, `findAll`) e `InMemoryLiquidacionRepository` que lanza si se guarda una segunda `ACTIVA` para el mismo lote; test `InMemoryLiquidacionRepositoryTest` con `@DisplayName("CA-G04 – …")` (depende de T032)
- [ ] T034 [CU01] Crear `LiquidacionEligibility` y `Eligibility` en `main/domain/` con el orden de validaciones de *Diseño transversal* (depende de T033, T016, T020)
- [ ] T035 [CU01] Implementar `ListGalponesService.list()`, `GalponRow` y `GalponListResult` en `main/application/` (depende de T034, T009)

**Checkpoint**: el punto de entrada del módulo funciona sobre la copia local y ya existe la regla de elegibilidad que CU03 reutiliza.

---

## Phase 8: CU03 – Generar Liquidación del Lote (Priority: P1)

**Goal**: Generar la Liquidación `ACTIVA` con los cuatro indicadores exactos, el snapshot por fuente, el marcado del resultado como utilizado y el aviso a M2.

**Independent Test**: con `CasoDorado` completo y precio $4.500, `generate` devuelve Venta Bruta $107.100.000, Mortalidad 500 (5,56 %), Costos $85.000.000, Utilidad $22.100.000, peso promedio 2,80, estado `ACTIVA`; el resultado queda `UTILIZADO` y el gateway falso de M2 recibió el aviso.

**Cobertura de FR**: FR-001..FR-004 → `LiquidacionCalculator` · FR-005 → `LiquidacionEligibility` · FR-006 → `PendingValuationException` · FR-007 → `ActiveLiquidacionExistsException` + repositorio · FR-008 → `Liquidacion` con fecha, hora, usuario, sin setters · FR-009 → solo repositorios locales + `SnapshotDatosOrigen` · FR-010 → marca `UTILIZADO`, crea aviso, `dispatchPending()` · FR-011 → `InvalidPrecioKgException` · FR-012 → `pollosVendidos` y `pesoTotalKg` copiados del resultado; `pesoPromedioKg` calculado · FR-013 → atributos de `Liquidacion`.

### Tests for CU03

- [ ] T036 [CU03] `test/domain/LiquidacionCalculatorTest.java`: Venta Bruta sobre peso total (23.800 × 4.500) y con precio decimal (23.800 × 4.500,50 → 107.111.900); mortalidad y porcentaje (500 / 9.000 → 5,56); siniestro total (0, 9.000, 100,00); utilidad negativa; peso promedio 2,80
- [ ] T037 [CU03] `test/application/CU03GenerarLiquidacionTest.java` con diez `@Test` en el orden del spec: `CU03-AS-01` Caso Dorado · `CU03-AS-02` sin resultado → mensaje literal · `CU03-AS-03` precio ≤ 0 → `InvalidPrecioKgException` con el campo · `CU03-AS-04` partida sin precio → lista de pendientes, ninguna Liquidación guardada · `CU03-AS-05` sin partidas → "No hay partidas de costo sincronizadas para este ciclo" · `CU03-AS-06` siniestro total ($40.000.000, −$40.000.000, 100,00 %) · `CU03-AS-07` `EN_COSECHA` con resultado → bloqueo · `CU03-AS-08` segunda `ACTIVA` → deniega y devuelve la existente · `CU03-AS-09` tras anulación (Fase 9; `@Disabled` hasta entonces) · `CU03-AS-10` M1/M2 indisponibles con copia local → genera y el snapshot conserva las fechas. Más `@DisplayName("CA-G04 – …")` a nivel de servicio, y un test de que un lote sin medicamentos sí liquida (CU10.FR-007)
- [ ] T038 [CU03] Activar los `@Disabled` de CU08 AS-04/05/06, CU09 AS-04 y CU10 AS-05

### Implementation for CU03

- [ ] T039 [P] [CU03] Implementar `LiquidacionCalculator` en `main/domain/` (`ventaBruta`, `mortalidadAves`, `porcentajeMortalidad`, `costosOperativos`, `utilidadNeta`, `pesoPromedio`) usando `Rounding`
- [ ] T040 [P] [CU03] Implementar `CostValuation.subtotal(cantidad, precioUnitario)` → `Rounding.copTotal(cantidad × precioUnitario)`; el impuesto no participa (D-05)
- [ ] T041 [CU03] Implementar `GenerateLiquidacionService.generate` y `GenerateLiquidacionCommand` con los pasos 5–9 de *Diseño transversal*, incluidas las excepciones y sus mensajes literales (depende de T034, T039, T040, T022, T025, T029)

**Checkpoint**: el entregable central del módulo funciona de extremo a extremo con la copia local.

---

## Phase 9: CU04 – Anular Liquidación (Priority: P1)

**Goal**: Anular una Liquidación `ACTIVA` con motivo, fecha, hora y responsable, conservándola íntegra y rehabilitando el lote.

**Independent Test**: `prepare(id, "Precio por kg digitado con error")` + `confirm` deja la Liquidación `ANULADA` con su `RegistroAnulacion`, todos los demás campos iguales, y `generate` sobre el mismo lote crea una nueva `ACTIVA`.

**Cobertura de FR**: FR-001 → opera sobre `Liquidacion` · FR-002 → `LiquidacionAlreadyAnuladaException` · FR-002b → `ActiveLiquidacionExistsException` de CU03 devuelve la existente; CU05/CU06 solo muestran el registro · FR-003 → motivo recortado con 10–500 caracteres · FR-004 → `prepare` devuelve `AnulacionPendiente` con lote y consecuencias; `confirm` es el único que persiste · FR-005/FR-006/FR-008 → `Liquidacion.anular` cambia estado y adjunta registro en una operación · FR-007 → `findActivaByLote` vuelve vacío · FR-009 → sin gateways · FR-010 → `registroAnulacion` expuesto en `DesgloseView` y `HistoryRow`.

### Tests for CU04

- [ ] T042 [CU04] `test/application/CU04AnularLiquidacionTest.java`: `CU04-AS-01` anulación exitosa (estado, registro, campos intactos comparados uno a uno, lote rehabilitado) · `CU04-AS-02` corrección completa de precio (anular + generar; la anterior sigue `ANULADA`) · `CU04-AS-03` motivo vacío o solo espacios → rechazo sin cambios · `CU04-AS-04` ya anulada → deniega y devuelve el registro existente · `CU04-AS-05` `prepare` sin `confirm` → sin cambios ni registro
- [ ] T043 [CU04] Activar `CU03-AS-09`

### Implementation for CU04

- [ ] T044 [CU04] Implementar `Liquidacion.anular(RegistroAnulacion)` (lanza si ya `ANULADA`) en `main/domain/`
- [ ] T045 [CU04] Implementar `AnnulLiquidacionService` (`prepare`, `confirm`) y `AnulacionPendiente` en `main/application/` (depende de T044)

**Checkpoint**: el ciclo `ACTIVA → ANULADA → nueva ACTIVA` funciona; CA-G08 y SC-006 verificables.

---

## Phase 10: CU05 – Consultar Desglose de Ventas y Gastos (Priority: P2)

**Goal**: Proyectar el desglose por categorías desde la Liquidación y sus partidas, con suma exacta a `costosOperativosCop`, y dejar el puerto de exportación.

**Independent Test**: `view(id)` del Caso Dorado devuelve 3 líneas `ALIMENTO`, 2 `MEDICINA`, 1 `POBLACION`, cuya suma es $85.000.000; `export` sobre una `ANULADA` lanza `ExportNotAllowedException`.

**Cobertura de FR**: FR-001 → `DesgloseView` con ingresos y categorías · FR-002 → test de suma exacta · FR-003 → `DesgloseExporter` (puerto) · FR-004 → `ExportNotAllowedException` + `registroAnulacion` en la vista · FR-005 → `ExportFailedException` sin alterar la vista · FR-006 → solo `LiquidacionRepository` · FR-007 → `DesgloseView` es un record que no se guarda.

### Tests for CU05

- [ ] T046 [CU05] `test/application/CU05ConsultarDesgloseTest.java`: `CU05-AS-01` desglose completo por categorías con suma exacta · `CU05-AS-02` exportación exitosa (el `FakeDesgloseExporter` recibe la misma `DesgloseView` de pantalla, con Matriz de Venta Final y desglose) · `CU05-AS-03` Liquidación anulada → vista de solo lectura con registro de anulación y exportación denegada · `CU05-AS-04` fallo del exportador → `ExportFailedException`, `view` sigue devolviendo lo mismo · `CU05-AS-05` siniestro total → ingresos con Venta Bruta 0 y aviso de "no registró venta"

### Implementation for CU05

- [ ] T047 [P] [CU05] Crear `DesgloseView`, `DesgloseLine`, `DesgloseExporter` (interfaz `void export(DesgloseView view, String usuario)`), `ExportNotAllowedException`, `ExportFailedException` en `main/application/`
- [ ] T048 [P] [CU05] Crear `test/support/FakeDesgloseExporter.java` (captura la vista; modo de fallo)
- [ ] T049 [CU05] Implementar `DesgloseQueryService` (`view`, `export`) (depende de T047)

**Checkpoint**: CA-G05 y SC-003 (trazabilidad ≤ 3 pasos) quedan sustentados por `fuenteOrigen` y `referenciaOrigen` en cada línea.

---

## Phase 11: CU06 – Consultar Historial de Liquidaciones (Priority: P3)

**Goal**: Listar Liquidaciones en orden descendente con filtros por galpón y rango de fechas, distinguiendo anuladas y los dos estados vacíos.

**Independent Test**: con tres Liquidaciones (dos galpones, una anulada), `query` sin filtros las devuelve en orden descendente; con filtro de galpón devuelve solo las suyas; `desde > hasta` lanza `InvalidDateRangeException`.

**Cobertura de FR**: FR-001 → orden descendente por `fechaHoraGeneracion` · FR-002 → `HistoryRow` · FR-003 → `HistoryFilter` · FR-004 → `estado` + `registroAnulacion` en la fila · FR-005 → `InvalidDateRangeException` · FR-006 → `HistoryRow.idLiquidacion` para abrir CU05 · FR-007 → `EmptyReason` · FR-008 → solo `LiquidacionRepository`.

### Tests for CU06

- [ ] T050 [CU06] `test/application/CU06ConsultarHistorialTest.java`: `CU06-AS-01` orden descendente y columnas · `CU06-AS-02` filtro por galpón y rango · `CU06-AS-03` anuladas identificables con su registro · `CU06-AS-04` vacío total vs. sin coincidencias (`NO_LIQUIDACIONES` / `NO_MATCHES`) · `CU06-AS-05` rango invertido rechazado. Más el edge "lote desvinculado sigue filtrable por galpón".

### Implementation for CU06

- [ ] T051 [P] [CU06] Crear `HistoryFilter`, `HistoryRow`, `HistoryResult`, `EmptyReason`, `InvalidDateRangeException` en `main/application/`
- [ ] T052 [CU06] Implementar `LiquidacionHistoryService.query` (depende de T051)

**Checkpoint**: los 9 casos de uso operan con `mvn test` sobre la copia local (CA-G01 en su versión de librería).

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: Cerrar la trazabilidad y dejar el repositorio listo para la revisión del equipo.

- [ ] T053 [P] Recorrer los 9 specs y verificar que cada acceptance scenario tiene su `@Test` con el `@DisplayName` correcto y en el orden del spec (CA-G02); anotar en este plan los que quedaron `@Disabled` y por qué
- [ ] T054 [P] Cablear los cuatro servicios de sincronización y `UsageNoticeDispatcher` en un ejemplo de arranque (`SyncScheduler`) documentado en `README.md`, sin pruebas
- [ ] T055 [P] Actualizar `README.md` con cómo ejecutar `mvn test` y un resumen de los servicios de `application` para el equipo que haga las pantallas P1–P5
- [ ] T056 Revisar las reglas de arquitectura 1–5 en una lectura de código y registrar hallazgos en `PENDIENTES.md` si algo se aparta
- [ ] T057 Pegar el resumen de surefire en el PR de cierre (checklist `AGENTS.md` §12)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup y bloquea todos los CU.
- **CU07 (Phase 3)**: depende de Foundational. Bloquea a CU09 y CU10 (recorren los lotes de la copia local) y a CU01.
- **CU08, CU09, CU10 (Phases 4–6)**: dependen de Foundational (y CU09/CU10 de CU07 por `CopiaLocalGalponLote`). Entre sí son independientes y se pueden repartir en paralelo.
- **CU01 (Phase 7)**: depende de CU07 y CU08 (elegibilidad). Crea `Liquidacion`, `LiquidacionRepository` y `LiquidacionEligibility`, que CU03 reutiliza.
- **CU03 (Phase 8)**: depende de CU01 y de las cuatro sincronizaciones. Activa los tests diferidos de CU08/CU09/CU10.
- **CU04 (Phase 9)**: depende de CU03. Activa `CU03-AS-09`.
- **CU05 y CU06 (Phases 10–11)**: dependen de CU03 y CU04 (muestran el registro de anulación); son independientes entre sí y se hacen en paralelo.
- **Polish (Phase 12)**: depende de todo lo anterior.

Es el orden de `spec.md` y `AGENTS.md` §5: **CU07 → CU10 → CU01 → CU03 → CU04 → CU05 ∥ CU06**.

### Within Each CU

- Entidades y repositorio antes que el servicio.
- Test de acceptance escrito primero con los valores del spec; se ejecuta al terminar la implementación.
- `mvn test` en verde antes de abrir el PR de la rama `feature/cuNN-…` hacia `develop`.
- Un CU por rama; los tests diferidos (`@Disabled("Fase N")`) se activan en la rama del CU que los completa.

## Open Questions

### Decisiones abiertas con otros módulos (D-01..D-07)

No se resuelven en código. Cada una queda en un único punto con un supuesto que el equipo puede cambiar sin tocar el resto.

| # | Punto único en el código | Supuesto de este plan |
|---|---|---|
| D-01 (criterio de costeo del alimento) | `Module2Gateway.fetchPartidasAlimento` y el adaptador futuro | La fuente que M2 decida se mapea a `PartidaAlimentoLote` con `cantidadKg` y `precioUnitarioKgCop`; el dominio no distingue si la cantidad es comprada, consumida o proyectada. El fixture usa cantidades genéricas. |
| D-02 (exposición formal de M1) | `Module1Gateway` y los records `Remote…` | Los campos de CU07.FR-002..FR-004 existen tal cual. El plan general de M1 (rama `plan-general-modulo-1`) propone `GET /galpones/{id}` con `loteActivo { poblacionInicial, poblacionActual, fechaIngreso, costoTotal }`, compatible con este puerto; no expone alertas de vaciado, punto a acordar. |
| D-03 (aviso de utilización) | `Module2Gateway.notifyResultUsage` + `UsageNoticeDispatcher` | Aviso con `idResultado` e `idLiquidacion`, sin confirmación de vuelta; idempotente por `idAviso`. |
| D-04 (siniestro total en `Productivo`) | `LiquidacionEligibility` paso 2 | Se exige `VACIADO_SANITARIO` + alerta también para el siniestro total, como dice CU03.FR-005. El fixture de AS-06 lleva la alerta. Si M1 abre otra ruta, cambia solo ese paso. |
| D-05 (impuesto de M2) | `CostValuation.subtotal` y `PartidaCostoLote.impuestoCop` | Costos sobre valor neto; el impuesto se conserva por partida para poder sumarlo después sin re-sincronizar. |
| D-06 (mortalidad fuera de `Productivo`) | `LiquidacionCalculator.mortalidadAves` | Se calcula con las poblaciones tal como las informa M1, sin corrección. |
| D-07 (grafía de estados) | `EstadoGalpon.fromLabel` | Se acepta la grafía de M1 (`"En cosecha"`, `"Vaciado sanitario"`) sin distinguir mayúsculas; cualquier otra cadena es incidencia. |

### Supuestos propios del plan (a confirmar con el equipo)

1. **Desglose del Caso Dorado.** Los specs dan solo los totales ($85.000.000 y $40.000.000); el plan fija las partidas. Hay que validarlas con el stakeholder (CA-G03).
2. **Partida de población con `cantidad = 1`.** Alternativa: `cantidad = poblacionInicial` con precio por ave, que introduce redondeo. Se prefirió la fila única.
3. **Redondeo por partida.** Cada `subtotalCop` se redondea a entero y los Costos Operativos son la suma; garantiza CU05.FR-002. Alternativa descartada: redondear solo el total.
4. **Ausencia de alimento bloquea; ausencia de medicina no.** Combina CU03 AS-05, CU09 AS-03 y CU10 AS-04. Mensaje de CU03 AS-05 cuando no hay ninguna partida; "No hay partidas de alimento sincronizadas para este ciclo" cuando solo falta alimento.
5. **Fecha del snapshot por fuente** = última sincronización exitosa de esa fuente en la bitácora, más las referencias de los registros usados. Alternativa: la fecha propia de cada registro; se descartó por producir varias fechas por fuente.
6. **`pesoPromedioKg` con 2 decimales** (`2,80`); el spec muestra `2,8`. Es presentación (CU03.FR-012) y no entra en ningún cálculo.
7. **Anulación en dos pasos** (`prepare`/`confirm`) para representar la confirmación de CU04.FR-004 sin UI.
8. **Exportación a Excel** solo como puerto. Implementarla requiere Apache POI: decisión del equipo sobre `pom.xml`.
9. **Persistencia definitiva y framework web**: fuera de este plan; depende de la alineación con M1/M2 (Java 21 + Spring + PostgreSQL). El diseño de puertos y repositorios está pensado para que ese cambio no toque `domain` ni `application`.
10. **Incidencias como lista de texto** en `RegistroSincronizacion`, en vez de una entidad nueva, para no salir del glosario.

## Notes

- `[P]` = puede hacerse en paralelo con otras tareas `[P]` del mismo bloque.
- `[CUNN]` enlaza la tarea con su caso de uso y con la rama `feature/cuNN-…`.
- Cada CU debe poder completarse y probarse por separado con los dobles de `test/support`.
- `mvn test` en verde antes de cada commit; un CU por PR hacia `develop`.
- Detenerse en cada checkpoint para validar el CU con el equipo.
- Si al implementar aparece algo que contradice un spec, no se cambia el código hacia el documento raíz: se abre el punto en `CAMBIOS.md` (`AGENTS.md` §2).
