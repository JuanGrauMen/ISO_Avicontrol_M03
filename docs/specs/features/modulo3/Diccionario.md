# Diccionario de Dominio y Datos — Módulo 3: Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)

**Versión**: 1.2  
**Fecha**: 03/10/2026  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad  
**Naturaleza**: Glosario Conceptual y Diccionario de Dominio / Base de Datos  

---

## Propósito

Este documento cumple la función dual de **Glosario Conceptual de Dominio (Domain-Driven Design)** y **Diccionario de Base de Datos Técnico**. Define los términos ubicuos del negocio utilizados en el Módulo 3 y los especifica con sus correspondientes atributos relacionales, tipos de datos SQL, llaves y restricciones de integridad.

---

## 1. Entidades Principales del Dominio

### A. Tabla `liquidacion` (Entidad `Liquidacion`)
Entidad central e inmutable del módulo que consolida los resultados económicos del cierre de un lote avícola.

| Campo | Tipo SQL | Nulo | Llave | Descripción y Regla de Negocio |
| :--- | :--- | :--- | :--- | :--- |
| `id_liquidacion` | `BIGSERIAL` | NO | PK | Identificador único interno autoincremental de la liquidación. |
| `id_lote` | `UUID` | NO | UQ | UUID del lote (M1). Restricción `UNIQUE` para liquidaciones `ACTIVA`. |
| `id_galpon` | `UUID` | NO | FK | UUID del galpón que albergó el lote (M1). |
| `id_resultado_sacrificio` | `UUID` | SI | FK | UUID del resultado de sacrificio (M2). Nulo en siniestro total. |
| `pollos_vendidos` | `INT` | SI | - | Cantidad total de aves comercializadas (de M2). Nulo si no se vendió. |
| `peso_total_kg` | `NUMERIC(12,2)` | SI | - | Peso acumulado total en kg (M2). Base de cálculo de Venta Bruta. |
| `peso_promedio_kg` | `NUMERIC(8,4)` | SI | - | Atributo derivado `peso_total_kg / pollos_vendidos`. Informativo. |
| `precio_kg_cop` | `NUMERIC(14,2)` | SI | - | Precio digitado en COP/kg por el Administrador Financiero. |
| `venta_bruta_cop` | `BIGINT` | NO | - | `peso_total_kg × precio_kg_cop` con redondeo `HALF_UP` a enteros COP. |
| `mortalidad_aves` | `INT` | NO | - | `poblacion_inicial - poblacion_actual` (de M1). |
| `porcentaje_mortalidad` | `NUMERIC(5,2)` | NO | - | `((mortalidad_aves / poblacion_inicial) × 100)` con 2 decimales. |
| `costos_operativos_cop` | `BIGINT` | NO | - | Suma de partidas directas (alimento + medicina + costo inicial lote). |
| `utilidad_neta_cop` | `BIGINT` | NO | - | Ganancia o pérdida neta `venta_bruta_cop - costos_operativos_cop`. |
| `estado` | `VARCHAR(10)` | NO | - | Estado del documento financiero (`ACTIVA` o `ANULADA`). |
| `fecha_hora_generacion` | `TIMESTAMP` | NO | - | Fecha y hora exacta de creación del registro. |
| `usuario_responsable` | `VARCHAR(100)` | NO | - | Identificador o correo del usuario financiero que procesó. |

---

### B. Tabla `galpon` (Entidad `Galpon` — Copia Local M1)
Reflejo local sincronizado de la infraestructura física desde Módulo 1.

| Campo | Tipo SQL | Nulo | Llave | Descripción y Regla de Negocio |
| :--- | :--- | :--- | :--- | :--- |
| `id_galpon` | `UUID` | NO | PK | Identificador global único del galpón (M1). |
| `nombre` | `VARCHAR(100)` | NO | - | Código o nombre asignado (ej. "Galpón 1"). |
| `aforo_maximo` | `INT` | NO | - | Capacidad máxima de aves autorizada. |
| `estado` | `VARCHAR(30)` | NO | - | Estado operativo sincronizado (`EstadoGalpon`). |
| `fecha_hora_sync` | `TIMESTAMP` | NO | - | Estampa de tiempo de la última ingesta desde M1. |

---

### C. Tabla `lote` (Entidad `Lote` — Copia Local M1)
Reflejo local sincronizado de la parvada alojada en un galpón desde Módulo 1.

| Campo | Tipo SQL | Nulo | Llave | Descripción y Regla de Negocio |
| :--- | :--- | :--- | :--- | :--- |
| `id_lote` | `UUID` | NO | PK | Identificador global único del lote (M1). |
| `id_galpon` | `UUID` | NO | FK | Referencia al galpón asignado en M1. |
| `nombre` | `VARCHAR(100)` | NO | - | Código del lote (ej. "Lote L-2026-A"). |
| `fecha_ingreso` | `DATE` | NO | - | Fecha de encasetamiento de la parvada. |
| `poblacion_inicial` | `INT` | NO | - | Cantidad inicial de aves ingresadas. |
| `poblacion_actual` | `INT` | NO | - | Aves vivas según el último reporte de M1. |
| `costo_total_cop` | `BIGINT` | NO | - | Costo inicial de adquisición del lote de pollitos. |
| `fecha_hora_sync` | `TIMESTAMP` | NO | - | Estampa de tiempo de la última ingesta desde M1. |

---

### D. Tabla `alerta_vaciado_sanitario` (Entidad `AlertaVaciadoSanitario` — Copia Local M1)
Evento registrado por Módulo 1 al concluir la cosecha y desvincular el lote del galpón. Es el pivote que permite a M3 identificar qué lote está listo para liquidar (`Por Liquidar`) y cuál ya fue liquidado (`Liquidado`). Sin esta alerta, M3 no puede derivar la etapa ni habilitar la generación de la Liquidación (CU01.FR-013, CU03.FR-005).

| Campo | Tipo SQL | Nulo | Llave | Descripción y Regla de Negocio |
| :--- | :--- | :--- | :--- | :--- |
| `id_alerta` | `UUID` | NO | PK | Identificador único de la alerta generado por M1. |
| `id_galpon` | `UUID` | NO | FK→galpon | Galpón que quedó en estado `Vaciado Sanitario`. |
| `id_lote` | `UUID` | NO | FK→lote | Lote desvinculado del galpón al cierre del ciclo productivo. |
| `fecha_hora_evento` | `TIMESTAMP` | NO | - | Momento exacto en que M1 registró el vaciado sanitario. |

> **Regla de negocio**: La alerta es inmutable una vez recibida de M1. Su presencia — combinada con la existencia o no de una Liquidación `ACTIVA` — determina la etapa del lote en la lista (CU01.FR-013). No se elimina aunque el lote sea liquidado.

---

### E. Tabla `partida_costo_lote` (Entidad `PartidaCostoLote`)
Detalle de las partidas de gasto congeladas asignadas a una liquidación.

| Campo | Tipo SQL | Nulo | Llave | Descripción y Regla de Negocio |
| :--- | :--- | :--- | :--- | :--- |
| `id_partida` | `BIGSERIAL` | NO | PK | Identificador único autoincremental de la partida. |
| `id_liquidacion` | `BIGINT` | NO | FK | Referencia a la liquidación asociada. |
| `id_lote` | `UUID` | NO | - | Referencia al lote evaluado. |
| `categoria` | `VARCHAR(20)` | NO | - | Clasificación (`ALIMENTO`, `MEDICINA`, `POBLACION`). |
| `concepto` | `VARCHAR(200)` | NO | - | Nombre del ítem (ej. "Alimento Engorde Fase 2"). |
| `cantidad` | `NUMERIC(12,3)` | NO | - | Cantidad consumida expresada en la unidad de medida. |
| `unidad_medida` | `VARCHAR(30)` | NO | - | Unidad de presentación (`KG`, `DOSIS`, `AVES`). |
| `precio_unitario_cop` | `NUMERIC(14,2)` | NO | - | Precio unitario aplicado. |
| `subtotal_cop` | `BIGINT` | NO | - | `cantidad × precio_unitario_cop` en entero COP. |
| `fuente_origen` | `VARCHAR(20)` | NO | - | Módulo emisor (`MODULO_1`, `MODULO_2`). |
| `referencia_origen` | `VARCHAR(200)` | SI | - | UUID o código de transacción origen. |

---

## 2. Entidades Secundarias y de Auditoría

### A. Tabla `registro_anulacion` (Entidad `RegistroAnulacion`)
Auditoría inalterable de cancelaciones de liquidaciones activas.

| Campo | Tipo SQL | Nulo | Llave | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `id_anulacion` | `BIGSERIAL` | NO | PK | Identificador único de la anulación. |
| `id_liquidacion` | `BIGINT` | NO | UQ/FK | Referencia a la liquidación anulada (1 a 1). |
| `motivo` | `VARCHAR(500)` | NO | - | Justificación escrita del usuario (10 a 500 caracteres). |
| `fecha_hora_anulacion` | `TIMESTAMP` | NO | - | Estampa de tiempo exacta del evento. |
| `usuario_responsable` | `VARCHAR(100)` | NO | - | Usuario autenticado que anuló. |

---

### B. Tabla `registro_sincronizacion` (Entidad `RegistroSincronizacion`)
Log de ejecuciones de ingesta periódica `@Scheduled` o por eventos Kafka.

| Campo | Tipo SQL | Nulo | Llave | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `BIGSERIAL` | NO | PK | Identificador único del registro de sync. |
| `fuente` | `VARCHAR(20)` | NO | - | Módulo consultado (`MODULO_1`, `MODULO_2`). |
| `fecha_hora_inicio` | `TIMESTAMP` | NO | - | Inicio del ciclo de ingesta. |
| `fecha_hora_fin` | `TIMESTAMP` | SI | - | Finalización del ciclo. |
| `resultado` | `VARCHAR(20)` | SI | - | `EXITOSA` o `FALLIDA`. |
| `descripcion_error` | `TEXT` | SI | - | Traza o detalle del fallo si ocurrió. |
| `registros_actualizados` | `INT` | NO | - | Filas creadas o modificadas (Default: 0). |

---

### C. Tabla `snapshot_datos_origen` (Entidad `SnapshotDatosOrigen`)
Captura inmutable de métricas maestras al momento de liquidar.

| Campo | Tipo SQL | Nulo | Llave | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `BIGSERIAL` | NO | PK | Identificador de la foto. |
| `id_liquidacion` | `BIGINT` | NO | FK | Liquidación asociada. |
| `fuente` | `VARCHAR(20)` | NO | - | `MODULO_1` o `MODULO_2`. |
| `fecha_hora_sync` | `TIMESTAMP` | NO | - | Hora de la sincronización usada. |
| `poblacion_inicial` | `INT` | SI | - | Población inicial congelada. |
| `poblacion_actual` | `INT` | SI | - | Población viva congelada. |
| `costo_total_lote_cop` | `BIGINT` | SI | - | Costo inicial del lote congelado. |

---

### D. Tabla `aviso_utilizacion_resultado` (Entidad `AvisoUtilizacionResultado`)
Notificación saliente hacia Módulo 2 indicando que el sacrificio fue consumido.

| Campo | Tipo SQL | Nulo | Llave | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `id_aviso` | `BIGSERIAL` | NO | PK | Identificador del aviso saliente. |
| `id_resultado` | `UUID` | NO | FK | Referencia al resultado final de sacrificio. |
| `id_liquidacion` | `BIGINT` | SI | - | Liquidación que lo consumió. |
| `fecha_hora_emision` | `TIMESTAMP` | NO | - | Fecha y hora de transmisión. |
| `estado_entrega` | `VARCHAR(20)` | NO | - | `PENDIENTE` o `ENTREGADO`. |
| `intentos` | `INT` | NO | - | Reintentos acumulados. |

---

## 3. Catálogos y Enumeraciones del Sistema

### `EstadoGalpon` (Enum)
- **`DISPONIBLE`**: Galpón sanitizado y listo para encasetar.
- **`VACIADO_SANITARIO`**: Aves retiradas; sanitización en curso. *(Requerido para liquidar)*.
- **`PRODUCTIVO`**: Aves en crecimiento activo.
- **`EN_COSECHA`**: Despacho de pollos en progreso.
- **`MANTENIMIENTO`**: Reparaciones locativas.
- **`AISLAMIENTO`**: Medida biosanitaria preventiva.

### `EstadoLiquidacion` (Enum)
- **`ACTIVA`**: Documento financiero oficial e inmutable.
- **`ANULADA`**: Cancelada formalmente por el usuario.

### `EstadoUtilizacion` (Enum)
- **`NO_UTILIZADO`**: Sacrificio disponible para liquidar.
- **`UTILIZADO`**: Sacrificio incorporado en una liquidación activa.

### `CategoriaCosto` (Enum)
- **`ALIMENTO`**: Concentrados y suplementos (M2).
- **`MEDICINA`**: Vacunas y tratamientos (M2).
- **`POBLACION`**: Adquisición inicial de la parvada (M1).

### `FuenteSincronizacion` (Enum)
- **`MODULO_1`**: Sistema de Galpones y Parvadas.
- **`MODULO_2`**: Sistema de Operaciones, Sacrificio e Insumos.

### `ResultadoSincronizacion` (Enum)
- **`EXITOSA`**: Ingesta procesada correctamente.
- **`FALLIDA`**: Error de comunicación o parseo.
