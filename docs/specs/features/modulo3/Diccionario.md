# Diccionario de Datos — Módulo 3: Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)

**Versión**: 1.0  
**Fecha**: 02/10/2026  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad  

---

## 1. Entidades Principales del Dominio

### A. Liquidación (`Liquidacion`)
Entidad central e inmutable del módulo que consolida los resultados económicos del cierre de un lote avícola.
- **`idLiquidacion`** (`BIGINT`, PK, Auto-incremental): Identificador único interno del registro de liquidación.
- **`idLote`** (`UUID`, NOT NULL): Identificador del lote de aves liquidado (proveniente de M1). Restricción de unicidad para liquidaciones activas (`uq_lote_activa`).
- **`idGalpon`** (`UUID`, NOT NULL): Identificador del galpón que albergó el lote (proveniente de M1).
- **`idResultadoSacrificio`** (`UUID`, NULLABLE): Referencia al resultado final de sacrificio emitido por M2. Nulo únicamente en caso de siniestro total (100% mortalidad).
- **`pollosVendidos`** (`INT`, NULLABLE): Cantidad total de aves comercializadas (de M2).
- **`pesoTotalKg`** (`NUMERIC(12,2)`, NULLABLE): Peso acumulado total comercializado en kilogramos (de M2). Base de cálculo de la Venta Bruta.
- **`pesoPromedioKg`** (`NUMERIC(8,4)`, NULLABLE): Peso promedio por ave (`pesoTotalKg / pollosVendidos`). Atributo puramente informativo para presentación.
- **`precioKgCop`** (`NUMERIC(14,2)`, NULLABLE): Precio de venta por kilogramo ingresado por el Administrador Financiero.
- **`ventaBrutaCop`** (`BIGINT`, NOT NULL): Ingreso total generado (`pesoTotalKg × precioKgCop`). Redondeado con `HALF_UP` a enteros COP.
- **`mortalidadAves`** (`INT`, NOT NULL): Cantidad de aves muertas en el ciclo (`poblacionInicial - poblacionActual` de M1).
- **`porcentajeMortalidad`** (`NUMERIC(5,2)`, NOT NULL): Porcentaje acumulado de mortalidad `((mortalidadAves / poblacionInicial) × 100)`. Expresado con 2 decimales.
- **`costosOperativosCop`** (`BIGINT`, NOT NULL): Suma total de los costos directos del ciclo (alimento, medicina y costo inicial de poblacion).
- **`utilidadNetaCop`** (`BIGINT`, NOT NULL): Ganancia o pérdida neta del lote (`ventaBrutaCop - costosOperativosCop`).
- **`estado`** (`VARCHAR(10)`, NOT NULL): Estado del documento financiero (`ACTIVA` o `ANULADA`).
- **`fechaHoraGeneracion`** (`TIMESTAMP`, NOT NULL): Fecha y hora exacta de creación del registro.
- **`usuarioResponsable`** (`VARCHAR(100)`, NOT NULL): Identificador o correo del usuario que procesó la liquidación.

### B. Galpón (`Galpon`) — Copia Local M1
Registro local que refleja la infraestructura física y su estado operativo sincronizado desde Módulo 1.
- **`idGalpon`** (`UUID`, PK): Identificador único del galpón en el sistema global.
- **`nombre`** (`VARCHAR(100)`, NOT NULL): Nombre o código identificador del galpón (ej. "Galpón 1").
- **`aforoMaximo`** (`INT`, NOT NULL): Capacidad máxima poblacional autorizada.
- **`estado`** (`VARCHAR(30)`, NOT NULL): Estado operativo sincronizado de M1 (`EstadoGalpon`).
- **`fechaHoraSync`** (`TIMESTAMP`, NOT NULL): Estampa de tiempo de la última sincronización con M1.

### C. Lote (`Lote`) — Copia Local M1
Registro local de la parvada alojada en un galpón, sincronizado desde Módulo 1.
- **`idLote`** (`UUID`, PK): Identificador único del lote en el sistema global.
- **`idGalpon`** (`UUID`, NOT NULL, FK): Referencia al galpón al que está asignado.
- **`nombre`** (`VARCHAR(100)`, NOT NULL): Nombre o código del lote (ej. "Lote L-2026-A").
- **`fechaIngreso`** (`DATE`, NOT NULL): Fecha de encasetamiento o inicio del ciclo.
- **`poblacionInicial`** (`INT`, NOT NULL): Cantidad de aves con las que inició el ciclo.
- **`poblacionActual`** (`INT`, NOT NULL): Cantidad de aves vivas reportadas en la última lectura de M1.
- **`costoTotalCop`** (`BIGINT`, NOT NULL): Costo inicial acumulado de adquisición del lote de pollitos.
- **`fechaHoraSync`** (`TIMESTAMP`, NOT NULL): Estampa de tiempo de la última sincronización con M1.

### D. Partida de Costo del Lote (`PartidaCostoLote`)
Detalle congelado de las partidas de costo directo asignadas a una liquidación.
- **`idPartida`** (`BIGSERIAL`, PK): Identificador interno autogenerado.
- **`idLiquidacion`** (`BIGINT`, NOT NULL, FK): Referencia a la liquidación correspondiente.
- **`idLote`** (`UUID`, NOT NULL): Referencia al lote de origen.
- **`categoria`** (`VARCHAR(20)`, NOT NULL): Clasificación del costo (`ALIMENTO`, `MEDICINA`, `POBLACION`).
- **`concepto`** (`VARCHAR(200)`, NOT NULL): Descripción detallada del ítem (ej. "Alimento Engorde Fase 2").
- **`cantidad`** (`NUMERIC(12,3)`, NOT NULL): Cantidad consumida expresada en la unidad de medida.
- **`unidadMedida`** (`VARCHAR(30)`, NOT NULL): Unidad de empaque o medida (ej. "KG", "DOSIS", "AVES").
- **`precioUnitarioCop`** (`NUMERIC(14,2)`, NOT NULL): Precio unitario COP aplicado a la partida.
- **`subtotalCop`** (`BIGINT`, NOT NULL): Subtotal monetario (`cantidad × precioUnitarioCop`) redondeado a entero COP.
- **`fuenteOrigen`** (`VARCHAR(20)`, NOT NULL): Módulo emisor original (`MODULO_1`, `MODULO_2`).
- **`referenciaOrigen`** (`VARCHAR(200)`, NULLABLE): UUID o código de transacción del registro origen.

---

## 2. Entidades Secundarias y de Auditoría

### A. Registro de Anulación (`RegistroAnulacion`)
Auditoría inalterable que documenta la cancelación formal de una liquidación activa.
- **`idAnulacion`** (`BIGSERIAL`, PK): Identificador del evento de anulación.
- **`idLiquidacion`** (`BIGINT`, NOT NULL, UNIQUE, FK): Referencia a la liquidación anulada.
- **`motivo`** (`VARCHAR(500)`, NOT NULL): Explicación justificada del usuario (entre 10 y 500 caracteres).
- **`fechaHoraAnulacion`** (`TIMESTAMP`, NOT NULL): Estampa de tiempo del evento.
- **`usuarioResponsable`** (`VARCHAR(100)`, NOT NULL): Usuario autenticado que ejecutó la anulación.

### B. Bitácora de Sincronización (`RegistroSincronizacion`)
Trazabilidad de las ejecuciones periódicas `@Scheduled` o por evento Kafka.
- **`id`** (`BIGSERIAL`, PK): Identificador único del log.
- **`fuente`** (`VARCHAR(20)`, NOT NULL): Módulo fuente consultado (`MODULO_1`, `MODULO_2`).
- **`fechaHoraInicio`** (`TIMESTAMP`, NOT NULL): Hora de inicio del proceso de sincronización.
- **`fechaHoraFin`** (`TIMESTAMP`, NULLABLE): Hora de finalización.
- **`resultado`** (`VARCHAR(20)`, NULLABLE): Estado final (`EXITOSA`, `FALLIDA`).
- **`descripcionError`** (`TEXT`, NULLABLE): Detalle técnico del fallo si aplica.
- **`registrosActualizados`** (`INT`, DEFAULT 0): Conteo de filas creadas o actualizadas.

### C. Snapshot de Datos de Origen (`SnapshotDatosOrigen`)
Fotografía inmutable de las cifras maestras de M1 y M2 al momento exacto de generar la liquidación.
- **`id`** (`BIGSERIAL`, PK): Identificador interno.
- **`idLiquidacion`** (`BIGINT`, NOT NULL, FK): Referencia a la liquidación congelada.
- **`fuente`** (`VARCHAR(20)`, NOT NULL): `MODULO_1` o `MODULO_2`.
- **`fechaHoraSync`** (`TIMESTAMP`, NOT NULL): Estampa de tiempo de los datos sincronizados utilizados.
- **`poblacionInicial`** (`INT`, NULLABLE): Población inicial registrada en la foto.
- **`poblacionActual`** (`INT`, NULLABLE): Población viva al momento de la foto.
- **`costoTotalLoteCop`** (`BIGINT`, NULLABLE): Costo total del lote en la foto.

### D. Aviso de Utilización de Resultado (`AvisoUtilizacionResultado`)
Registro de entrega saliente hacia Módulo 2 notificando que el resultado de sacrificio fue consumido.
- **`idAviso`** (`BIGSERIAL`, PK): Identificador interno.
- **`idResultado`** (`UUID`, NOT NULL, FK): UUID del resultado final de sacrificio de M2.
- **`idLiquidacion`** (`BIGINT`, NULLABLE): ID de la liquidación que lo utilizó.
- **`fechaHoraEmision`** (`TIMESTAMP`, NOT NULL): Estampa de tiempo de envío.
- **`estadoEntrega`** (`VARCHAR(20)`, NOT NULL): `PENDIENTE` o `ENTREGADO`.
- **`intentos`** (`INT`, NOT NULL, DEFAULT 0): Conteo de reintentos realizados.

---

## 3. Catálogos y Enumeraciones del Sistema

### `EstadoGalpon` (Enum)
Catálogo compartido de estados operativos de un galpón (sincronizado desde M1):
1. **`DISPONIBLE`**: Galpón limpio, desinfectado y listo para recibir un nuevo lote.
2. **`VACIADO_SANITARIO`**: Aves retiradas completamente; periodo de descanso y sanitización. Estado requerido para habilitar la liquidación.
3. **`PRODUCTIVO`**: Aves en etapa de engorde y crecimiento activo.
4. **`EN_COSECHA`**: Proceso de retiro y despacho de aves hacia centro de sacrificio.
5. **`MANTENIMIENTO`**: Galpón fuera de servicio por reparaciones locativas o de equipo.
6. **`AISLAMIENTO`**: Galpón bajo cuarentena o medida biosanitaria preventiva.

### `EstadoLiquidacion` (Enum)
Ciclo de vida del documento financiero de liquidación en M3:
1. **`ACTIVA`**: Liquidación vigente, inmutable y oficial del lote.
2. **`ANULADA`**: Liquidación anulada por error o corrección. No altera los indicadores activos.

### `EstadoUtilizacion` (Enum)
Estado del resultado final de sacrificio recibido de M2:
1. **`NO_UTILIZADO`**: Resultado sincronizado disponible para ser incorporado en una liquidación.
2. **`UTILIZADO`**: Resultado ya consumido por una liquidación activa.

### `CategoriaCosto` (Enum)
Clasificación de las partidas de gasto en la liquidación:
1. **`ALIMENTO`**: Partidas de concentrado y suplementos (origen M2).
2. **`MEDICINA`**: Consumos de fármacos, vacunas y tratamientos (origen M2).
3. **`POBLACION`**: Costo inicial de adquisición de la parvada (origen M1).

### `FuenteSincronizacion` (Enum)
Módulos externos de origen:
1. **`MODULO_1`**: Módulo de Gestión de Galpones y Parvadas.
2. **`MODULO_2`**: Módulo de Operaciones, Sacrificio e Insumos.

### `ResultadoSincronizacion` (Enum)
1. **`EXITOSA`**: Proceso de ingesta completado sin errores.
2. **`FALLIDA`**: Ocurrió un fallo de conexión, timeout o parseo.
