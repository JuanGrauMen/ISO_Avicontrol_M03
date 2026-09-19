# Feature Specification: M3-CU09 – Consultar Alimento del Lote al Módulo 2

**Created**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Actor Externo**: Módulo 2 – Gestión Operativa del Ciclo  
**Rol Principal**: Módulo 3 (proceso automático de sincronización)  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de las Partidas de Alimento del Ciclo (Priority: P1)

Como Módulo 3, quiero consultar a Módulo 2 las partidas de alimento asociadas al ciclo de un lote, con su cantidad en kilogramos y su precio unitario aplicable, para valorizar el Costo de Alimento de la Liquidación sin depender de la disponibilidad de Módulo 2 en el momento de liquidar.

**Why this priority**: El alimento es habitualmente la partida de costo más significativa del ciclo avícola. Sin ella, los Costos Operativos de M3-CU03 quedan incompletos y la Liquidación no puede generarse.

**Independent Test**: Se ejecuta la sincronización contra Módulo 2 para un lote con partidas de alimento registradas y se verifica que la copia local almacena, por cada partida, su tipo de alimento, cantidad en kilogramos, precio unitario por kilogramo y referencia de origen, sin calcular ningún total.

**Acceptance Scenarios**:

1. **Scenario**: Sincronización exitosa de partidas de alimento
   - **Given** Módulo 2 dispone de partidas de alimento asociadas al ciclo de un lote, con su tipo, cantidad en kilogramos y precio por kilogramo
   - **When** el proceso de sincronización se ejecuta
   - **Then** el sistema almacena por cada partida: `idPartidaOrigen`, `idLote`, `tipoAlimento`, `cantidadKg`, `precioUnitarioKgCop`, `referenciaOrigen` y la fecha y hora de sincronización

2. **Scenario**: Partida sin precio aplicable
   - **Given** Módulo 2 informa una partida de alimento sin precio unitario asociado
   - **When** se ejecuta la sincronización
   - **Then** el sistema almacena la partida marcándola como no valorizada y la expone como partida pendiente, de modo que M3-CU03 bloquee la Liquidación y liste la partida faltante

3. **Scenario**: Ausencia total de partidas de alimento para el ciclo
   - **Given** Módulo 2 no dispone de partidas de alimento asociadas al lote
   - **When** se ejecuta la sincronización
   - **Then** el sistema registra la ausencia, no genera partidas con valor cero y expone el estado para que M3-CU03 bloquee la Liquidación con el mensaje correspondiente

4. **Scenario**: Módulo 2 no responde con copia local previa existente
   - **Given** existe una copia local de las partidas de alimento y Módulo 2 no responde
   - **When** el administrador solicita generar la Liquidación
   - **Then** el sistema usa la copia local, conserva su fecha y hora de sincronización en el snapshot financiero y no realiza consultas en vivo

5. **Scenario**: Sincronización idempotente
   - **Given** una partida de alimento ya almacenada en la copia local
   - **When** la misma partida se obtiene nuevamente en una sincronización posterior
   - **Then** el sistema no crea un registro duplicado y actualiza únicamente la fecha y hora de sincronización

---

### Edge Cases

- **M3 no calcula en la sincronización**: Esta especificación obtiene y almacena cantidad y precio unitario. La multiplicación `cantidad × precio unitario` y la consolidación del Costo de Alimento son responsabilidad exclusiva de M3-CU03 (regla transversal RT-07).
- **Precios distintos para el mismo tipo de alimento**: Cuando una misma cantidad proviene de recepciones con precios diferentes, el sistema conserva cada tramo con su precio propio y **no promedia precios**. Esta separación es la que permite el desglose por línea de M3-CU05.
- **Impuestos**: Módulo 2 informa el valor neto y el impuesto por separado. M3 almacena ambos y no los consolida hasta que se defina su tratamiento.
- **Ausencia de registro de suministro real**: A la fecha de esta especificación, Módulo 2 no cuenta con una especificación que registre el alimento efectivamente suministrado a un lote, a diferencia de lo que sí ocurre con los medicamentos. Las únicas fuentes disponibles son las recepciones de compra y el requerimiento proyectado por etapa. Esta especificación se redacta en términos neutros —"partida de alimento"— para no comprometer el criterio de costeo antes de que se acuerde.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST consultar a Módulo 2, mediante un proceso automático en segundo plano con intervalo configurable, las partidas de alimento asociadas al ciclo de cada lote.
- **FR-002**: Por cada partida, el sistema MUST obtener y almacenar: `idPartidaOrigen`, `idLote`, `idGalpon`, `tipoAlimento`, `cantidadKg`, `precioUnitarioKgCop`, `valorImpuestoCop`, `referenciaOrigen` y la fecha y hora de sincronización.
- **FR-003**: El sistema NO DEBE calcular subtotales ni totales durante la sincronización. Almacena cantidad y precio unitario; la valorización corresponde a M3-CU03.
- **FR-004**: El sistema NO DEBE promediar precios entre partidas. Cada tramo de cantidad conserva el precio unitario que le corresponde.
- **FR-005**: El sistema MUST marcar como no valorizada toda partida que carezca de precio unitario aplicable, y MUST exponerla como partida pendiente para que M3-CU03 bloquee la Liquidación y la liste.
- **FR-006**: Ante la ausencia total de partidas de alimento para un lote, el sistema MUST registrar el hecho y NO DEBE generar partidas con valor cero.
- **FR-007**: Ante una respuesta fallida o ausente de Módulo 2, el sistema MUST conservar íntegra la copia local previa, registrar el fallo y reintentar en el siguiente ciclo, sin bloquear ninguna operación de M3.
- **FR-008**: El sistema MUST procesar las sincronizaciones de forma idempotente: obtener la misma partida más de una vez no DEBE crear registros duplicados.
- **FR-009**: El sistema NO DEBE escribir, modificar ni eliminar información de alimento en Módulo 2. La relación es exclusivamente de lectura.

[NEEDS CLARIFICATION – D-01: el criterio de costeo del alimento está pendiente de acuerdo con el equipo de Módulo 2. Existen tres criterios en juego, mutuamente excluyentes:
  1. **Compras del ciclo, sin descontar sobrantes** — criterio de la versión anterior de M3-CU03. Simple, pero castiga el sobrestock del ciclo.
  2. **Consumo real aplicado al lote, valorizado al precio histórico de cada recepción** — criterio declarado por Módulo 2 para alimento y medicamentos. Es el contablemente correcto, pero **Módulo 2 no cuenta hoy con una especificación que registre el suministro real de alimento**, por lo que el dato no existe.
  3. **Requerimiento proyectado por etapa, valorizado al precio unitario vigente** — criterio de la consulta que Módulo 2 declara como su punto de integración oficial con Módulo 3. Está disponible hoy, pero liquida sobre alimento estimado, no gastado, y emplea precio vigente en lugar de histórico.
  
  Mientras la decisión no se tome, `cantidadKg` y `precioUnitarioKgCop` se definen en términos neutros y la elección de la fuente no altera el resto de esta especificación.]

[NEEDS CLARIFICATION – D-05: tratamiento del impuesto informado por Módulo 2. Debe definirse si los Costos Operativos de M3-CU03 se calculan sobre valor neto o con impuesto incluido.]

### Key Entities

- **PartidaAlimentoLote**: Réplica local de una partida de alimento del ciclo. Atributos: `idPartidaOrigen`, `idLote`, `idGalpon`, `tipoAlimento`, `cantidadKg`, `precioUnitarioKgCop`, `valorImpuestoCop`, `estadoValorizacion` (`VALORIZADA` | `PENDIENTE_PRECIO`), `referenciaOrigen`, `fechaHoraSincronizacion`.
- **RegistroSincronizacion**: Bitácora técnica del proceso, compartida con M3-CU07, M3-CU08 y M3-CU10. Atributos: `idSincronizacion`, `fuente` (`MODULO_2`), `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las partidas de alimento disponibles en Módulo 2 para un lote se refleja en la copia local tras una sincronización exitosa.
- **SC-002**: 0% de precios promediados entre partidas con precios distintos.
- **SC-003**: El 100% de las partidas sin precio aplicable se expone como pendiente y bloquea la Liquidación en M3-CU03.
- **SC-004**: 0% de subtotales o totales calculados durante la sincronización.
- **SC-005**: 0% de registros duplicados ante sincronizaciones repetidas de la misma partida.
- **SC-006**: 0% de escrituras de M3 sobre los datos de alimento de Módulo 2.
