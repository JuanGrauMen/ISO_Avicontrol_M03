# Feature Specification: M3-CU02 – Registrar Liquidación 

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro Definitivo de la Matriz de Ventas (Priority: P1)

Como administrador financiero, quiero registrar la Matriz de Ventas de un lote usando la cantidad final y el peso total del sacrificio obtenidos de Módulo 2 e ingresando el precio por kilogramo, para dejar constancia formal del ingreso comercial del lote y habilitar su posterior Liquidación.

**Why this priority**: La Matriz de Ventas es el documento de ingreso del módulo. Sin ella no puede calcularse la Venta Bruta y el lote no puede liquidarse, salvo en siniestro total.

**Independent Test**: Se selecciona un lote en `Vaciado Sanitario` identificado por su alerta histórica, con resultado final de sacrificio sincronizado desde Módulo 2. Se ingresa un precio por kg válido y se verifica que M3 calcula el peso promedio, guarda la Matriz de Ventas con fecha, hora y responsable, y bloquea registros duplicados.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso de la Matriz de Ventas al cierre
   - **Given** un galpón en estado `Vaciado Sanitario`, identificado con la alerta histórica de Módulo 1, con resultado final sincronizado desde Módulo 2 de 8.500 pollos sacrificados y 23.800 kg totales, y sin Matriz de Ventas `ACTIVA` previa
   - **When** el administrador ingresa un precio de \$4.500 COP/kg
   - **Then** el sistema almacena la Matriz de Ventas definitiva con 8.500 pollos vendidos, 23.800 kg de peso total, peso promedio calculado de 2,8 kg, precio por kg, fecha, hora y usuario responsable, y habilita el lote para generar la Liquidación (M3-CU03)

2. **Scenario**: Rechazo por resultado final de sacrificio inválido
   - **Given** un resultado final sincronizado desde Módulo 2 con cantidad de pollos o peso total igual a cero, negativo o ausente
   - **When** el usuario intenta registrar la Matriz de Ventas
   - **Then** el sistema bloquea el guardado e informa que debe existir un resultado final válido de sacrificio antes de registrar el precio comercial

3. **Scenario**: Rechazo por precio por kg no positivo
   - **Given** un lote con resultado final válido sincronizado desde Módulo 2
   - **When** el usuario ingresa un precio por kg menor o igual a 0 COP
   - **Then** el sistema rechaza el formulario e indica explícitamente el campo con valor no permitido

4. **Scenario**: Intento de duplicidad de registro
   - **Given** un lote que ya cuenta con una Matriz de Ventas en estado `ACTIVA`
   - **When** el usuario intenta registrar una nueva Matriz de Ventas sobre el mismo lote
   - **Then** el sistema deniega la creación, notifica que el lote ya posee una Matriz de Ventas definitiva y ofrece consultar la existente o iniciar su anulación (M3-CU04)

5. **Scenario**: Registro tras anulación de la matriz anterior
   - **Given** un lote cuya Matriz de Ventas previa fue anulada formalmente mediante M3-CU04
   - **When** el administrador registra una nueva Matriz de Ventas con el precio corregido
   - **Then** el sistema crea la nueva Matriz de Ventas en estado `ACTIVA`, conserva la anterior en estado `ANULADA` para auditoría y habilita nuevamente la generación de la Liquidación

6. **Scenario**: Indisponibilidad temporal de los módulos de origen
   - **Given** Módulo 1 o Módulo 2 no responden y M3 cuenta con las copias locales previamente sincronizadas del lote y del resultado final de sacrificio
   - **When** el administrador registra una Matriz de Ventas válida
   - **Then** el sistema usa los datos sincronizados, guarda sus fechas y horas de sincronización junto con la Matriz de Ventas y no realiza consultas en vivo a los módulos de origen

---

### Edge Cases

- **Venta registrada en galpón equivocado**: No se permite edición silenciosa. El usuario debe anular formalmente la Matriz de Ventas mediante M3-CU04 y crear el registro sobre el lote correcto.
- **Cambio posterior del resultado final en Módulo 2**: Una vez guardada la Matriz de Ventas en estado `ACTIVA`, el registro permanece inmutable y no se recalcula automáticamente si Módulo 2 corrige sus cifras. El cambio exige anulación formal (M3-CU04) y una nueva Matriz de Ventas.
- **Precios con decimales**: El precio por kg admite valores con decimales (ej. \$4.500,50 COP). La cantidad vendida y el peso total provienen del resultado final de sacrificio de Módulo 2; M3 no los digita ni los modifica.
- **Peso promedio como valor derivado**: El peso promedio se calcula y almacena únicamente con fines de presentación y control. La Venta Bruta se calcula en M3-CU03 sobre el peso total en kilogramos, nunca sobre el peso promedio.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir registrar una única Matriz de Ventas definitiva por lote, capturando manualmente solo el precio por kilogramo en pesos colombianos (decimal positivo mayor que 0). La cantidad final de pollos vendidos y el peso total provienen exclusivamente del resultado final de sacrificio sincronizado desde Módulo 2 (M3-CU08).
- **FR-002**: El sistema MUST exigir un resultado final de sacrificio válido, con cantidad final entera mayor a cero y peso total en kg mayor a cero. M3 MUST calcular `pesoPromedioKg = pesoTotalKg / pollosVendidos` y NO DEBE editar ninguno de los valores operativos de origen.
- **FR-003**: El sistema MUST rechazar un precio por kg menor o igual a cero y MUST bloquear el registro cuando el resultado final requerido no exista o sea inválido.
- **FR-004**: El sistema MUST impedir registrar una segunda Matriz de Ventas en estado `ACTIVA` sobre un lote que ya cuente con una.
- **FR-005**: El sistema MUST asociar de forma inmutable a cada Matriz de Ventas: identificador de lote, identificador del resultado final de sacrificio, cantidad final, peso total, peso promedio calculado, precio por kg, fecha y hora de registro, usuario responsable y fechas de sincronización de Módulo 1 y Módulo 2.
- **FR-006**: Si no existen las copias locales sincronizadas del lote y del resultado final de sacrificio, el sistema MUST bloquear el registro e informar que no hay datos disponibles. Si existen, MUST usarlas aunque Módulo 1 o Módulo 2 estén temporalmente indisponibles.
- **FR-007**: Toda Matriz de Ventas `ACTIVA` es inmutable. Su corrección se ejecuta exclusivamente mediante el flujo de anulación definido en **M3-CU04 – Anular Documento Financiero**. El lote solo admite una nueva Matriz de Ventas cuando la anterior haya sido anulada formalmente.

### Key Entities

- **ResultadoFinalSacrificio**: Resultado operativo originado en Módulo 2 y obtenido por M3 mediante consulta (M3-CU08). Atributos sincronizados: `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg`, `fechaRegistro`, `fechaHoraSincronizacion`, `estadoUtilizacionLocal`.
- **MatrizVentas**: Documento de ingreso del lote. Atributos clave: `idVenta`, `idLote`, `idResultadoSacrificio`, `pollosVendidos` (origen M2), `pesoTotalKg` (origen M2), `pesoPromedioKg` (calculado, solo presentación), `precioKgCop`, `estado` (`ACTIVA` | `ANULADA`), `fechaHoraRegistro`, `usuarioResponsable`.

> **Nota**: La entidad `RegistroAnulacionVenta` de la versión anterior de esta especificación se traslada a **M3-CU04** como `RegistroAnulacion`, unificada con la anulación de Liquidación.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las Matrices de Ventas utiliza un resultado final válido sincronizado desde Módulo 2 y un precio por kg mayor a cero.
- **SC-002**: 0% de ocurrencia de Matrices de Ventas duplicadas en estado `ACTIVA` sobre un mismo lote.
- **SC-003**: El tiempo de validación y guardado del formulario es inferior a 2 segundos.
- **SC-004**: El 100% de los registros conserva el resultado final de sacrificio y las fechas de sincronización utilizadas.
- **SC-005**: 0% de Matrices de Ventas `ACTIVA` modificadas sin pasar por M3-CU04.
