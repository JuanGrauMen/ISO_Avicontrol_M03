# Feature Specification: M3-CU08 – Consultar Resultado Final de Sacrificio al Módulo 2

**Created**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Actor Externo**: Módulo 2 – Gestión Operativa del Ciclo  
**Rol Principal**: Módulo 3 (proceso automático de sincronización)  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización del Resultado Final de Sacrificio (Priority: P1)

Como Módulo 3, quiero consultar a Módulo 2 el resultado final de sacrificio de cada lote —cantidad final de pollos y peso total en kilogramos— para disponer de la base física sobre la cual el administrador financiero registra la Matriz de Ventas, sin depender de la disponibilidad de Módulo 2 en el momento de registrar.

**Why this priority**: Es el único insumo físico de la Matriz de Ventas (M3-CU02) y, por lo tanto, de la Venta Bruta. Sin él, M3 no puede registrar ninguna venta ni liquidar ningún lote que no sea un siniestro total.

**Independent Test**: Se ejecuta la sincronización contra Módulo 2 para un lote con sacrificio registrado y se verifica que la copia local almacena cantidad final, peso total y fecha de registro como una sola unidad de información, con su fecha y hora de sincronización.

**Acceptance Scenarios**:

1. **Scenario**: Sincronización exitosa del resultado final
   - **Given** Módulo 2 ha registrado el resultado final de sacrificio de un lote con 8.500 pollos y 23.800 kg
   - **When** el proceso de sincronización se ejecuta
   - **Then** el sistema almacena `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg`, `fechaRegistro` y la fecha y hora de sincronización, y habilita el registro de la Matriz de Ventas para ese lote

2. **Scenario**: Rechazo de un resultado inválido
   - **Given** Módulo 2 informa un resultado con cantidad final o peso total igual a cero, negativo o ausente
   - **When** se ejecuta la sincronización
   - **Then** el sistema no almacena el resultado como válido, lo registra como incidencia y mantiene bloqueado el registro de la Matriz de Ventas para ese lote

3. **Scenario**: Corrección del resultado antes de ser utilizado
   - **Given** un resultado final sincronizado que aún no ha sido consumido por ninguna Matriz de Ventas `ACTIVA` y que Módulo 2 corrigió
   - **When** se ejecuta la siguiente sincronización
   - **Then** el sistema actualiza la copia local con las cifras corregidas y conserva la fecha y hora de la nueva sincronización

4. **Scenario**: Aviso de utilización tras generar la Liquidación
   - **Given** una Liquidación `ACTIVA` recién generada que consume una Matriz de Ventas basada en un resultado final de sacrificio
   - **When** M3 persiste la Liquidación
   - **Then** el sistema marca el resultado como utilizado en su copia local y emite a Módulo 2 el aviso de utilización con el identificador del resultado, para que Módulo 2 pueda bloquear correcciones posteriores

5. **Scenario**: Fallo del aviso de utilización
   - **Given** M3 generó la Liquidación y el aviso de utilización a Módulo 2 no pudo entregarse
   - **When** falla la entrega
   - **Then** el sistema conserva la Liquidación generada, marca el aviso como pendiente, lo reintenta en segundo plano y NO DEBE revertir ni bloquear ninguna operación financiera ya completada

6. **Scenario**: Módulo 2 no responde con copia local previa existente
   - **Given** existe una copia local del resultado final y Módulo 2 no responde
   - **When** el administrador registra la Matriz de Ventas
   - **Then** el sistema usa la copia local, conserva su fecha y hora de sincronización y no realiza consultas en vivo

---

### Edge Cases

- **Unidad indivisible**: La cantidad final de pollos y el peso total constituyen una sola unidad de información. M3 nunca almacena ni utiliza uno sin el otro.
- **Corrección posterior al consumo**: Si Módulo 2 corrige el resultado después de que M3 lo consumió en una Matriz de Ventas `ACTIVA`, M3 **no** recalcula la Matriz ni la Liquidación. El cambio exige anulación formal (M3-CU04). Esta es la razón por la cual el aviso de utilización debe existir.
- **Un único resultado por lote**: Módulo 2 registra un solo resultado final por lote. Si M3 recibe más de uno para el mismo lote, lo registra como incidencia y conserva el previamente sincronizado.
- **Peso promedio**: M3 calcula el peso promedio a partir de estos dos valores únicamente con fines de presentación. El cálculo monetario de M3-CU03 se realiza sobre el peso total.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST consultar a Módulo 2, mediante un proceso automático en segundo plano con intervalo configurable, el resultado final de sacrificio de los lotes con ciclo cerrado.
- **FR-002**: El sistema MUST obtener y almacenar, como una sola unidad de información: `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg` y `fechaRegistro`.
- **FR-003**: El sistema MUST validar que `cantidadFinalPollos` sea un entero mayor que cero y que `pesoTotalKg` sea mayor que cero. Los resultados que no cumplan MUST registrarse como incidencia y NO DEBEN habilitar el registro de la Matriz de Ventas.
- **FR-004**: El sistema MUST actualizar la copia local cuando Módulo 2 corrija un resultado que aún no haya sido consumido por una Matriz de Ventas `ACTIVA`.
- **FR-005**: El sistema NO DEBE recalcular una Matriz de Ventas ni una Liquidación `ACTIVA` ante una corrección posterior del resultado en Módulo 2. Toda corrección exige anulación formal (M3-CU04).
- **FR-006**: Al persistir una Liquidación `ACTIVA` que consume una Matriz de Ventas, el sistema MUST marcar el resultado final asociado como utilizado en su copia local y MUST emitir a Módulo 2 el aviso de utilización con el identificador del resultado. Esta es la **única escritura** de M3 hacia un módulo externo en todo el módulo.
- **FR-007**: El aviso de utilización MUST ser asíncrono e idempotente. Si su entrega falla, el sistema MUST conservar la Liquidación generada, marcar el aviso como pendiente y reintentarlo en segundo plano, sin revertir ni bloquear ninguna operación financiera completada.
- **FR-008**: Ante una respuesta fallida o ausente de Módulo 2, el sistema MUST conservar íntegra la copia local previa, registrar el fallo y reintentar en el siguiente ciclo, sin bloquear ninguna operación de M3.
- **FR-009**: El sistema MUST procesar las sincronizaciones de forma idempotente: obtener el mismo resultado más de una vez no DEBE crear registros duplicados.
- **FR-010**: El sistema MUST registrar como incidencia la recepción de más de un resultado final para el mismo lote, conservando el previamente sincronizado.

[NEEDS CLARIFICATION – D-03: Módulo 2 condiciona la corrección del resultado final a que Módulo 3 no lo haya utilizado, lo que requiere el aviso definido en FR-006. El mecanismo de ese aviso —su transporte, su formato y su confirmación— debe acordarse con el equipo de Módulo 2.]

### Key Entities

- **ResultadoFinalSacrificio**: Réplica local del resultado operativo de Módulo 2. Atributos: `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg`, `fechaRegistro`, `fechaHoraSincronizacion`, `estadoUtilizacionLocal` (`NO_UTILIZADO` | `UTILIZADO`).
- **AvisoUtilizacionResultado**: Notificación saliente de M3 hacia Módulo 2. Atributos: `idAviso`, `idResultado`, `idLiquidacion`, `fechaHoraEmision`, `estadoEntrega` (`PENDIENTE` | `ENTREGADO` | `FALLIDO`), `intentos`.
- **RegistroSincronizacion**: Bitácora técnica del proceso, compartida con M3-CU07, M3-CU09 y M3-CU10. Atributos: `idSincronizacion`, `fuente` (`MODULO_2`), `fechaHoraInicio`, `fechaHoraFin`, `resultado`, `descripcionError`, `cantidadRegistrosActualizados`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los resultados finales válidos disponibles en Módulo 2 se refleja en la copia local tras una sincronización exitosa.
- **SC-002**: 0% de Matrices de Ventas registradas sobre un resultado final inválido o ausente.
- **SC-003**: El 100% de las Liquidaciones `ACTIVA` que consumen una Matriz de Ventas genera su aviso de utilización hacia Módulo 2.
- **SC-004**: 0% de Liquidaciones revertidas o bloqueadas por un fallo en la entrega del aviso de utilización.
- **SC-005**: 0% de registros duplicados ante sincronizaciones repetidas del mismo resultado.
- **SC-006**: El 100% de los registros de Matriz de Ventas conserva la fecha y hora de sincronización del resultado final utilizado.
