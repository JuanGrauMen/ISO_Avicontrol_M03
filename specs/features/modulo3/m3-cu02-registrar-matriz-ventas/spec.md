# Feature Specification: M3-CU02 – Registrar Matriz de Ventas

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro Definitivo de Matriz de Ventas (Priority: P1)

Como administrador financiero, quiero registrar la matriz de ventas definitiva de un lote al cierre de la cosecha (cantidad de pollos vendidos, peso promedio y precio por kg), para dejar constancia formal del ingreso comercial obtenido por el lote y habilitar su posterior liquidación.

**Why this priority**: Sin el registro de la venta es imposible calcular los ingresos reales y la rentabilidad del lote. Es el dato comercial fundamental del Módulo 3.

**Independent Test**: Se selecciona un galpón en estado `En Cosecha` desde la copia local sincronizada con origen en Módulo 1, se ingresan pollos vendidos, peso promedio y precio/kg válidos, y se verifica que la venta queda guardada con fecha, hora, responsable, población sincronizada utilizada y asociada al lote, bloqueando registros duplicados.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso de matriz de ventas al cierre
   - **Given** un galpón en estado `En Cosecha` con población viva sincronizada cuyo origen es el Módulo 1 y sin matriz de ventas activa previa
   - **When** el administrador digita una cantidad de pollos vendidos tal que `1 ≤ pollos vendidos ≤ población viva`, un peso promedio en kg mayor a 0, y un precio por kg en COP mayor a 0
   - **Then** el sistema almacena la matriz de ventas definitiva, registra la fecha/hora y el usuario responsable, y habilita el lote para la generación de la liquidación

2. **Scenario**: Rechazo por cantidad de pollos vendidos inválida
   - **Given** un galpón seleccionado con población viva sincronizada de 8.500 pollos, cuyo origen es el Módulo 1
   - **When** el usuario intenta registrar 0 pollos vendidos, un valor negativo, o una cantidad superior a 8.500 (ej. 9.000 pollos)
   - **Then** el sistema bloquea el guardado, resalta el campo de pollos vendidos y muestra un mensaje de error indicando que la cantidad debe estar en el rango `[1, 8.500]`

3. **Scenario**: Rechazo por peso promedio o precio por kg no positivos
   - **Given** un galpón en estado `En Cosecha`
   - **When** el usuario ingresa un peso promedio menor o igual a 0 kg o un precio por kg menor o igual a 0 COP
   - **Then** el sistema rechaza el formulario e indica explícitamente el campo con valor no permitido

4. **Scenario**: Intento de duplicidad de registro de venta
   - **Given** un lote que ya cuenta con una matriz de ventas en estado `ACTIVA`
   - **When** el usuario intenta registrar una nueva venta sobre el mismo lote
   - **Then** el sistema deniega la creación, notifica que el lote ya posee una venta registrada definitiva y ofrece consultar la existente o iniciar el flujo de anulación

5. **Scenario**: Anulación formal de matriz de ventas por error
   - **Given** una matriz de ventas previamente registrada con datos erróneos
   - **When** el administrador financiero solicita la anulación, ingresa el motivo de la anulación y confirma la acción
   - **Then** el sistema marca la matriz anterior como `ANULADA` (conservándola para auditoría con fecha, hora y responsable de la anulación) y rehabilita el lote para registrar una nueva matriz de ventas

6. **Scenario**: Indisponibilidad temporal de Módulo 1 durante el registro
   - **Given** Módulo 1 no está disponible y M3 cuenta con una copia local previamente sincronizada del lote
   - **When** el administrador registra una matriz de ventas válida
   - **Then** el sistema valida contra la población sincronizada, guarda la fecha/hora de esa sincronización junto con la venta y no requiere una consulta en vivo a Módulo 1

---

### Edge Cases

- **Venta registrada en galpón equivocado**: No se permite edición silenciosa; el usuario debe anular formalmente la venta con justificación y crear el registro en el galpón correspondiente.
- **Cambio posterior de población viva en Módulo 1**: Una vez guardada la matriz de ventas en estado `ACTIVA`, el registro permanece inmutable y no se recalcula automáticamente si el Módulo 1 cambia retroactivamente sus cifras.
- **Indisponibilidad o ausencia de datos sincronizados**: Si Módulo 1 no está disponible, M3 utiliza la última copia local completa. Si no existe una copia sincronizada del lote, el sistema bloquea el registro e informa que debe esperarse la sincronización; no realiza consultas en vivo desde la página.
- **Precios con decimales**: El precio por kg admite valores con decimales (ej. \$4.500,50 COP), mientras que la cantidad de pollos vendidos debe ser estrictamente un número entero positivo.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir registrar una única matriz de ventas definitiva por lote, capturando exclusivamente mediante digitación manual:
  - Cantidad de pollos vendidos (entero positivo).
  - Peso promedio por ave en kilogramos (decimal positivo > 0).
  - Precio por kilogramo en pesos colombianos - COP (decimal positivo > 0).
- **FR-002**: El sistema MUST validar que la cantidad de pollos vendidos cumpla la regla de negocio `1 ≤ pollos vendidos ≤ población viva sincronizada`, cuyo origen oficial es el Módulo 1. La validación no DEBE requerir una consulta en vivo a Módulo 1.
- **FR-003**: El sistema MUST rechazar valores menores o iguales a cero en peso promedio y precio por kg.
- **FR-004**: El sistema MUST impedir registrar una segunda matriz de ventas activa sobre un lote que ya cuente con una venta guardada.
- **FR-005**: El sistema MUST asociar de forma inmutable a cada registro de venta: identificador de lote, fecha y hora de registro, usuario responsable, población viva sincronizada utilizada y fecha/hora de dicha sincronización.
- **FR-006**: El sistema MUST proveer un flujo de anulación formal de matrices de venta que exija ingresar motivo de anulación, registrando fecha, hora y responsable, y cambiando el estado de la matriz a `ANULADA`.
- **FR-007**: El sistema MUST rehabilitar el lote para un nuevo registro de ventas únicamente cuando su matriz previa haya sido anulada formalmente.
- **FR-008**: Si no existe una copia local sincronizada del lote, el sistema MUST bloquear el registro de venta e informar que no hay datos disponibles para validar la población. Si existe, MUST usarla aunque Módulo 1 esté temporalmente indisponible.

### Key Entities

- **MatrizVentas**: Registro de la transacción comercial del lote. Atributos clave: `idVenta`, `idLote`, `pollosVendidos`, `pesoPromedioKg`, `precioKgCop`, `poblacionVivaSincronizada`, `fechaHoraSincronizacionPoblacion`, `estado` (`ACTIVA` | `ANULADA`), `fechaRegistro`, `usuarioResponsable`.
- **RegistroAnulacionVenta**: Auditoría de anulación. Atributos: `idAnulacion`, `idVenta`, `motivo`, `fechaAnulacion`, `usuarioResponsable`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las ventas registradas cumplen estrictamente el rango de población viva sincronizada cuyo origen es el Módulo 1 y valores mayores a cero.
- **SC-002**: 0% de ocurrencia de registros duplicados de ventas activas sobre un mismo lote.
- **SC-003**: El 100% de las anulaciones cuentan con justificación, marca temporal y usuario responsable registrado.
- **SC-004**: El tiempo de validación y guardado del formulario de ventas es inferior a 2 segundos.
- **SC-005**: El 100% de los registros de venta conserva la población sincronizada y la fecha/hora usadas para su validación.
