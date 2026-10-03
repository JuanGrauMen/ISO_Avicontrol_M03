# Implementation Plan: Sincronización de Datos Locales (M1 y M2)

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
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

---

## Esquema SQL DDL de la Copia Local (Flyway V1 y V2)

```sql
-- V1__create_sync_tables.sql
CREATE TABLE registro_sincronizacion (
    id                BIGSERIAL PRIMARY KEY,
    fuente            VARCHAR(20) NOT NULL,
    fecha_hora_inicio TIMESTAMP NOT NULL,
    fecha_hora_fin    TIMESTAMP,
    resultado         VARCHAR(20),
    descripcion_error TEXT,
    registros_actualizados INT DEFAULT 0
);

CREATE TABLE galpon (
    id_galpon     UUID PRIMARY KEY,
    nombre        VARCHAR(100) NOT NULL,
    aforo_maximo  INT NOT NULL,
    estado        VARCHAR(30) NOT NULL,
    fecha_hora_sync TIMESTAMP NOT NULL
);

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

CREATE TABLE alerta_vaciado_sanitario (
    id_alerta          UUID PRIMARY KEY,
    id_galpon          UUID NOT NULL REFERENCES galpon(id_galpon),
    id_lote            UUID NOT NULL REFERENCES lote(id_lote),
    fecha_hora_evento  TIMESTAMP NOT NULL
);
```

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

---

## Snippets de Código Java de Puertos e Integración

### Puerto de Salida: `Modulo1Port.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.out;

import co.edu.unimagdalena.avicontrol.domain.model.Galpon;
import co.edu.unimagdalena.avicontrol.domain.model.AlertaVaciadoSanitario;
import java.util.List;

public interface Modulo1Port {
    /** CU07 — Sincroniza galpones con su lote activo vigente. */
    List<Galpon> obtenerGalponesYLotesVigentes();

    /**
     * CU07 — Recupera las alertas de vaciado sanitario emitidas por M1
     * desde la última sincronización. Cada alerta indica que un lote fue
     * desvinculado de su galpón y está disponible para liquidar (CU01.FR-013,
     * CU03.FR-005). Las alertas se almacenan en la tabla local
     * {@code alerta_vaciado_sanitario} y son inmutables una vez recibidas.
     */
    List<AlertaVaciadoSanitario> obtenerAlertasVaciadoSanitario();
}
```

### Consumidor Kafka: `SacrificioKafkaListener.java`
```java
package co.edu.unimagdalena.avicontrol.infrastructure.adapter.client;

import co.edu.unimagdalena.avicontrol.application.service.sync.SyncSacrificioService;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class SacrificioKafkaListener {

    private final SyncSacrificioService syncSacrificioService;

    public SacrificioKafkaListener(SyncSacrificioService syncSacrificioService) {
        this.syncSacrificioService = syncSacrificioService;
    }

    @KafkaListener(topics = "avicontrol.m2.resultados-sacrificio", groupId = "avicontrol-modulo3-sync-group")
    public void consumirResultadoSacrificio(String eventoJson) {
        syncSacrificioService.procesarEventoResultadoSacrificio(eventoJson);
    }
}
```

### Servicio de Sincronización: `SyncModulo1Service.java`
```java
package co.edu.unimagdalena.avicontrol.application.service.sync;

import co.edu.unimagdalena.avicontrol.domain.port.out.GalponRepository;
import co.edu.unimagdalena.avicontrol.domain.port.out.AlertaVaciadoSanitarioRepository;
import co.edu.unimagdalena.avicontrol.domain.port.out.Modulo1Port;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SyncModulo1Service {

    private final Modulo1Port modulo1Port;
    private final GalponRepository galponRepository;
    private final AlertaVaciadoSanitarioRepository alertaRepository;

    public SyncModulo1Service(Modulo1Port modulo1Port,
                               GalponRepository galponRepository,
                               AlertaVaciadoSanitarioRepository alertaRepository) {
        this.modulo1Port = modulo1Port;
        this.galponRepository = galponRepository;
        this.alertaRepository = alertaRepository;
    }

    @Scheduled(cron = "${sync.cron.m1:0 */15 * * * *}")
    @Transactional
    public void ejecutarSincronizacionModulo1() {
        try {
            // 1. Sincronizar galpones y lotes activos
            var galpones = modulo1Port.obtenerGalponesYLotesVigentes();
            galpones.forEach(galponRepository::guardarOActualizar);

            // 2. Sincronizar alertas de vaciado sanitario (CU01.FR-013, CU03.FR-005)
            // Las alertas son inmutables: solo se insertan si no existen (idempotente por id_alerta)
            var alertas = modulo1Port.obtenerAlertasVaciadoSanitario();
            alertas.forEach(alertaRepository::guardarSiNoExiste);
        } catch (Exception e) {
            // Log en registro_sincronizacion sin romper el timer
        }
    }
}
```

---

## Phase 1: Foundational – Esquema SQL e Infraestructura de Sincronización

- [ ] **T001** Crear migración Flyway `V1__create_sync_tables.sql`.
- [ ] **T002** Crear migración Flyway `V2__create_partidas_tables.sql`.
- [ ] **T003** Crear entidades JPA y repositorios Spring Data.
- [ ] **T004** Crear modelos de dominio y mappers.

---

## Phase 2: CU07 – Sincronización M1 (Galpones, Lotes y Alertas)

- [ ] **T005** Crear puerto `Modulo1Port` y su adaptador `Modulo1RestAdapter` / `SacrificioKafkaListener`.
- [ ] **T006** Unit Test `SyncModulo1ServiceTest.java`.
- [ ] **T007** Crear `SyncModulo1Service.java`.

---

## Phase 3: CU08 – Sincronización M2 Sacrificio y Aviso de Utilización

- [ ] **T008** Crear puertos `Modulo2SacrificioPort` y `AvisoUtilizacionPort`.
- [ ] **T009** Unit Test `SyncSacrificioServiceTest.java`.
- [ ] **T010** Crear `SyncSacrificioService.java` y `AvisoUtilizacionService.java`.

---

## Phase 4: CU09 y CU10 – Sincronización M2 (Alimento y Medicamentos)

- [ ] **T011** Crear puertos `Modulo2AlimentoPort` y `Modulo2MedicamentoPort`.
- [ ] **T012** Unit Test `SyncAlimentoServiceTest.java` y `SyncMedicamentoServiceTest.java`.
- [ ] **T013** Implementar `SyncAlimentoService.java` y `SyncMedicamentoService.java`.
- [ ] **T014** Configurar `@Scheduled` en `SyncSchedulerConfig.java`.
