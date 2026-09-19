# Feature Specification: M3-CU07 – Consultar Población y Costo del Lote al Módulo 1

**Created**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Actor Externo**: Módulo 1 – Gestión de Galpones y Lotes  
**Rol Principal**: Módulo 3 (proceso automático de sincronización)  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de Galpones, Lotes y Alertas de Vaciado (Priority: P1)

Como Módulo 3, quiero consultar periódicamente a Módulo 1 el estado de los galpones, los datos de sus lotes —incluidos población inicial, población actual y costo total— y las alertas de vaciado sanitario registradas, para mantener una copia local que sostenga todas mis operaciones financieras sin depender de la disponibilidad de Módulo 1 en el momento de operar.

**Why this priority**: Es el insumo de los tres cálculos que Módulo 1 alimenta en la Liquidación: la mortalidad del lote, el Costo de Población y la precondición de vaciado sanitario que habilita liquidar. Sin esta sincronización, M3-CU01 no tiene qué listar y M3-CU03 no puede calcular.

**Independent Test**: Se ejecuta el proceso de sincronización contra Módulo 1 y se verifica que la copia local queda poblada con los galpones, sus lotes activos, las alertas de vaciado sanitario y la fecha y hora de la consulta. Se detiene Módulo 1 y se comprueba que M3 conserva la copia anterior, registra el fallo y reintenta sin bloquear ninguna operación.

**Acceptance Scenarios**:

1. **Scenario**: Sincronización inicial exitosa
   - **Given** Módulo 1 responde y M3 no tiene copia local previa
   - **When** el proceso de sincronización se ejecuta
   - **Then** el sistema almacena por cada galpón su UUID, nombre, aforo máximo y estado operativo; por cada lote activo su UUID, nombre, fecha de ingreso, población inicial, población actual y costo total en COP; y registra la fecha y hora de la consulta

2. **Scenario**: Sincronización incremental con cambios de población
   - **Given** existe una copia local previa y Módulo 1 ha actualizado la población actual de un lote por mortalidad reportada desde Módulo 2
   - **When** se ejecuta la siguiente sincronización
   - **Then** el sistema actualiza la población actual en la copia local, conserva la población inicial sin alteración y actualiza la fecha y hora de sincronización

3. **Scenario**: Sincronización de alertas de vaciado sanitario
   - **Given** Módulo 1 ha aceptado una alerta de vaciado sanitario y desvinculado el lote de su galpón
   - **When** se ejecuta la sincronización
   - **Then** el sistema almacena la alerta con `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento`, y conserva los datos del lote desvinculado para que siga siendo identificable y liquidable

4. **Scenario**: Módulo 1 no responde con copia local previa existente
   - **Given** existe una copia local sincronizada y Módulo 1 no responde o retorna error
   - **When** se ejecuta la sincronización
   - **Then** el sistema conserva íntegra la copia local anterior, registra el fallo con su fecha y hora, no marca los datos como inválidos y reintenta en el siguiente ciclo sin bloquear ninguna operación de M3

5. **Scenario**: Módulo 1 no responde sin copia local previa
   - **Given** no existe copia local para un galpón o lote y Módulo 1 no responde
   - **When** el usuario intenta operar sobre ese lote
   - **Then** el sistema bloquea la operación e informa que no hay datos sincronizados disponibles para ese lote

6. **Scenario**: Sincronización idempotente
   - **Given** una alerta de vaciado sanitario ya almacenada en la copia local
   - **When** la misma alerta se obtiene nuevamente en una sincronización posterior
   - **Then** el sistema no crea un registro duplicado y actualiza únicamente la fecha y hora de sincronización

---

### Edge Cases

- **Población actual mayor a cero en `Vaciado Sanitario`**: Módulo 1 no exige población cero para transicionar a `Vaciado sanitario`. La diferencia entre población inicial y población actual representa aves muertas, no aves vendidas. M3 almacena ambos valores tal como los informa Módulo 1 y no los interpreta durante la sincronización.
- **Costo total inmutable**: El costo total del lote se captura en Módulo 1 al registrarlo y es un entero positivo en COP. M3 lo almacena sin transformarlo y lo usa como Costo de Población en M3-CU03.
- **Datos parciales**: Si Módulo 1 responde con un galpón sin lote activo, M3 almacena el galpón y deja vacíos los campos de lote. No se infiere ni se completa ningún valor ausente.
- **Ausencia de contrato formal**: A la fecha de esta especificación, ninguna especificación de Módulo 1 declara la exposición de estos datos a Módulo 3. Esta especificación documenta lo que M3 necesita consultar; el compromiso correspondiente debe quedar registrado del lado de Módulo 1.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST consultar a Módulo 1, mediante un proceso automático en segundo plano con intervalo configurable, los datos de galpones, lotes y alertas de vaciado sanitario.
- **FR-002**: Por cada galpón, el sistema MUST obtener y almacenar: `idGalpon`, `nombre`, `aforoMaximo` y `estado` operativo.
- **FR-003**: Por cada lote activo, el sistema MUST obtener y almacenar: `idLote`, `idGalpon`, `nombre`, `fechaIngreso`, `poblacionInicial`, `poblacionActual` y `costoTotalCop`.
- **FR-004**: Por cada alerta de vaciado sanitario aceptada, el sistema MUST obtener y almacenar: `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento`, conservándolos aun después de que Módulo 1 desvincule el lote del galpón.
- **FR-005**: El sistema MUST registrar, por cada sincronización, su fecha y hora y su resultado (exitosa o fallida), y MUST exponer la fecha y hora de la última sincronización exitosa a las especificaciones que consumen la copia local.
- **FR-006**: Ante una respuesta fallida o ausente de Módulo 1, el sistema MUST conservar íntegra la copia local previa, registrar el fallo y reintentar en el siguiente ciclo, sin bloquear ninguna operación de M3 ni afectar a los demás módulos.
- **FR-007**: El sistema MUST procesar las sincronizaciones de forma idempotente: obtener el mismo dato más de una vez no DEBE crear registros duplicados.
- **FR-008**: El sistema NO DEBE escribir, modificar ni eliminar información en Módulo 1 bajo ninguna circunstancia. La relación con Módulo 1 es exclusivamente de lectura.
- **FR-009**: El sistema NO DEBE transformar, redondear ni interpretar los valores obtenidos. La población inicial, la población actual y el costo total se almacenan tal como los informa Módulo 1.
- **FR-010**: El sistema MUST rechazar y registrar como incidencia los datos que lleguen incompletos o inconsistentes (lote sin población inicial, costo total no positivo, alerta sin UUID de lote), sin sobrescribir con ellos la copia local válida previa.
- **FR-011**: El sistema MUST admitir el catálogo de estados operativos definido en M3-CU01.FR-003 y MUST registrar como incidencia cualquier estado recibido que no pertenezca a ese catálogo.

[NEEDS CLARIFICATION – D-02: Módulo 1 no cuenta actualmente con requisitos funcionales que declaren la exposición de población inicial, población actual, costo total del lote, estado del galpón ni alerta de vaciado sanitario hacia Módulo 3. El mecanismo de consulta, los campos y los tiempos de respuesta deben acordarse con el equipo de Módulo 1.]

[NEEDS CLARIFICATION – D-04: Módulo 1 acepta alertas de vaciado sanitario únicamente cuando el galpón está en estado `En cosecha`, y no inicia el vaciado cuando la población actual llega a cero. Un lote con mortalidad total en estado `Productivo` no alcanza el estado que habilita la Liquidación en M3-CU03. Ruta de estado pendiente de acuerdo.]

[NEEDS CLARIFICATION – D-06: Módulo 1 acepta alertas de mortalidad únicamente cuando el galpón está en estado `Productivo`. La mortalidad ocurrida durante `En cosecha` o `Aislamiento` no reduce la población actual, lo que subestima la mortalidad calculada en M3-CU03. Tratamiento pendiente de acuerdo.]

### Key Entities

- **CopiaLocalGalponLote**: Réplica local de los galpones y sus lotes activos, mantenida por esta especificación. Atributos: los definidos en FR-002 y FR-003, más `fechaHoraUltimaSincronizacion` y `estadoUltimaSincronizacion`.
- **AlertaVaciadoSanitario**: Réplica local de la alerta aceptada por Módulo 1. Atributos: `idAlerta`, `idGalpon`, `idLote`, `fechaHoraEvento`, `fechaHoraSincronizacion`.
- **RegistroSincronizacion**: Bitácora técnica del proceso. Atributos: `idSincronizacion`, `fuente` (`MODULO_1`), `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los galpones, lotes y alertas de vaciado sanitario disponibles en Módulo 1 se refleja en la copia local tras una sincronización exitosa.
- **SC-002**: El 100% de las operaciones de M3 se resuelve con la copia local, sin consultas en vivo a Módulo 1.
- **SC-003**: 0% de pérdidas de la copia local previa ante fallos de Módulo 1.
- **SC-004**: 0% de registros duplicados ante sincronizaciones repetidas del mismo dato.
- **SC-005**: 0% de escrituras de M3 sobre Módulo 1.
- **SC-006**: El 100% de las sincronizaciones queda registrado con fecha, hora y resultado.
