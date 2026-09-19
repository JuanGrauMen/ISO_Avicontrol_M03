# Feature Specification: M3-CU10 – Consultar Medicina Consumida al Módulo 2

**Created**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Actor Externo**: Módulo 2 – Gestión Operativa del Ciclo  
**Rol Principal**: Módulo 3 (proceso automático de sincronización)  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de los Consumos de Medicamento del Ciclo (Priority: P1)

Como Módulo 3, quiero consultar a Módulo 2 los consumos de medicamento aplicados a un lote, con su cantidad en unidad base y el desglose por recepción con su precio histórico, para valorizar los Insumos Médicos de la Liquidación sin depender de la disponibilidad de Módulo 2 en el momento de liquidar.

**Why this priority**: Es una de las tres partidas que componen los Costos Operativos de M3-CU03. A diferencia del alimento, Módulo 2 ya registra el consumo real aplicado al lote, por lo que esta integración puede especificarse por completo.

**Independent Test**: Se ejecuta la sincronización contra Módulo 2 para un lote con consumos de medicamento registrados y se verifica que la copia local almacena, por cada consumo, el medicamento, la cantidad en unidad base y un tramo por cada recepción utilizada con su precio histórico, sin promediar precios y sin calcular totales.

**Acceptance Scenarios**:

1. **Scenario**: Sincronización exitosa de un consumo abastecido por una sola recepción
   - **Given** Módulo 2 registró un consumo de medicamento aplicado a un lote, abastecido íntegramente por una recepción
   - **When** el proceso de sincronización se ejecuta
   - **Then** el sistema almacena `idConsumoOrigen`, `idLote`, `idGalpon`, `medicamento`, `fechaConsumo`, `cantidadUnidadBase`, `unidadBase`, el precio histórico por unidad base de la recepción, la referencia de la recepción y la fecha y hora de sincronización

2. **Scenario**: Consumo abastecido por recepciones con precios distintos
   - **Given** un consumo de medicamento que Módulo 2 abasteció desde dos recepciones con precios por unidad base diferentes
   - **When** se ejecuta la sincronización
   - **Then** el sistema almacena un tramo por cada recepción, cada uno con su cantidad y su precio histórico propio, y NO promedia los precios entre tramos

3. **Scenario**: Consumo sin precio histórico disponible
   - **Given** Módulo 2 informa un consumo cuya recepción de origen carece de precio por unidad base
   - **When** se ejecuta la sincronización
   - **Then** el sistema almacena el tramo marcándolo como no valorizado y lo expone como partida pendiente, de modo que M3-CU03 bloquee la Liquidación y liste la partida faltante

4. **Scenario**: Ausencia total de consumos de medicamento para el ciclo
   - **Given** Módulo 2 no registra consumos de medicamento para el lote
   - **When** se ejecuta la sincronización
   - **Then** el sistema registra la ausencia y no genera partidas con valor cero. Un lote sin medicación es un escenario válido y no bloquea por sí solo la Liquidación

5. **Scenario**: Módulo 2 no responde con copia local previa existente
   - **Given** existe una copia local de los consumos de medicamento y Módulo 2 no responde
   - **When** el administrador solicita generar la Liquidación
   - **Then** el sistema usa la copia local, conserva su fecha y hora de sincronización en el snapshot financiero y no realiza consultas en vivo

6. **Scenario**: Sincronización idempotente
   - **Given** un consumo de medicamento ya almacenado en la copia local
   - **When** el mismo consumo se obtiene nuevamente en una sincronización posterior
   - **Then** el sistema no crea un registro duplicado y actualiza únicamente la fecha y hora de sincronización

---

### Edge Cases

- **Unidad base como unidad de valorización**: Los medicamentos se consumen en unidad base (gramos, mililitros o unidades) y se valorizan al precio por unidad base de la recepción correspondiente. M3 no convierte entre presentaciones comerciales y unidad base; recibe ambas cifras ya resueltas por Módulo 2.
- **Prohibición de promediar precios**: Cuando un consumo se abastece de varias recepciones, cada tramo conserva su precio histórico. El promedio destruiría la trazabilidad exigida por CA-G05 y haría imposible el desglose por línea de M3-CU05.
- **Lote sin medicación**: Es un escenario operativo legítimo. La ausencia de consumos de medicamento no bloquea la Liquidación, a diferencia de la ausencia total de partidas de alimento.
- **Impuestos**: Módulo 2 informa el valor neto y el impuesto por separado. M3 almacena ambos y no los consolida hasta que se defina su tratamiento.
- **M3 no calcula en la sincronización**: La multiplicación `cantidad × precio unitario` y la consolidación de los Insumos Médicos son responsabilidad exclusiva de M3-CU03 (regla transversal RT-07).

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST consultar a Módulo 2, mediante un proceso automático en segundo plano con intervalo configurable, los consumos de medicamento aplicados a cada lote.
- **FR-002**: Por cada consumo, el sistema MUST obtener y almacenar: `idConsumoOrigen`, `idLote`, `idGalpon`, `medicamento`, `fechaConsumo`, `cantidadUnidadBase` y `unidadBase`.
- **FR-003**: Por cada consumo, el sistema MUST obtener y almacenar el **desglose por recepción**: un tramo por cada recepción utilizada, con su cantidad, su precio histórico por unidad base, su valor de impuesto y su referencia de origen.
- **FR-004**: El sistema NO DEBE promediar precios entre tramos de un mismo consumo. Cada tramo conserva el precio histórico de su recepción.
- **FR-005**: El sistema NO DEBE calcular subtotales ni totales durante la sincronización. Almacena cantidad y precio unitario; la valorización corresponde a M3-CU03.
- **FR-006**: El sistema MUST marcar como no valorizado todo tramo cuya recepción carezca de precio por unidad base, y MUST exponerlo como partida pendiente para que M3-CU03 bloquee la Liquidación y lo liste.
- **FR-007**: Ante la ausencia total de consumos de medicamento para un lote, el sistema MUST registrar el hecho, NO DEBE generar partidas con valor cero y NO DEBE bloquear por sí solo la Liquidación.
- **FR-008**: Ante una respuesta fallida o ausente de Módulo 2, el sistema MUST conservar íntegra la copia local previa, registrar el fallo y reintentar en el siguiente ciclo, sin bloquear ninguna operación de M3.
- **FR-009**: El sistema MUST procesar las sincronizaciones de forma idempotente: obtener el mismo consumo más de una vez no DEBE crear registros duplicados.
- **FR-010**: El sistema NO DEBE escribir, modificar ni eliminar información de medicamentos en Módulo 2. La relación es exclusivamente de lectura.

[NEEDS CLARIFICATION – D-05: tratamiento del impuesto informado por Módulo 2. Debe definirse si los Costos Operativos de M3-CU03 se calculan sobre valor neto o con impuesto incluido.]

### Key Entities

- **ConsumoMedicamentoLote**: Réplica local de un consumo de medicamento aplicado al lote. Atributos: `idConsumoOrigen`, `idLote`, `idGalpon`, `medicamento`, `fechaConsumo`, `cantidadUnidadBase`, `unidadBase`, `fechaHoraSincronizacion`.
- **TramoRecepcionConsumo**: Porción de un consumo abastecida por una recepción específica, con su precio histórico. Atributos: `idTramo`, `idConsumoOrigen`, `referenciaRecepcion`, `cantidadUnidadBase`, `precioUnitarioBaseCop`, `valorImpuestoCop`, `estadoValorizacion` (`VALORIZADO` | `PENDIENTE_PRECIO`).
- **RegistroSincronizacion**: Bitácora técnica del proceso, compartida con M3-CU07, M3-CU08 y M3-CU09. Atributos: `idSincronizacion`, `fuente` (`MODULO_2`), `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los consumos de medicamento disponibles en Módulo 2 para un lote se refleja en la copia local tras una sincronización exitosa.
- **SC-002**: 0% de precios promediados entre tramos de recepciones distintas.
- **SC-003**: El 100% de los tramos sin precio histórico aplicable se expone como pendiente y bloquea la Liquidación en M3-CU03.
- **SC-004**: 0% de subtotales o totales calculados durante la sincronización.
- **SC-005**: 0% de registros duplicados ante sincronizaciones repetidas del mismo consumo.
- **SC-006**: 0% de escrituras de M3 sobre los datos de medicamentos de Módulo 2.
