# Feature Specification: M3-CU04 – Anular Documento Financiero

**Created**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Corrección Auditable de un Documento Financiero (Priority: P1)

Como administrador financiero, quiero anular formalmente una Matriz de Ventas o una Liquidación emitida con datos erróneos, dejando constancia del motivo, la fecha, la hora y el responsable, para corregir el error sin destruir el registro original y habilitar la emisión de un documento nuevo sobre el mismo lote.

**Why this priority**: La inmutabilidad de los Documentos Financieros (RT-03) solo es sostenible si existe un mecanismo formal de corrección. Sin este caso de uso, un error de digitación deja el lote bloqueado de forma permanente y obliga a editar registros contables, lo que destruye la auditabilidad de todo el módulo.

**Independent Test**: Sobre un lote con Matriz de Ventas `ACTIVA` y sin Liquidación `ACTIVA`, se solicita la anulación de la matriz, se ingresa un motivo válido y se confirma. Se verifica que la matriz queda en estado `ANULADA` con su registro de auditoría completo, que sus valores originales se conservan íntegros y que el lote admite una nueva Matriz de Ventas.

**Acceptance Scenarios**:

1. **Scenario**: Anulación exitosa de una Matriz de Ventas sin liquidación asociada
   - **Given** un lote con una Matriz de Ventas en estado `ACTIVA` y sin ninguna Liquidación en estado `ACTIVA` asociada
   - **When** el administrador financiero solicita la anulación, ingresa el motivo y confirma la acción
   - **Then** el sistema marca la Matriz de Ventas como `ANULADA`, conserva íntegros todos sus valores originales, registra motivo, fecha, hora y usuario responsable, y rehabilita el lote para registrar una nueva Matriz de Ventas (M3-CU02)

2. **Scenario**: Anulación exitosa de una Liquidación
   - **Given** un lote con una Liquidación en estado `ACTIVA`
   - **When** el administrador financiero solicita la anulación, ingresa el motivo y confirma la acción
   - **Then** el sistema marca la Liquidación como `ANULADA`, conserva íntegros su snapshot financiero y sus cuatro indicadores, registra motivo, fecha, hora y usuario responsable, y rehabilita el lote para generar una nueva Liquidación (M3-CU03)
   - **And** la Matriz de Ventas asociada permanece en estado `ACTIVA` y sin alteración alguna

3. **Scenario**: Bloqueo por orden de anulación incorrecto
   - **Given** un lote con una Matriz de Ventas en estado `ACTIVA` y una Liquidación en estado `ACTIVA` que la consume
   - **When** el administrador financiero intenta anular la Matriz de Ventas
   - **Then** el sistema bloquea la acción, no modifica ningún documento e informa: "Debe anular primero la liquidación del lote. Una matriz de ventas con liquidación activa no puede anularse"

4. **Scenario**: Corrección completa de un error de precio
   - **Given** un lote con Matriz de Ventas `ACTIVA` registrada con un precio por kg erróneo y su Liquidación `ACTIVA` ya generada
   - **When** el administrador financiero anula primero la Liquidación y a continuación la Matriz de Ventas, ingresando motivo en ambas
   - **Then** el sistema conserva ambos documentos en estado `ANULADA` con sus respectivos registros de auditoría, y el lote queda habilitado para registrar una nueva Matriz de Ventas con el precio correcto y generar una nueva Liquidación

5. **Scenario**: Rechazo por motivo de anulación vacío
   - **Given** un Documento Financiero en estado `ACTIVA` seleccionado para anulación
   - **When** el usuario confirma la acción sin ingresar el motivo o ingresando únicamente espacios en blanco
   - **Then** el sistema rechaza la operación, no modifica el documento e indica que el motivo de anulación es obligatorio

6. **Scenario**: Intento de anular un documento ya anulado
   - **Given** un Documento Financiero que se encuentra en estado `ANULADA`
   - **When** el usuario intenta anularlo nuevamente
   - **Then** el sistema deniega la acción, informa que el documento ya fue anulado y muestra el registro de anulación existente con su motivo, fecha, hora y responsable

7. **Scenario**: Cancelación de la anulación por parte del usuario
   - **Given** el usuario abrió el flujo de anulación de un Documento Financiero `ACTIVA` e ingresó un motivo
   - **When** el usuario cancela en la pantalla de confirmación
   - **Then** el sistema no modifica el estado del documento, no crea registro de anulación y regresa a la vista anterior sin efectos

---

### Edge Cases

- **Conservación total del documento anulado**: La anulación **nunca** edita ni elimina valores del documento original. Un documento `ANULADA` conserva todos sus campos tal como fueron emitidos, incluido el snapshot financiero en el caso de la Liquidación. El estado es el único atributo que cambia.
- **Anulaciones sucesivas sobre el mismo lote**: Un lote puede acumular varias Matrices de Ventas y varias Liquidaciones en estado `ANULADA`, pero como máximo una de cada tipo en estado `ACTIVA`. Todas permanecen consultables en el historial (M3-CU06).
- **Anulación con módulos de origen indisponibles**: La anulación opera exclusivamente sobre documentos almacenados en M3 y no requiere consulta alguna a Módulo 1 o Módulo 2. La indisponibilidad de los módulos de origen no la bloquea.
- **Concurrencia**: Si dos usuarios intentan anular el mismo documento simultáneamente, solo la primera operación se aplica; la segunda recibe el mensaje de documento ya anulado.
- **Efecto sobre las vistas derivadas**: El Desglose (M3-CU05) y el Historial (M3-CU06) reflejan el nuevo estado de inmediato, sin requerir acción adicional, por tratarse de proyecciones de solo lectura.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST proveer un flujo único de anulación aplicable a los dos tipos de Documento Financiero definidos en el glosario del módulo: **Matriz de Ventas** y **Liquidación**.
- **FR-002**: El sistema MUST permitir anular únicamente documentos que se encuentren en estado `ACTIVA`, y MUST rechazar la anulación de documentos en estado `ANULADA`.
- **FR-003**: El sistema MUST exigir un motivo de anulación no vacío (mínimo 10 caracteres, máximo 500) antes de permitir la confirmación.
- **FR-004**: El sistema MUST mostrar una pantalla de confirmación explícita antes de persistir la anulación, indicando el tipo de documento, el lote afectado y las consecuencias de la acción. Si el usuario cancela, el sistema NO DEBE realizar cambio alguno.
- **FR-005**: Al confirmarse la anulación, el sistema MUST cambiar el estado del documento a `ANULADA` y MUST crear un registro de auditoría con motivo, fecha, hora y usuario responsable.
- **FR-006**: El sistema MUST conservar íntegros todos los valores del documento anulado. La anulación NO DEBE editar, recalcular ni eliminar ningún atributo distinto del estado.
- **FR-007**: El sistema MUST impedir la anulación de una Matriz de Ventas cuando exista una Liquidación en estado `ACTIVA` asociada a ella, informando que debe anularse primero la Liquidación (regla transversal RT-05).
- **FR-008**: La anulación de una Liquidación NO DEBE modificar el estado de la Matriz de Ventas asociada (regla transversal RT-06).
- **FR-009**: Tras anular una Matriz de Ventas, el sistema MUST rehabilitar el lote para registrar una nueva Matriz de Ventas (M3-CU02). Tras anular una Liquidación, MUST rehabilitar el lote para generar una nueva Liquidación (M3-CU03).
- **FR-010**: El cambio de estado y la creación del registro de auditoría MUST ejecutarse dentro de una única transacción atómica. Si alguna operación falla, el sistema NO DEBE conservar cambios parciales.
- **FR-011**: El sistema MUST operar exclusivamente sobre documentos almacenados en M3, sin requerir consultas a Módulo 1 ni a Módulo 2.
- **FR-012**: El sistema MUST permitir consultar el registro de anulación de cualquier documento `ANULADA` desde el historial (M3-CU06) y desde el desglose (M3-CU05).

### Key Entities

- **DocumentoFinanciero**: Abstracción común de **MatrizVentas** y **Liquidacion**. Atributos compartidos relevantes para este caso de uso: `idDocumento`, `tipoDocumento` (`MATRIZ_VENTAS` | `LIQUIDACION`), `idLote`, `estado` (`ACTIVA` | `ANULADA`).
- **RegistroAnulacion**: Auditoría de la anulación de un Documento Financiero. Atributos: `idAnulacion`, `idDocumento`, `tipoDocumento` (`MATRIZ_VENTAS` | `LIQUIDACION`), `motivo`, `fechaHoraAnulacion`, `usuarioResponsable`. Reemplaza y unifica las entidades `RegistroAnulacionVenta` y `RegistroAnulacionLiquidacion` de las versiones anteriores de M3-CU02 y M3-CU03.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las anulaciones cuenta con motivo no vacío, fecha, hora y usuario responsable registrados.
- **SC-002**: 0% de documentos anulados presenta alteración en algún atributo distinto del estado.
- **SC-003**: 0% de anulaciones de Matriz de Ventas completadas mientras exista una Liquidación `ACTIVA` asociada.
- **SC-004**: El tiempo de validación, confirmación y persistencia de la anulación es inferior a 2 segundos.
- **SC-005**: El 100% de los documentos anulados permanece consultable en el historial con su registro de anulación asociado.
- **SC-006**: 0% de anulaciones deja cambios parciales ante una falla durante la transacción.
