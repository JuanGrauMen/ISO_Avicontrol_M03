# Implementation Plan: Gestión del Ciclo de Vida de Liquidaciones

**Date**: 2026-09-28  
**Actualizado**: 2026-10-03  
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
    precio_kg_cop       NUMERIC(14,2),
    venta_bruta_cop     BIGINT NOT NULL,
    mortalidad_aves     INT NOT NULL,
    porcentaje_mortalidad NUMERIC(5,2) NOT NULL,
    costos_operativos_cop BIGINT NOT NULL,
    utilidad_neta_cop   BIGINT NOT NULL,
    estado              VARCHAR(10) NOT NULL DEFAULT 'ACTIVA',
    fecha_hora_generacion TIMESTAMP NOT NULL,
    usuario_responsable VARCHAR(100) NOT NULL,
    CONSTRAINT uq_lote_activa UNIQUE (id_lote)
);
```

---

## Snippets de Código Java de Puertos, DTOs y Servicios

### Puertos de Entrada: `GenerarLiquidacionUseCase.java` y `AnularLiquidacionUseCase.java`
```java
package co.edu.unimagdalena.avicontrol.domain.port.in;

import co.edu.unimagdalena.avicontrol.application.dto.AnulacionDto;
import co.edu.unimagdalena.avicontrol.application.dto.LiquidacionDto;

public interface GenerarLiquidacionUseCase {
    LiquidacionDto generarLiquidacion(UUID idGalpon, UUID idLote, BigDecimal precioKgCop, String usuario);
}

public interface AnularLiquidacionUseCase {
    AnulacionDto anularLiquidacion(Long idLiquidacion, String motivo, String usuario);
}
```

### Servicio de Generación: `GenerarLiquidacionService.java`
```java
package co.edu.unimagdalena.avicontrol.application.service;

import co.edu.unimagdalena.avicontrol.domain.model.Liquidacion;

@Service
public class GenerarLiquidacionService implements GenerarLiquidacionUseCase {

    @Override
    @Transactional
    public LiquidacionDto generarLiquidacion(UUID idGalpon, UUID idLote, BigDecimal precioKgCop, String usuario) {
        // 1. Validar que no exista otra liquidación ACTIVA sobre el mismo lote
        if (liquidacionRepository.existeActivaPorLote(idLote)) {
            throw new LiquidacionDuplicadaException(idLote);
        }

        // 2. Obtener datos de M1 y M2
        var sac = resultadoSacrificioRepository.buscarPorLote(idLote);
        var lote = loteRepository.buscarPorId(idLote);

        // 3. Cálculos matemáticos con HALF_UP
        BigDecimal ventaBrutaDecimal = sac.getPesoTotalKg().multiply(precioKgCop);
        long ventaBruta = Rounding.copTotal(ventaBrutaDecimal);
        long costosOperativos = calcularCostosOperativos(idLote);
        long utilidadNeta = ventaBruta - costosOperativos;

        // 4. Persistir entidad y snapshot
        var liq = Liquidacion.crearActiva(idLote, idGalpon, sac, precioKgCop, ventaBruta, costosOperativos, utilidadNeta, usuario);
        liquidacionRepository.guardar(liq);

        return LiquidacionMapper.toDto(liq);
    }
}
```

---

## Phase 1: Foundational – Esquema SQL de Liquidación y Dominio

- [ ] **T001** Crear migración Flyway `V3__create_liquidacion_tables.sql`.
- [ ] **T002** Crear modelos de dominio `Liquidacion`, `PartidaCostoLote`, `SnapshotDatosOrigen`, `RegistroAnulacion`.
- [ ] **T003** Crear entidades JPA y repositorios Spring Data.
- [ ] **T004** Implementar `Rounding.java` en `domain/shared/`.

---

## Phase 2: CU03 – Generar Liquidación del Lote

- [ ] **T005** Unit Test `GenerarLiquidacionServiceTest.java`: Caso Dorado, Siniestro Total y prevención de duplicados `409 Conflict`.
- [ ] **T006** Implementar `GenerarLiquidacionService.java` y `GenerarLiquidacionUseCase.java`.
- [ ] **T007** Controller Test e implementación de `POST /api/v1/liquidaciones`.

---

## Phase 3: CU04 – Anular Liquidación

- [ ] **T008** Unit Test `AnularLiquidacionServiceTest.java`: anulación exitosa con motivo y validación de 10-500 caracteres.
- [ ] **T009** Implementar `AnularLiquidacionService.java` y `AnularLiquidacionUseCase.java`.
- [ ] **T010** Controller Test e implementación de `PUT /api/v1/liquidaciones/{id}/anulacion`.
