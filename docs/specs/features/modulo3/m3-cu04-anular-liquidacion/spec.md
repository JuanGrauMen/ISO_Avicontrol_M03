# Feature Specification: M3-CU04 – Anular Liquidación

**Created**: 2026-09-18  
**Actualizado**: 2026-09-21  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Corrección Auditable de una Liquidación (Priority: P1)

Como administrador financiero, quiero anular formalmente una Liquidación emitida con datos erróneos, dejando constancia del motivo, la fecha, la hora y el responsable, para corregir el error sin destruir el registro original y habilitar la generación de una Liquidación nueva sobre el mismo lote.

**Why this priority**: La inmutabilidad de la Liquidación (RT-03) solo es sostenible si existe un mecanismo formal de corrección. Sin este caso de uso, un error de digitación del precio por kg deja el lote bloqueado de forma permanente y obliga a editar registros contables, lo que destruye la auditabilidad de todo el módulo.

**Independent Test**: Sobre un lote con Liquidación `ACTIVA`, se solicita la anulación, se ingresa un motivo válido y se confirma. Se verifica que la Liquidación queda en estado `ANULADA` con su registro de auditoría completo, que su snapshot financiero y sus indicadores se conservan íntegros y que el lote admite una nueva Liquidación.

**Acceptance Scenarios**:

1. **Scenario**: Anulación exitosa de una Liquidación
   - **Given** un lote con una Liquidación en estado `ACTIVA`
   - **When** el administrador financiero solicita la anulación, ingresa el motivo y confirma la acción
   - **Then** el sistema marca la Liquidación como `ANULADA`, conserva íntegros su snapshot financiero, sus datos de venta y sus cuatro indicadores, registra motivo, fecha, hora y usuario responsable, y rehabilita el lote para generar una nueva Liquidación (M3-CU03)

2. **Scenario**: Corrección completa de un error de precio
   - **Given** un lote con Liquidación `ACTIVA` generada con un precio por kg erróneo
   - **When** el administrador financiero anula la Liquidación ingresando el motivo y a continuación genera una nueva con el precio correcto
   - **Then** el sistema conserva la Liquidación errónea en estado `ANULADA` con su registro de auditoría y la nueva Liquidación queda en estado `ACTIVA`

3. **Scenario**: Rechazo por motivo de anulación vacío
   - **Given** una Liquidación en estado `ACTIVA` seleccionada para anulación
   - **When** el usuario confirma la acción sin ingresar el motivo o ingresando únicamente espacios en blanco
   - **Then** el sistema rechaza la operación, no modifica la Liquidación e indica que el motivo de anulación es obligatorio

4. **Scenario**: Intento de anular una Liquidación ya anulada
   - **Given** una Liquidación que se encuentra en estado `ANULADA`
   - **When** el usuario intenta anularla nuevamente
   - **Then** el sistema deniega la acción, informa que la Liquidación ya fue anulada y muestra el registro de anulación existente con su motivo, fecha, hora y responsable

5. **Scenario**: Cancelación de la anulación por parte del usuario
   - **Given** el usuario abrió el flujo de anulación de una Liquidación `ACTIVA` e ingresó un motivo
   - **When** el usuario cancela en la pantalla de confirmación
   - **Then** el sistema no modifica el estado de la Liquidación, no crea registro de anulación y regresa a la vista anterior sin efectos

---

### Edge Cases

- **Conservación total de la Liquidación anulada**: La anulación **nunca** edita ni elimina valores de la Liquidación original. Una Liquidación `ANULADA` conserva todos sus campos tal como fueron emitidos, incluido el snapshot financiero y el precio por kg ingresado. El estado es el único atributo que cambia.
- **Anulaciones sucesivas sobre el mismo lote**: Un lote puede acumular varias Liquidaciones en estado `ANULADA`, pero como máximo una en estado `ACTIVA`. Todas permanecen consultables en el historial (M3-CU06).
- **Resultado final de sacrificio ya avisado como utilizado**: Anular la Liquidación no retira el aviso de utilización enviado a Módulo 2 (M3-CU08). El resultado permanece marcado como utilizado; la nueva Liquidación reutiliza el mismo resultado sincronizado.
- **Anulación con módulos de origen indisponibles**: La anulación opera exclusivamente sobre Liquidaciones almacenadas en M3 y no requiere consulta alguna a Módulo 1 o Módulo 2. La indisponibilidad de los módulos de origen no la bloquea.
- **Concurrencia**: Si dos usuarios intentan anular la misma Liquidación simultáneamente, solo la primera operación se aplica; la segunda recibe el mensaje de Liquidación ya anulada.
- **Efecto sobre las vistas derivadas**: El Desglose (M3-CU05) y el Historial (M3-CU06) reflejan el nuevo estado de inmediato, sin requerir acción adicional, por tratarse de proyecciones de solo lectura.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST proveer un flujo de anulación aplicable a la **Liquidación**, único Documento Financiero definido en el glosario del módulo.
- **FR-002**: El sistema MUST permitir anular únicamente Liquidaciones que se encuentren en estado `ACTIVA`, y MUST rechazar la anulación de Liquidaciones en estado `ANULADA`.
- **FR-002b**: El único punto de entrada a la anulación es **M3-CU03**: al intentar generar la Liquidación de un lote que ya tiene una `ACTIVA` (CU03 escenario 8), el sistema ofrece consultarla o iniciar su anulación. El Desglose (M3-CU05) y el Historial (M3-CU06) solo **consultan** el registro de anulación; no la inician. Corresponde al `«extend»` de *Anular Liquidación* sobre *Generar Liquidación* en el diagrama de casos de uso.
- **FR-003**: El sistema MUST exigir un motivo de anulación no vacío (mínimo 10 caracteres, máximo 500) antes de permitir la confirmación.
- **FR-004**: El sistema MUST mostrar una pantalla de confirmación explícita antes de persistir la anulación, indicando el lote afectado y las consecuencias de la acción. Si el usuario cancela, el sistema NO DEBE realizar cambio alguno.
- **FR-005**: Al confirmarse la anulación, el sistema MUST cambiar el estado de la Liquidación a `ANULADA` y MUST crear un registro de auditoría con motivo, fecha, hora y usuario responsable.
- **FR-006**: El sistema MUST conservar íntegros todos los valores de la Liquidación anulada. La anulación NO DEBE editar, recalcular ni eliminar ningún atributo distinto del estado.
- **FR-007**: Tras anular una Liquidación, el sistema MUST rehabilitar el lote para generar una nueva Liquidación (M3-CU03).
- **FR-008**: El cambio de estado y la creación del registro de auditoría MUST ejecutarse dentro de una única transacción atómica. Si alguna operación falla, el sistema NO DEBE conservar cambios parciales.
- **FR-009**: El sistema MUST operar exclusivamente sobre Liquidaciones almacenadas en M3, sin requerir consultas a Módulo 1 ni a Módulo 2.
- **FR-010**: El sistema MUST permitir consultar el registro de anulación de cualquier Liquidación `ANULADA` desde el historial (M3-CU06) y desde el desglose (M3-CU05).

### Key Entities

- **Liquidacion**: Documento Financiero sobre el que opera esta especificación. Definida en M3-CU03. Atributos relevantes aquí: `idLiquidacion`, `idLote`, `estado` (`ACTIVA` | `ANULADA`).
- **RegistroAnulacion**: Auditoría de la anulación de una Liquidación. Atributos: `idAnulacion`, `idLiquidacion`, `motivo`, `fechaHoraAnulacion`, `usuarioResponsable`.

> **Nota**: La versión anterior de esta especificación (*Anular Documento Financiero*) cubría también la Matriz de Ventas como documento independiente y definía el orden de anulación entre ambos (RT-05, RT-06). Al unificarse la Matriz de Ventas en la Liquidación (2026-09-21), esas reglas quedan sin objeto y la entidad `DocumentoFinanciero` con su atributo `tipoDocumento` se elimina.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las anulaciones cuenta con motivo no vacío, fecha, hora y usuario responsable registrados.
- **SC-002**: 0% de Liquidaciones anuladas presenta alteración en algún atributo distinto del estado.
- **SC-003**: El tiempo de validación, confirmación y persistencia de la anulación es inferior a 2 segundos.
- **SC-004**: El 100% de las Liquidaciones anuladas permanece consultable en el historial con su registro de anulación asociado.
- **SC-005**: 0% de anulaciones deja cambios parciales ante una falla durante la transacción.
