# Feature Specification: M3-CU03 – Generar Matriz de Ventas (Liquidación del Lote)

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consolidación y Liquidación Financiera del Lote (Priority: P1)

Como administrador financiero, quiero generar la liquidación económica definitiva de un lote vendido calculando automáticamente la Venta Bruta, Pérdida por Mortalidad, Costos Operativos y Utilidad Neta, para conocer la rentabilidad real y auditable del ciclo avícola.

**Why this priority**: Es el entregable central del Módulo 3 y el cálculo económico final del sistema. Determina si el lote generó ganancias o pérdidas reales.

**Independent Test**: Con un lote en estado `Vaciado Sanitario`, matriz de ventas activa, datos locales sincronizados completos de población (origen Módulo 1) y compras valorizadas del ciclo (origen Módulo 2), se ejecuta la generación de la liquidación y se comprueba que los cuatro indicadores cuadran con precisión de 100% contra el cálculo matemático verificado.

**Acceptance Scenarios**:

1. **Scenario**: Generación exitosa de liquidación definitiva (Caso Dorado)
   - **Given** un galpón en estado `Vaciado Sanitario`, con Población Inicial sincronizada de 9.000 pollos (origen M1), Matriz de Ventas ACTIVA con 8.500 pollos vendidos, peso promedio 2,8 kg y precio \$4.500 COP/kg, y Costos Operativos valorizados de \$85.000.000 COP (compras de Alimento, Medicina y costo inicial de Pollitos)
   - **When** el administrador financiero solicita generar la liquidación
   - **Then** el sistema calcula y presenta:
     - **Venta Bruta**: \$107.100.000 COP (`8.500 × 2,8 × 4.500`)
     - **Pérdida por Mortalidad**: 500 pollos (`9.000 - 8.500`), equivalente al 5,56% (`(500 / 9.000) × 100`)
     - **Costos Operativos**: \$85.000.000 COP
     - **Utilidad Neta**: \$22.100.000 COP (`107.100.000 - 85.000.000`)
     - Estado de la liquidación: `ACTIVA`, fechada con usuario responsable

2. **Scenario**: Bloqueo por lote sin matriz de ventas registrada
   - **Given** un galpón en estado `Vaciado Sanitario` que NO posee una matriz de ventas en estado `ACTIVA`
   - **When** se intenta generar la liquidación del lote
   - **Then** el sistema bloquea la acción e informa claramente: "El lote no cuenta con registro de venta activo. Debe registrar la matriz de ventas (M3-CU02) antes de liquidar"

3. **Scenario**: Bloqueo por compras sin precio configurado
   - **Given** un lote en estado `Vaciado Sanitario` con venta registrada, pero con compras de alimento o medicina sincronizadas cuyo origen es Módulo 2 que carecen de tarifa/precio unitario vigente en el sistema
   - **When** se solicita generar la liquidación
   - **Then** el sistema detiene el proceso, no genera registros preliminares ni parciales, y muestra la lista detallada de insumos pendientes de precio

4. **Scenario**: Ausencia total de compras sincronizadas
   - **Given** un lote en estado `Vaciado Sanitario` con venta registrada para el cual M3 no cuenta con compras de alimento o medicina sincronizadas cuyo origen es Módulo 2
   - **When** se intenta generar la liquidación
   - **Then** el sistema notifica "No hay compras sincronizadas para este ciclo" y mantiene bloqueada la liquidación

5. **Scenario**: Escenario de mortalidad del 100% (Siniestro total)
   - **Given** un galpón en estado `Vaciado Sanitario` con un lote de 9.000 pollos iniciales y 0 pollos vendidos por mortalidad total, con compras del ciclo por \$40.000.000 COP
   - **When** se procesa la liquidación de cierre
   - **Then** el sistema genera la liquidación mostrando Venta Bruta = \$0 COP, Mortalidad = 9.000 aves (100%), Costos Operativos = \$40.000.000 COP y Utilidad Neta = -\$40.000.000 COP

6. **Scenario**: Anulación formal de liquidación definitiva
   - **Given** una liquidación en estado `ACTIVA` con datos que deben ser corregidos
   - **When** el administrador financiero ejecuta la anulación ingresando la justificación
   - **Then** el sistema marca la liquidación como `ANULADA`, conserva el registro histórico con fecha/responsable de anulación y rehabilita el lote para ser liquidado nuevamente

7. **Scenario**: Intento de liquidación antes del vaciado sanitario
   - **Given** un galpón en estado `En Cosecha` con una matriz de ventas `ACTIVA`
   - **When** el administrador intenta generar la liquidación
   - **Then** el sistema bloquea la acción e informa que la liquidación se habilita cuando Módulo 1 cambie el galpón a `Vaciado Sanitario` después de retirar todas las aves

8. **Scenario**: Indisponibilidad temporal de módulos de origen
   - **Given** Módulo 1 o Módulo 2 no está disponible y M3 cuenta con los datos locales sincronizados requeridos para el lote
   - **When** el administrador solicita generar la liquidación
   - **Then** el sistema genera la liquidación usando esos datos, conserva su fecha/hora de sincronización en el snapshot financiero y no realiza consultas en vivo a los módulos de origen

---

### Edge Cases

- **Regla matemática de redondeo monetario**: Las cifras monetarias totales en pesos colombianos (COP) se expresan como números enteros aplicando redondeo estándar `HALF_UP` (hacia el entero más cercano; si el decimal es exactamente 0.5 o mayor, redondea hacia arriba).
- **Porcentajes de mortalidad**: Se calculan sobre la población inicial oficial de Módulo 1 y se presentan formateados a 2 cifras decimales.
- **Inmutabilidad absoluta (Snapshot financiero)**: Una vez generada una liquidación `ACTIVA`, los valores quedan congelados. Si con posterioridad se modifican compras en Módulo 2 o poblaciones en Módulo 1, la liquidación existente no se altera; cualquier cambio requiere una anulación formal y nueva liquidación.
- **Exclusión de costos indirectos**: Los costos de administración, arrendamientos, nómina general o servicios no forman parte del cálculo operativo en este módulo.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST calcular la **Venta Bruta** aplicando la fórmula:  
  `Venta Bruta = Pollos Vendidos × Peso Promedio (kg) × Precio por kg (COP)` con redondeo `HALF_UP` a enteros.
- **FR-002**: El sistema MUST calcular la **Pérdida por Mortalidad** aplicando las fórmulas:
  - `Mortalidad Absoluta = Población Inicial sincronizada (origen Módulo 1) − Pollos Vendidos`
  - `Porcentaje Mortalidad = (Mortalidad Absoluta / Población Inicial) × 100` (con 2 decimales).
- **FR-003**: El sistema MUST consolidar los **Costos Operativos** exclusivamente como la suma valorizada de:
  - Costo de Alimento: suma de las compras de alimento asociadas al ciclo, sin descontar sobrantes.
  - Insumos Médicos: suma de las compras de medicamentos asociadas al ciclo, sin descontar sobrantes.
  - Costo de Población: costo inicial no acumulativo de adquisición de pollitos informado por Módulo 1.
- **FR-004**: El sistema MUST calcular la **Utilidad Neta** aplicando la fórmula:  
  `Utilidad Neta = Venta Bruta − Costos Operativos`.
- **FR-005**: El sistema MUST exigir que el galpón esté en estado `Vaciado Sanitario` y que exista una matriz de ventas en estado `ACTIVA` como condiciones previas obligatorias para liquidar.
- **FR-006**: El sistema MUST impedir la liquidación y listar las partidas faltantes si alguna compra de alimento o medicina del ciclo carece de precio asignado. No se permiten liquidaciones preliminares ni parciales.
- **FR-007**: El sistema MUST guardar la liquidación con estado `ACTIVA`, fecha/hora de generación y usuario responsable.
- **FR-008**: Toda liquidación generada MUST ser inmutable. Para corregir un error se MUST ejecutar un flujo de anulación que capture motivo, responsable y marque el estado en `ANULADA`, permitiendo generar una nueva liquidación sobre el lote.
- **FR-009**: El sistema MUST generar la liquidación exclusivamente con datos locales sincronizados cuyo origen sea Módulo 1 o Módulo 2, según corresponda, y conservar en el snapshot financiero la fecha/hora de sincronización de cada fuente. La generación no DEBE requerir consultas en vivo a los módulos de origen.

### Key Entities

- **Liquidacion**: Resultado financiero consolidado del lote. Atributos clave: `idLiquidacion`, `idLote`, `idVenta`, `ventaBruta`, `mortalidadAves`, `porcentajeMortalidad`, `costosOperativos`, `utilidadNeta`, `estado` (`ACTIVA` | `ANULADA`), `fechaGeneracion`, `usuarioResponsable`.
- **RegistroAnulacionLiquidacion**: Atributos: `idAnulacion`, `idLiquidacion`, `motivo`, `fechaAnulacion`, `usuarioResponsable`.
- **PrecioInsumo**: Tarifa vigente por insumo. Atributos: `idInsumo`, `tipoInsumo` (Alimento, Medicina, Pollito), `precioUnitarioCop`, `unidadMedida`.
- **SnapshotDatosOrigen**: Datos sincronizados utilizados para una liquidación, con `idLote`, `fuente` (Módulo 1 | Módulo 2), `fechaHoraSincronizacion` y referencias a población o partidas de costo.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las liquidaciones de prueba coinciden con exactitud al peso contra el cálculo aritmético manual verificado.
- **SC-002**: El tiempo total de procesamiento y despliegue de la liquidación en pantalla es inferior a 5 segundos tras pulsar "Generar Liquidación".
- **SC-003**: 0% de liquidaciones generadas con partidas de costo sin valorizar o sin matriz de ventas.
- **SC-004**: 100% de inmutabilidad: ninguna liquidación activa cambia su valor ante modificaciones posteriores de datos maestros sin anulación previa.
- **SC-005**: El 100% de las liquidaciones generadas conserva la fecha/hora de sincronización de los datos de origen usados en el cálculo.
