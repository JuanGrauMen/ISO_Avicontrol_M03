# Implementation Plan: Sincronización de Datos Locales (M1 y M2)

**Date**: 2026-09-28  
**Actualizado**: 2026-10-02  
**Specs**:
- [m3-cu07-consultar-galpon-lote-m1](../m3-cu07-consultar-galpon-lote-m1/spec.md) – Consultar Galpón y Lote al M1  
- [m3-cu08-consultar-resultado-sacrificio-m2](../m3-cu08-consultar-resultado-sacrificio-m2/spec.md) – Consultar Resultado Final de Sacrificio al M2  
- [m3-cu09-consultar-alimento-requerido-m2](../m3-cu09-consultar-alimento-requerido-m2/spec.md) – Consultar Alimento Requerido al M2  
- [m3-cu10-consultar-consumo-medicamento-m2](../m3-cu10-consultar-consumo-medicamento-m2/spec.md) – Consultar Consumo de Medicamento al M2  

## Summary

Este subsistema se encarga de la **sincronización e ingesta de datos locales** desde Módulo 1 (Galpones, Lotes y Alertas de Vaciado Sanitario) y Módulo 2 (Resultados Finales de Sacrificio, Partidas de Alimento y Consumos de Medicamentos con sus tramos por recepción). Mantiene la copia local aislada en la base de datos de Módulo 3 de forma incremental e idempotente, registra la bitácora de sincronización (`registro_sincronizacion`) y emite el aviso de utilización de resultados de sacrificio (`aviso_utilizacion_resultado`) de forma asíncrona reintentable por REST o Apache Kafka.

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, `@Scheduled`, Kafka (`spring-kafka`) / REST (`RestClient`), Jakarta Validation, JUnit 5, MockRestServiceServer, Testcontainers
- **Storage**: PostgreSQL (producción) / H2 en memoria (dev/test), gestionado mediante Flyway (`V1__create_sync_tables.sql` y `V2__create_partidas_tables.sql`)
- **Testing**: JUnit 5, Mockito, Spring Boot Test, MockRestServiceServer (verificación de endpoints M1/M2)
- **Target Platform**: JVM 21 / Docker Container
- **Project Type**: Backend service (capa de sincronización e integración)
- **Performance Goals**: Sincronización periódica eficiente (< 3s por ciclo), reintentos con backoff ante fallos de conexión
- **Constraints**: Operación autónoma en M3 sobre copia local previa si M1 o M2 están indisponibles; actualización idempotente sin duplicar registros ni alterar datos de liquidaciones generadas.

---

## Intercambio de Datos y Payloads JSON

### A. Payload Recibido de M1 (Galpón y Lote)
```json
{
  "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
  "nombreGalpon": "Galpón 1",
  "aforoMaximo": 10000,
  "estado": "VACIADO_SANITARIO",
  "loteVigente": {
    "idLote": "98765432-e89b-12d3-a456-426614174000",
    "nombreLote": "Lote L-2026-A",
    "fechaIngreso": "2026-08-01",
    "poblacionInicial": 9000,
    "poblacionActual": 8500,
    "costoTotalCop": 18000000
  },
  "fechaHoraSync": "2026-10-02T14:00:00Z"
}
```

### B. Payload Recibido de M2 (Resultado Final de Sacrificio)
```json
{
  "idResultado": "456e7890-e89b-12d3-a456-426614174000",
  "idLote": "98765432-e89b-12d3-a456-426614174000",
  "idGalpon": "123e4567-e89b-12d3-a456-426614174000",
  "cantidadFinalPollos": 8500,
  "pesoTotalKg": 23800.00,
  "fechaRegistro": "2026-09-25T18:00:00Z"
}
```

### C. Payload Saliente de Aviso de Utilización (M3 → M2)
```json
{
  "idAviso": 101,
  "idResultado": "456e7890-e89b-12d3-a456-426614174000",
  "idLiquidacion": 1045,
  "fechaHoraEmision": "2026-10-02T15:30:00Z",
  "estadoEntrega": "ENTREGADO"
}
```

---

## Integración con Apache Kafka

- **Tópico de Lectura M1**: `avicontrol.m1.alertas-vaciado`
- **Tópico de Lectura M2**: `avicontrol.m2.resultados-sacrificio`
- **Tópico de Escritura M3**: `avicontrol.m3.avisos-utilizacion`

---

## Phase 1: Foundational – Esquema SQL e Infraestructura de Sincronización

**Purpose**: Crear tablas de copia local y repositorios base.

- [ ] **T001** Crear migración Flyway `V1__create_sync_tables.sql` (`registro_sincronizacion`, `galpon`, `lote`, `alerta_vaciado_sanitario`, `resultado_final_sacrificio`, `aviso_utilizacion_resultado`).
- [ ] **T002** Crear migración Flyway `V2__create_partidas_tables.sql` (`partida_alimento_lote`, `consumo_medicamento_lote`, `tramo_recepcion_consumo`).
- [ ] **T003** Crear entidades JPA y repositorios Spring Data para las tablas de copia local y bitácora.
- [ ] **T004** Crear modelos de dominio y mappers Entity ↔ Domain.

---

## Phase 2: CU07 – Sincronización M1 (Galpones, Lotes y Alertas)

**Purpose**: Sincronización periódica idempotente desde Módulo 1.

- [ ] **T005** Crear puerto `Modulo1Port` y su adaptador `Modulo1RestAdapter` / `Modulo1KafkaConsumer`.
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
- [ ] **T014** Configurar `@Scheduled(cron = "${sync.cron.m2:0 */15 * * * *}")` en `SyncSchedulerConfig.java` para disparar periódicamente las tareas de ingesta.
