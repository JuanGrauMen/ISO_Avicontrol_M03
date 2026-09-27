# Feature Specification: M3-CU06 – Consultar Historial de Liquidaciones

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta Histórica y Comparabilidad de Ciclos (Priority: P3)

Como administrador financiero, quiero consultar el historial cronológico de todas las Liquidaciones generadas filtrando por galpón y rango de fechas, para comparar los resultados de rentabilidad y mortalidad entre ciclos y tomar decisiones de mejora operativa.

**Why this priority**: Habilita la analítica comparativa y la trazabilidad histórica a mediano y largo plazo, sin bloquear la operación diaria del módulo.

**Independent Test**: Con al menos dos Liquidaciones generadas en distintos momentos o galpones, se accede al historial, se aplican filtros por galpón y fecha, y se comprueba el listado ordenado descendentemente con la identificación clara del estado de cada Liquidación.

**Acceptance Scenarios**:

1. **Scenario**: Listado del historial con orden cronológico
   - **Given** existen Liquidaciones previas, tanto en estado `ACTIVA` como `ANULADA`
   - **When** el administrador financiero accede a la vista de historial
   - **Then** el sistema presenta el listado ordenado de forma descendente por fecha de generación (la más reciente primero), mostrando por cada fila: Fecha y Hora de Liquidación, Galpón / Lote, Venta Bruta (COP), Porcentaje de Mortalidad (%), Costos Operativos (COP), Utilidad Neta (COP) y Estado (`ACTIVA` | `ANULADA`)

2. **Scenario**: Filtrado por galpón y rango de fechas
   - **Given** un historial con múltiples Liquidaciones de diversos galpones a lo largo del año
   - **When** el usuario selecciona un galpón específico y define un rango de fechas
   - **Then** el sistema actualiza la grilla mostrando únicamente las Liquidaciones del galpón seleccionado dentro del intervalo indicado

3. **Scenario**: Visualización diferenciada de Liquidaciones anuladas
   - **Given** el historial contiene Liquidaciones activas y anuladas
   - **When** el usuario visualiza la tabla de resultados
   - **Then** las Liquidaciones anuladas se distinguen a simple vista mediante etiqueta o color diferenciado con el texto explícito "ANULADA", y permiten consultar el motivo, la fecha, la hora y el responsable de la anulación registrados en M3-CU04

4. **Scenario**: Historial sin registros o sin coincidencias de filtro
   - **Given** no se han generado Liquidaciones o los filtros aplicados no retornan coincidencias
   - **When** se ejecuta la consulta
   - **Then** el sistema muestra un estado vacío informativo diferenciando si es por ausencia total de datos ("No hay liquidaciones registradas aún") o por filtros ("No se encontraron liquidaciones para los criterios de búsqueda aplicados")

5. **Scenario**: Rango de fechas invertido
   - **Given** el usuario está configurando los filtros del historial
   - **When** ingresa una fecha de inicio posterior a la fecha de fin
   - **Then** el sistema valida en línea, alerta al usuario e impide ejecutar la consulta hasta corregir el intervalo

---

### Edge Cases

- **Navegación al detalle**: Al seleccionar cualquier registro del historial, el sistema permite abrir el desglose completo de esa Liquidación (M3-CU05).
- **Rendimiento con alto volumen**: El sistema mantiene paginación o carga eficiente para soportar historiales de hasta 24 meses con múltiples ciclos por galpón.
- **Varias Liquidaciones por lote**: Un lote puede aparecer varias veces en el historial si tuvo Liquidaciones anuladas, pero como máximo una de ellas estará en estado `ACTIVA`.
- **Lote desvinculado del galpón**: Las Liquidaciones de lotes ya desvinculados siguen siendo consultables y filtrables por galpón, porque la Liquidación conserva los UUID históricos de galpón y lote tomados de la alerta de vaciado sanitario.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST listar el historial de Liquidaciones ordenado descendentemente por fecha y hora de generación.
- **FR-002**: Cada fila MUST incluir: Fecha y Hora de Liquidación, Galpón / Lote, Venta Bruta (COP), Porcentaje de Mortalidad (%), Costos Operativos (COP), Utilidad Neta (COP) y Estado (`ACTIVA` | `ANULADA`).
- **FR-003**: El sistema MUST permitir filtrar las Liquidaciones por galpón específico (o todos) y por rango de fechas (desde – hasta).
- **FR-004**: El sistema MUST mostrar de forma clara y visible el estado de cada Liquidación, asegurando que las anuladas sean reconocibles de inmediato, y MUST permitir consultar su registro de anulación.
- **FR-005**: El sistema MUST validar que la fecha inicial del filtro sea menor o igual a la fecha final.
- **FR-006**: El sistema MUST permitir seleccionar un registro del historial para visualizar su desglose pormenorizado en M3-CU05.
- **FR-007**: Ante la ausencia de registros, el sistema MUST desplegar un mensaje descriptivo según el contexto (vacío total o sin resultados para los filtros).
- **FR-008**: El sistema MUST construir el historial exclusivamente a partir de las Liquidaciones almacenadas en M3, sin requerir consultas a Módulo 1 ni a Módulo 2.

### Key Entities

- **Liquidacion**: Única entidad consultada por esta especificación. Definida en M3-CU03. El historial es una **consulta filtrada y ordenada** sobre este conjunto.
- **RegistroAnulacion**: Auditoría de anulación consultada para las Liquidaciones en estado `ANULADA`. Definida en M3-CU04.

> **Nota**: Las entidades `HistorialLiquidacion` y `CriteriosFiltroHistorial` de la versión anterior de esta especificación se eliminan. La primera era una vista sobre `Liquidacion` y la segunda un conjunto de parámetros de búsqueda; ninguna constituye una entidad de dominio.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de respuesta de la consulta con filtros es inferior a 3 segundos con un volumen acumulado de hasta 24 meses de histórico.
- **SC-002**: 100% de cumplimiento del ordenamiento cronológico descendente por defecto.
- **SC-003**: 100% de precisión en los filtros por fecha y galpón (cero registros excluidos indebidamente o mostrados fuera de rango).
- **SC-004**: El 100% de las Liquidaciones anuladas muestra su estado de forma diferenciada e inequívoca y permite consultar su motivo de anulación.
- **SC-005**: El 100% de las consultas de historial se resuelve sin consultas a Módulo 1 ni a Módulo 2.
