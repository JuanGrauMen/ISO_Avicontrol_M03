# Feature Specification: M3-CU02 – Registrar Matriz de Ventas

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro Definitivo de Matriz de Ventas (Priority: P1)

Como administrador financiero, quiero valorar la venta definitiva de un lote usando la cantidad final y el peso total del sacrificio registrados por Módulo 2, ingresando el precio por kg, para dejar constancia formal del ingreso comercial obtenido por el lote y habilitar su posterior liquidación.

**Why this priority**: Sin el registro de la venta es imposible calcular los ingresos reales y la rentabilidad del lote. Es el dato comercial fundamental del Módulo 3.

**Independent Test**: Se selecciona un lote en `Vaciado Sanitario` identificado por su alerta histórica, con resultado final de sacrificio sincronizado desde Módulo 2. Se ingresa un precio/kg válido y se verifica que M3 calcula el peso promedio, guarda la matriz comercial con fecha, hora y responsable, y bloquea registros duplicados.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso de matriz de ventas al cierre
   - **Given** un galpón en estado `Vaciado Sanitario`, identificado con la alerta histórica de Módulo 1, con resultado final sincronizado desde Módulo 2 de 8.500 pollos sacrificados y 23.800 kg totales, y sin matriz de ventas activa previa
   - **When** el administrador ingresa un precio de $4.500 COP/kg
   - **Then** el sistema almacena la matriz comercial definitiva con 8.500 pollos vendidos, 23.800 kg de peso total, peso promedio calculado de 2,8 kg, fecha/hora y usuario responsable, y habilita el lote para generar la liquidación

2. **Scenario**: Rechazo por resultado final de sacrificio inválido
   - **Given** un resultado final sincronizado desde Módulo 2 con cantidad de pollos o peso total igual a cero, negativo o ausente
   - **When** el usuario intenta valorar la venta
   - **Then** el sistema bloquea el guardado e informa que debe existir un resultado final válido de sacrificio antes de registrar el precio comercial

3. **Scenario**: Rechazo por precio por kg no positivo
   - **Given** un lote con resultado final válido sincronizado desde Módulo 2
   - **When** el usuario ingresa un precio por kg menor o igual a 0 COP
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
   - **Given** Módulo 1 o Módulo 2 no está disponible y M3 cuenta con las copias locales previamente sincronizadas del lote y del resultado final de sacrificio
   - **When** el administrador registra una valoración comercial válida
   - **Then** el sistema usa los resultados sincronizados, guarda sus fechas/horas de sincronización junto con la matriz comercial y no requiere consultas en vivo a los módulos de origen

---

### Edge Cases

- **Venta registrada en galpón equivocado**: No se permite edición silenciosa; el usuario debe anular formalmente la venta con justificación y crear el registro en el galpón correspondiente.
- **Cambio posterior del resultado final en Módulo 2**: Una vez guardada la matriz comercial en estado `ACTIVA`, el registro permanece inmutable y no se recalcula automáticamente si Módulo 2 corrige posteriormente sus cifras; el cambio exige anulación formal y nueva matriz.
- **Precios con decimales**: El precio por kg admite valores con decimales (ej. \$4.500,50 COP). La cantidad vendida y el peso total provienen del resultado operativo de Módulo 2; M3 no los digita ni los modifica.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir registrar una única matriz comercial definitiva por lote, capturando manualmente solo el precio por kilogramo en pesos colombianos - COP (decimal positivo > 0). La cantidad final de pollos vendidos y el peso total provienen exclusivamente del resultado final de sacrificio sincronizado desde Módulo 2.
- **FR-002**: El sistema MUST exigir un resultado final de sacrificio válido, con cantidad final entera mayor a cero y peso total en kg mayor a cero, sincronizado desde Módulo 2. M3 MUST calcular `pesoPromedioKg = pesoTotalKg / pollosVendidos` y no debe editar ninguno de los tres valores operativos.
- **FR-003**: El sistema MUST rechazar un precio por kg menor o igual a cero y bloquear el registro cuando el resultado final requerido no exista o sea inválido.
- **FR-004**: El sistema MUST impedir registrar una segunda matriz de ventas activa sobre un lote que ya cuente con una venta guardada.
- **FR-005**: El sistema MUST asociar de forma inmutable a cada matriz comercial: identificador de lote, identificador del resultado final de sacrificio, cantidad final, peso total, peso promedio calculado, precio por kg, fecha/hora de registro, usuario responsable y fechas/horas de sincronización de Módulo 1 y Módulo 2.
- **FR-006**: El sistema MUST proveer un flujo de anulación formal de matrices de venta que exija ingresar motivo de anulación, registrando fecha, hora y responsable, y cambiando el estado de la matriz a `ANULADA`.
- **FR-007**: El sistema MUST rehabilitar el lote para un nuevo registro de ventas únicamente cuando su matriz previa haya sido anulada formalmente.
- **FR-008**: Si no existen las copias locales sincronizadas del lote y del resultado final de sacrificio, el sistema MUST bloquear el registro comercial e informar que no hay datos disponibles. Si existen, MUST usarlas aunque Módulo 1 o Módulo 2 estén temporalmente indisponibles.

### Key Entities

- **ResultadoFinalSacrificio**: Resultado operativo originado en Módulo 2. Atributos sincronizados: `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg`, `fechaRegistro`, `estadoUtilizacion`.
- **MatrizVentas**: Registro financiero de la transacción comercial del lote. Atributos clave: `idVenta`, `idLote`, `idResultadoSacrificio`, `pollosVendidos` (origen M2), `pesoTotalKg` (origen M2), `pesoPromedioKg` (calculado), `precioKgCop`, `estado` (`ACTIVA` | `ANULADA`), `fechaRegistro`, `usuarioResponsable`.
- **RegistroAnulacionVenta**: Auditoría de anulación. Atributos: `idAnulacion`, `idVenta`, `motivo`, `fechaAnulacion`, `usuarioResponsable`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las matrices comerciales utiliza un resultado final válido sincronizado desde Módulo 2 y un precio por kg mayor a cero.
- **SC-002**: 0% de ocurrencia de registros duplicados de ventas activas sobre un mismo lote.
- **SC-003**: El 100% de las anulaciones cuentan con justificación, marca temporal y usuario responsable registrado.
- **SC-004**: El tiempo de validación y guardado del formulario de ventas es inferior a 2 segundos.
- **SC-005**: El 100% de los registros conserva el resultado final de sacrificio y las fechas/horas de sincronización utilizadas para su valoración.
