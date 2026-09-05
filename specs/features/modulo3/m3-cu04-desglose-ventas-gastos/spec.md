# Feature Specification: M3-CU04 – Desglose de Ventas y Gastos

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Auditoría y Desglose Detallado de Partidas (Priority: P2)

Como administrador financiero, quiero consultar en pantalla y exportar a Excel el desglose pormenorizado de las partidas de ingresos y costos operativos (resultado operativo de sacrificio, compras de alimento, compras de medicamentos y costo inicial de pollitos) de una liquidación, para auditar el origen exacto de cada cifra antes de tomar decisiones de negocio.

**Why this priority**: Proporciona total transparencia, trazabilidad financiera y justificación detallada de los costos consolidados generados en M3-CU03.

**Independent Test**: Se abre la consulta de una liquidación activa existente almacenada en M3, se verifica que cada subtotal por partida suma con exactitud al peso el valor de Costos Operativos y que la exportación a Excel genera un archivo idéntico a lo mostrado en pantalla.

**Acceptance Scenarios**:

1. **Scenario**: Visualización del desglose completo por categorías
   - **Given** un lote con liquidación en estado `ACTIVA` que incluye compras de alimento (Pre-inicio, Inicio, Engorde/Broiler), compras de insumos médicos y costo inicial de adquisición de aves
   - **When** el administrador financiero accede a la vista de desglose detallado
   - **Then** el sistema presenta una tabla estructurada por categorías donde cada fila detalla: Concepto, Cantidad comprada/vendida, Unidad de medida, Precio unitario aplicado (COP) y Subtotal valorizado (COP)
   - **And** la suma total de los subtotales de egreso coincide con exactitud matemática al peso con el valor de Costos Operativos de la liquidación

2. **Scenario**: Exportación exitosa a archivo Excel
   - **Given** se visualiza en pantalla el desglose de una liquidación activa
   - **When** el usuario presiona el botón "Exportar a Excel"
   - **Then** el sistema genera y descarga un archivo `.xlsx` que contiene el resultado final de sacrificio de Módulo 2, la matriz comercial valorizada por M3, los indicadores consolidados (Venta Bruta, Mortalidad, Utilidad Neta) y la tabla de desglose completa con los mismos valores de pantalla

3. **Scenario**: Consulta de desglose sobre liquidación ANULADA
   - **Given** una liquidación que fue marcada en estado `ANULADA`
   - **When** el usuario consulta su desglose detallado
   - **Then** el sistema muestra los datos en modo exclusivamente de solo lectura, con una marca de agua/aviso visual destacado "LIQUIDACIÓN ANULADA", y mantiene deshabilitada la función de exportación a Excel para evitar emisión de reportes no válidos

4. **Scenario**: Error durante la generación del archivo Excel
   - **Given** el usuario solicita la exportación a Excel pero ocurre una falla de E/S o memoria
   - **When** se interrumpe la exportación
   - **Then** el sistema notifica el error mediante un mensaje claro ofreciendo reintentar la acción, sin alterar ni perder la vista de datos en pantalla

---

### Edge Cases

- **Trazabilidad de origen (≤ 3 clics)**: Cada línea del desglose debe indicar la referencia de su fuente original sincronizada (ej. "Módulo 2 - Compra de alimento", "Módulo 1 / Costo inicial de pollitos") permitiendo al auditor verificar el sustento del dato sin requerir una consulta en vivo al módulo de origen.
- **Formato numérico en Excel**: Los montos en pesos colombianos (COP) en el archivo Excel deben exportarse con formato numérico monetario estándar sin decimales en totales, preservando la fórmula de suma en las celdas de subtotal.
- **Exportación concurrente**: Múltiples usuarios exportando liquidaciones al mismo tiempo no deben experimentar bloqueos ni corrupción de archivos.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar el desglose pormenorizado de la liquidación agrupado en las siguientes categorías de costos operativos:
  - **Alimento**: Detalle por compra y tipo (Pre-inicio, Inicio, Engorde, etc.) con kg comprados, precio aplicado por kg y subtotal en COP.
  - **Insumos Médicos**: Detalle por compra de medicamento/vacuna con cantidad comprada, unidad de medida, tarifa unitaria y subtotal en COP.
  - **Población Inicial**: Costo inicial no acumulativo de adquisición de pollitos informado por Módulo 1.
- **FR-002**: La suma de los subtotales de todas las partidas MUST ser idéntica al peso con el campo `costosOperativos` de la liquidación (cero discrepancias).
- **FR-003**: El sistema MUST proveer la funcionalidad de exportación a formato Excel (`.xlsx`), estructurando la información del lote, el resultado final de sacrificio de Módulo 2, la matriz comercial valorizada por M3, el resumen de rentabilidad y el desglose de costos.
- **FR-004**: Para liquidaciones en estado `ANULADA`, el sistema MUST restringir la vista a solo lectura, presentar un encabezado visual prominente de anulación y deshabilitar el botón de exportación a Excel.
- **FR-005**: Ante fallas en la exportación a Excel, el sistema MUST capturar la excepción, informar al usuario y habilitar el reintento sin degradar la sesión activa.
- **FR-006**: El sistema MUST consultar y exportar exclusivamente el snapshot financiero y el desglose almacenados en M3, sin requerir consultas en vivo a Módulo 1 ni Módulo 2.

### Key Entities

- **DesgloseLiquidacion**: Estructura de detalle asociada a una liquidación. Atributos clave: `idLiquidacion`, `categoria` (Alimento | Medicina | Población), `concepto`, `cantidad`, `unidadMedida`, `precioUnitarioCop`, `subtotalCop`, `fuenteOrigen`.
- **ArchivoExportacion**: Documento generado en formato `.xlsx` con metadatos de exportación (`nombreArchivo`, `fechaExportacion`, `usuarioGenerador`).

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% de coincidencia numérica entre los subtotales del desglose en pantalla, el total de costos operativos y el archivo Excel generado.
- **SC-002**: La generación y descarga del archivo Excel se completa en menos de 10 segundos para cualquier ciclo histórico.
- **SC-003**: Toda partida del desglose es trazable a su registro fuente en un máximo de 3 pasos de navegación.
- **SC-004**: 0% de exportaciones permitidas sobre liquidaciones en estado `ANULADA`.
