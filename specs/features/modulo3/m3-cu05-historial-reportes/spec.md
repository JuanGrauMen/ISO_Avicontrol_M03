# Feature Specification: M3-CU05 – Consultar Historial de Reportes Financieros

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta Histórica y Comparabilidad de Ciclos (Priority: P3)

Como administrador financiero, quiero consultar el historial cronológico de todas las liquidaciones generadas filtrando por galpón y rango de fechas, para comparar los resultados de rentabilidad y mortalidad entre ciclos y tomar decisiones de mejora operativa.

**Why this priority**: Permite la analítica comparativa y la trazabilidad histórica del negocio a mediano y largo plazo, sin bloquear la operación diaria del módulo.

**Independent Test**: Habiendo al menos dos liquidaciones generadas en distintos momentos o galpones, se accede al historial, se aplican filtros por galpón y fecha, y se comprueba el listado ordenado descendentemente con la identificación clara del estado de cada liquidación.

**Acceptance Scenarios**:

1. **Scenario**: Listado de historial con registros y orden cronológico
   - **Given** existen liquidaciones previas (tanto en estado `ACTIVA` como `ANULADA`)
   - **When** el administrador financiero accede a la vista de historial de reportes
   - **Then** el sistema presenta el listado ordenado de forma descendente por fecha de generación (la más reciente primero), mostrando por cada fila: Fecha de Liquidación, Identificador de Galpón/Lote, Venta Bruta (COP), % de Mortalidad, Costos Operativos (COP), Utilidad Neta (COP) y Estado (`ACTIVA` / `ANULADA`)

2. **Scenario**: Filtrado por galpón y rango de fechas
   - **Given** un historial con múltiples liquidaciones de diversos galpones a lo largo del año
   - **When** el usuario selecciona un galpón específico y define un rango de fechas (fecha inicio y fecha fin)
   - **Then** el sistema actualiza la grilla mostrando únicamente las liquidaciones que corresponden al galpón seleccionado dentro del intervalo temporal indicado

3. **Scenario**: Visualización diferenciada de liquidaciones ANULADAS
   - **Given** el historial contiene liquidaciones activas y anuladas
   - **When** el usuario visualiza la tabla de resultados
   - **Then** las liquidaciones anuladas se distinguen a simple vista mediante una etiqueta o color diferenciado (ej. color gris/rojo con texto explícito "ANULADA") para evitar confusiones financieras

4. **Scenario**: Historial sin registros (Estado inicial o sin coincidencias de filtro)
   - **Given** no se han generado liquidaciones en el sistema o los filtros aplicados no retornan coincidencias
   - **When** se ejecuta la consulta
   - **Then** el sistema muestra un estado vacío informativo diferenciando si es por ausencia total de datos ("No hay reportes de liquidación registrados aún") o por filtros ("No se encontraron liquidaciones para los criterios de búsqueda aplicados")

---

### Edge Cases

- **Navegación al detalle**: Al hacer clic sobre cualquier registro del historial, el sistema permite abrir el desglose completo de esa liquidación (M3-CU04).
- **Rango de fechas invertido**: Si el usuario ingresa una fecha de inicio posterior a la fecha de fin, el sistema valida en línea y alerta al usuario impidiendo la consulta hasta corregir el intervalo.
- **Rendimiento con alto volumen**: El sistema mantiene paginación o carga eficiente de registros para soportar historiales de hasta 24 meses (múltiples ciclos por galpón).

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST listar el historial de liquidaciones ordenado descendentemente por fecha y hora de generación.
- **FR-002**: Cada fila del historial MUST incluir: Fecha y Hora de Liquidación, Galpón / Lote, Venta Bruta (COP), Porcentaje de Mortalidad (%), Costos Operativos Totales (COP), Utilidad Neta (COP) y Estado (`ACTIVA` | `ANULADA`).
- **FR-003**: El sistema MUST permitir filtrar las liquidaciones por:
  - Galpón específico (o todos).
  - Rango de fechas (Fecha Desde - Fecha Hasta).
- **FR-004**: El sistema MUST mostrar de forma clara y visible el estado de cada liquidación, asegurando que las liquidaciones `ANULADA` sean reconocibles de inmediato.
- **FR-005**: El sistema MUST validar que la fecha inicial del filtro sea menor o igual a la fecha final.
- **FR-006**: El sistema MUST permitir seleccionar un registro del historial para visualizar su desglose pormenorizado en M3-CU04.
- **FR-007**: Ante la ausencia de registros, el sistema MUST desplegar un mensaje descriptivo según el contexto (vacío total o sin resultados para los filtros).

### Key Entities

- **HistorialLiquidacion**: Vista consolidada de registros de liquidación. Atributos: `idLiquidacion`, `idLote`, `idGalpon`, `fechaGeneracion`, `ventaBrutaCop`, `porcentajeMortalidad`, `costosOperativosCop`, `utilidadNetaCop`, `estado`, `usuarioResponsable`.
- **CriteriosFiltroHistorial**: Parámetros de búsqueda (`idGalpon`, `fechaDesde`, `fechaHasta`).

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de respuesta de la consulta del historial con filtros es inferior a 3 segundos con un volumen acumulado de hasta 24 meses de histórico.
- **SC-002**: 100% de cumplimiento del ordenamiento cronológico descendente por defecto.
- **SC-003**: 100% de precisión en los filtros por fecha y galpón (cero registros excluidos indebidamente o mostrados fuera de rango).
- **SC-004**: El 100% de las liquidaciones anuladas muestran su estado de forma diferenciada e inequívoca.
