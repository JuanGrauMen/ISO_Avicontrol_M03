# Implementation Plan: Gestión del Ciclo de Vida de Liquidaciones

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
**Specs**:
- [m3-cu03-generar-liquidacion](../m3-cu03-generar-liquidacion/spec.md) – Generar Liquidación del Lote  
- [m3-cu04-anular-liquidacion](../m3-cu04-anular-liquidacion/spec.md) – Anular Liquidación  

## Summary

Este plan cubre el núcleo de dominio del Módulo 3: la **generación inmutable de la Liquidación** de un lote (`CU03`) y su **anulación auditable** (`CU04`).

**Generación en dos pasos (CU03.FR-015)**: El usuario ingresa primero el precio por kg (paso 1 — captura); el sistema genera una **vista previa no persistida** que muestra la Matriz de Venta Final con todos los indicadores (paso 2 — vista previa); el usuario confirma "Generar liquidación" para persistir. "Volver" regresa sin crear ningún registro. La vista previa se identifica con el aviso "Vista previa · sin generar" — que **no es un estado** de la Liquidación.

**Consulta de Liquidación existente (CU03.FR-014)**: Desde la lista de lotes (acción "Ver liquidación") o desde el historial, se abre la vista de la Liquidación (`ACTIVA` o `ANULADA`) con la Matriz de Venta Final, subtotales por categoría (Alimento, Insumos Médicos, Costo de Población), fecha de sincronización de cada fuente y — si está `ACTIVA` — el botón para iniciar la anulación (CU04.FR-002b).

**Anulación (CU04)**: Se inicia exclusivamente desde la vista de una Liquidación `ACTIVA`. Requiere motivo justificado (10-500 caracteres), confirmación explícita y opera en transacción atómica. Preserva el registro histórico sin borrar filas y rehabilita el lote para una nueva liquidación.

Calcula Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta con redondeo `HALF_UP` a enteros COP y 2 decimales para porcentajes. Bloquea duplicados activos (`CONSTRAINT uq_lote_activa`).

## Technical Context

- **Language/Version**: Java 21 (LTS)
- **Primary Dependencies**: Spring Boot 3.3.x, Spring Data JPA, Spring Web MVC, Jakarta Validation, Flyway (`V3__create_liquidacion_tables.sql`), JUnit 5, MockMvc
- **Storage**: PostgreSQL / H2 en memoria (`liquidacion`, `partida_costo_lote`, `snapshot_datos_origen`, `registro_anulacion`)
- **Testing**: Unit tests exhaustivos del cálculo financiero y pruebas de integración REST end-to-end
- **Target Platform**: JVM 21 / REST API

---

## Esquema SQL DDL (Flyway V3)

```sql
CREATE TABLE liquidacion (
    id_liquidacion      BIGSERIAL PRIMARY KEY,
    id_lote             UUID NOT NULL REFERENCES lote(id_lote),
    id_galpon           UUID NOT NULL REFERENCES galpon(id_galpon),
    id_resultado_sacrificio UUID REFERENCES resultado_final_sacrificio(id_resultado),
    pollos_vendidos     INT,
    peso_total_kg       NUMERIC(12,2),
    peso_promedio_kg    NUMERIC(8,4),
    precio_kg_cop       NUMERIC(14,2),       -- NULL en siniestro total
    venta_bruta_cop     BIGINT NOT NULL,
    mortalidad_aves     INT NOT NULL,
    porcentaje_mortalidad NUMERIC(5,2) NOT NULL,
    costos_operativos_cop BIGINT NOT NULL,
    utilidad_neta_cop   BIGINT NOT NULL,
    estado              VARCHAR(10) NOT NULL DEFAULT 'ACTIVA',
    fecha_hora_generacion TIMESTAMP NOT NULL,
    usuario_responsable VARCHAR(100) NOT NULL,
    CONSTRAINT chk_estado CHECK (estado IN ('ACTIVA', 'ANULADA')),
    CONSTRAINT uq_lote_activa UNIQUE (id_lote)  -- solo una ACTIVA por lote
);
```

---

## JSON Payloads

### Paso 1 — Captura del Precio (Request)

```json
POST /api/v1/liquidaciones/previa

{
  "idLote": "a1b2c3d4-...",
  "idGalpon": "f1e2d3c4-...",
  "precioKgCop": 5800.00,
  "usuario": "jgr@avicontrol.co"
}
```

### Paso 2 — Vista Previa (Response — NO se persiste)

```json
HTTP/1.1 200 OK
{
  "esVistaPrevia": true,
  "aviso": "Vista previa · sin generar",
  "idLote": "a1b2c3d4-...",
  "nombreLote": "Lote Norte Ciclo 4",
  "nombreGalpon": "Galpón Norte",
  "precioKgCop": 5800.00,
  "pollosVendidos": 9600,
  "pesoTotalKg": 24000.00,
  "pesoPromedioKg": 2.50,
  "ventaBrutaCop": 139200000,
  "mortalidadAves": 400,
  "porcentajeMortalidad": 4.00,
  "subtotalesCategoria": {
    "alimento": 45000000,
    "insumosMedicos": 3200000,
    "costoPoblacion": 12000000
  },
  "costosOperativosCop": 60200000,
  "utilidadNetaCop": 79000000,
  "fechaSincronizacionM1": "2026-10-01T08:00:00Z",
  "fechaSincronizacionM2": "2026-10-02T14:30:00Z"
}
```

### Confirmación — Generar Liquidación

```json
POST /api/v1/liquidaciones

{
  "idLote": "a1b2c3d4-...",
  "idGalpon": "f1e2d3c4-...",
  "precioKgCop": 5800.00,
  "usuario": "jgr@avicontrol.co"
}

HTTP/1.1 201 Created
{
  "idLiquidacion": 42,
  "estado": "ACTIVA",
  "idLote": "a1b2c3d4-...",
  ...
}
```

### Consulta de Liquidación Existente (FR-014)

```json
GET /api/v1/liquidaciones/42

HTTP/1.1 200 OK
{
  "idLiquidacion": 42,
  "estado": "ACTIVA",
  "nombreLote": "Lote Norte Ciclo 4",
  "nombreGalpon": "Galpón Norte",
  "precioKgCop": 5800.00,
  "pollosVendidos": 9600,
  "pesoTotalKg": 24000.00,
  "pesoPromedioKg": 2.50,
  "ventaBrutaCop": 139200000,
  "mortalidadAves": 400,
  "porcentajeMortalidad": 4.00,
  "subtotalesCategoria": {
    "alimento": 45000000,
    "insumosMedicos": 3200000,
    "costoPoblacion": 12000000
  },
  "costosOperativosCop": 60200000,
  "utilidadNetaCop": 79000000,
  "fechaHoraGeneracion": "2026-10-03T11:00:00Z",
  "usuarioResponsable": "jgr@avicontrol.co",
  "fechaSincronizacionM1": "2026-10-01T08:00:00Z",
  "fechaSincronizacionM2": "2026-10-02T14:30:00Z",
  "accionesDisponibles": ["VER_DESGLOSE", "ANULAR"]
}
```

### Error — Segunda Liquidación Activa (409 Conflict)

```json
HTTP/1.1 409 Conflict
{
  "type": "https://avicontrol.co/errors/liquidacion-duplicada",
  "title": "Liquidación duplicada",
  "status": 409,
  "detail": "El lote a1b2c3d4-... ya tiene una Liquidación ACTIVA (id=42). Anúlala antes de generar una nueva.",
  "instance": "/api/v1/liquidaciones"
}
```

### Siniestro Total

```json
POST /api/v1/liquidaciones

{
  "idLote": "b2c3d4e5-...",
  "idGalpon": "e2d3c4b5-...",
  "precioKgCop": null,
  "usuario": "jgr@avicontrol.co"
}

HTTP/1.1 201 Created
{
  "idLiquidacion": 43,
  "estado": "ACTIVA",
  "ventaBrutaCop": 0,
  "pollosVendidos": 0,
  "siniestroTotal": true,
  ...
}
```

### Anulación (Request/Response CU04)

```json
PUT /api/v1/liquidaciones/42/anulacion

{
  "motivo": "Error en el precio por kg ingresado. Se corrige en nueva liquidación.",
  "usuario": "jgr@avicontrol.co"
}

HTTP/1.1 200 OK
{
  "idLiquidacion": 42,
  "estado": "ANULADA",
  "registroAnulacion": {
    "idAnulacion": 7,
    "motivo": "Error en el precio por kg ingresado. Se corrige en nueva liquidación.",
    "fechaHoraAnulacion": "2026-10-03T12:00:00Z",
    "usuarioResponsable": "jgr@avicontrol.co"
  }
}
```

---

## Snippets de Código Java de Puertos, DTOs y Servicios

### Puertos de Entrada
```java
package co.edu.unimagdalena.avicontrol.domain.port.in;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LiquidacionPreviaDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.dto.LiquidacionDto;
import co.edu.unimagdalena.avicontrol.domain.port.in.dto.AnulacionDto;
import java.math.BigDecimal;
import java.util.UUID;

/** CU03 — Paso 1+2: calcular vista previa sin persistir */
public interface PreviewLiquidacionUseCase {
    LiquidacionPreviaDto calcularVistaPrevia(UUID idLote, UUID idGalpon,
                                              BigDecimal precioKgCop, String usuario);
}

/** CU03 — Paso 2 confirmado: persistir la Liquidación */
public interface GenerarLiquidacionUseCase {
    LiquidacionDto generarLiquidacion(UUID idLote, UUID idGalpon,
                                      BigDecimal precioKgCop, String usuario);
}

/** CU03.FR-014 — Consultar Liquidación existente */
public interface ConsultarLiquidacionUseCase {
    LiquidacionDto consultarLiquidacion(Long idLiquidacion);
}

/** CU04 — Anular Liquidación ACTIVA */
public interface AnularLiquidacionUseCase {
    AnulacionDto anularLiquidacion(Long idLiquidacion, String motivo, String usuario);
}
```

### Servicio de Vista Previa + Generación: `GenerarLiquidacionService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.domain.model.Liquidacion;
import co.edu.unimagdalena.avicontrol.domain.shared.Rounding;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.UUID;

@Service
public class GenerarLiquidacionService
        implements PreviewLiquidacionUseCase, GenerarLiquidacionUseCase {

    @Override
    public LiquidacionPreviaDto calcularVistaPrevia(UUID idLote, UUID idGalpon,
                                                     BigDecimal precioKgCop, String usuario) {
        // Validaciones previas (FR-005, FR-006, FR-007, FR-009) sin persistir nada
        validarPrecondiciones(idLote, precioKgCop);
        var calculo = calcular(idLote, precioKgCop);
        // Retorna DTO con esVistaPrevia=true; NO se llama a liquidacionRepository.guardar()
        return LiquidacionMapper.toPreviaDto(calculo);
    }

    @Override
    @Transactional
    public LiquidacionDto generarLiquidacion(UUID idLote, UUID idGalpon,
                                              BigDecimal precioKgCop, String usuario) {
        // Se re-ejecutan validaciones antes de persistir (FR-015: validaciones en ambos pasos)
        validarPrecondiciones(idLote, precioKgCop);
        var calculo = calcular(idLote, precioKgCop);

        var liq = Liquidacion.crearActiva(idLote, idGalpon,
                calculo.sac(), precioKgCop,
                calculo.ventaBruta(), calculo.costosOperativos(), calculo.utilidadNeta(),
                usuario, clock.instant());
        liquidacionRepository.guardar(liq);
        return LiquidacionMapper.toDto(liq);
    }

    private void validarPrecondiciones(UUID idLote, BigDecimal precioKgCop) {
        // FR-007: no exista otra ACTIVA
        if (liquidacionRepository.existeActivaPorLote(idLote)) {
            throw new LiquidacionDuplicadaException(idLote);
        }
        // FR-011: precio > 0 (null solo en siniestro total)
        if (precioKgCop != null && precioKgCop.compareTo(BigDecimal.ZERO) <= 0) {
            throw new PrecioInvalidoException();
        }
    }

    private Calculo calcular(UUID idLote, BigDecimal precioKgCop) {
        var sac = resultadoSacrificioRepository.buscarPorLote(idLote);
        var lote = loteRepository.buscarPorId(idLote);
        boolean esSiniestro = sac == null; // FR-005: siniestro total sin resultado

        long ventaBruta = esSiniestro ? 0L
            : Rounding.copTotal(sac.getPesoTotalKg().multiply(precioKgCop));
        long costosOperativos = calcularCostosOperativos(idLote);
        long utilidadNeta = ventaBruta - costosOperativos;
        return new Calculo(sac, lote, ventaBruta, costosOperativos, utilidadNeta);
    }
}
```

### Servicio de Anulación: `AnularLiquidacionService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.domain.model.RegistroAnulacion;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AnularLiquidacionService implements AnularLiquidacionUseCase {

    @Override
    @Transactional  // FR-008: transacción atómica
    public AnulacionDto anularLiquidacion(Long idLiquidacion, String motivo, String usuario) {
        var liq = liquidacionRepository.buscarPorId(idLiquidacion)
            .orElseThrow(() -> new LiquidacionNoEncontradaException(idLiquidacion));

        // FR-002: solo se anula si está ACTIVA
        if (!liq.estaActiva()) {
            throw new LiquidacionYaAnuladaException(idLiquidacion);
        }

        // FR-003: motivo obligatorio, mínimo 10 caracteres, máximo 500
        if (motivo == null || motivo.isBlank() || motivo.length() < 10 || motivo.length() > 500) {
            throw new MotivoInvalidoException();
        }

        // FR-006: solo cambia el estado; NO se tocan otros atributos
        liq.anular();
        var registro = RegistroAnulacion.crear(liq.getId(), motivo, clock.instant(), usuario);
        anulacionRepository.guardar(registro);

        return AnulacionMapper.toDto(liq, registro);
    }
}
```

### Controlador REST: `LiquidacionController.java`
```java
package co.edu.unimagdalena.avicontrol.infrastructure.adapter.rest;

import co.edu.unimagdalena.avicontrol.domain.port.in.dto.*;
import co.edu.unimagdalena.avicontrol.domain.port.in.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@RestController
@RequestMapping("/api/v1/liquidaciones")
public class LiquidacionController {

    private final PreviewLiquidacionUseCase previewUseCase;
    private final GenerarLiquidacionUseCase generarUseCase;
    private final ConsultarLiquidacionUseCase consultarUseCase;
    private final AnularLiquidacionUseCase anularUseCase;

    // Constructor inyección...

    /** CU03 Paso 1+2: generar vista previa sin persistir */
    @PostMapping("/previa")
    public ResponseEntity<LiquidacionPreviaDto> vistaPrevia(
            @RequestBody GenerarLiquidacionRequest req) {
        return ResponseEntity.ok(previewUseCase.calcularVistaPrevia(
            req.idLote(), req.idGalpon(), req.precioKgCop(), req.usuario()));
    }

    /** CU03 Paso 2 confirmado: persistir la Liquidación */
    @PostMapping
    public ResponseEntity<LiquidacionDto> generarLiquidacion(
            @RequestBody GenerarLiquidacionRequest req) {
        var dto = generarUseCase.generarLiquidacion(
            req.idLote(), req.idGalpon(), req.precioKgCop(), req.usuario());
        return ResponseEntity.created(URI.create("/api/v1/liquidaciones/" + dto.idLiquidacion()))
                .body(dto);
    }

    /** CU03.FR-014: consultar Liquidación existente (ACTIVA o ANULADA) */
    @GetMapping("/{id}")
    public ResponseEntity<LiquidacionDto> consultar(@PathVariable Long id) {
        return ResponseEntity.ok(consultarUseCase.consultarLiquidacion(id));
    }

    /** CU04: anulación (iniciada desde vista de Liquidación ACTIVA) */
    @PutMapping("/{id}/anulacion")
    public ResponseEntity<AnulacionDto> anular(
            @PathVariable Long id,
            @RequestBody AnulacionRequest req) {
        return ResponseEntity.ok(anularUseCase.anularLiquidacion(id, req.motivo(), req.usuario()));
    }
}
```

---

## Phase 1: Foundational — Esquema SQL de Liquidación y Dominio

- [ ] **T001** Crear migración Flyway `V3__create_liquidacion_tables.sql`.
- [ ] **T002** Crear modelos de dominio `Liquidacion`, `PartidaCostoLote`, `SnapshotDatosOrigen`, `RegistroAnulacion`.
- [ ] **T003** Crear entidades JPA y repositorios Spring Data.
- [ ] **T004** Implementar `Rounding.java` en `domain/shared/`.

---

## Phase 2: CU03 — Vista Previa (Paso 1+2, sin persistir)

- [ ] **T005** Unit Test `PreviewLiquidacionServiceTest.java`: verificar cálculos de Caso Dorado, Siniestro Total, rechazo de precio ≤ 0, validación de duplicado previo, y que **no se persiste nada**.
- [ ] **T006** Implementar `PreviewLiquidacionUseCase.java` y lógica en `GenerarLiquidacionService.java`.
- [ ] **T007** Controller Test e implementación de `POST /api/v1/liquidaciones/previa` (responde 200, no 201).

---

## Phase 3: CU03 — Confirmación (Persistir Liquidación)

- [ ] **T008** Unit Test `GenerarLiquidacionServiceTest.java`: Caso Dorado, Siniestro Total, prevención de duplicados `409 Conflict`, re-ejecución de validaciones en la confirmación.
- [ ] **T009** Implementar `GenerarLiquidacionUseCase.java` (persistencia real).
- [ ] **T010** Controller Test e implementación de `POST /api/v1/liquidaciones` (responde 201 Created).

---

## Phase 4: CU03.FR-014 — Consultar Liquidación Existente

- [ ] **T011** Unit Test `ConsultarLiquidacionServiceTest.java`: Liquidación `ACTIVA`, `ANULADA`, no encontrada (404).
- [ ] **T012** Implementar `ConsultarLiquidacionUseCase.java` y `ConsultarLiquidacionService.java`.
- [ ] **T013** Controller Test e implementación de `GET /api/v1/liquidaciones/{id}` con `accionesDisponibles`.

---

## Phase 5: CU04 — Anular Liquidación

- [ ] **T014** Unit Test `AnularLiquidacionServiceTest.java`: anulación exitosa con motivo, rechazo de motivo < 10 caracteres, rechazo de anulación sobre `ANULADA`, transacción atómica (rollback ante fallo).
- [ ] **T015** Implementar `AnularLiquidacionService.java` y `AnularLiquidacionUseCase.java`.
- [ ] **T016** Controller Test e implementación de `PUT /api/v1/liquidaciones/{id}/anulacion`.
