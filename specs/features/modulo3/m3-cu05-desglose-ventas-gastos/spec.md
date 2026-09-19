# Feature Specification: M3-CU05 – Consultar Desglose de Ventas y Gastos

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Auditoría y Desglose Detallado de Partidas (Priority: P2)

Como administrador financiero, quiero consultar en pantalla y exportar a Excel el desglose pormenorizado de las partidas de ingreso y de costo de una Liquidación, para auditar el origen exacto de cada cifra antes de tomar decisiones de negocio.

**Why this priority**: Aporta transparencia y trazabilidad sobre los costos consolidados en M3-CU03. No bloquea la operación diaria, pero es indispensable para la auditoría.

**Independent Test**: Se abre el desglose de una Liquidación `ACTIVA` existente y se verifica que cada subtotal por partida suma con exactitud al peso el valor de Costos Operativos, y que la exportación a Excel genera un archivo idéntico a lo mostrado en pantalla.

**Acceptance Scenarios**:

1. **Scenario**: Visualización del desglose completo por categorías
   - **Given** un lote con Liquidación en estado `ACTIVA` que incluye partidas de alimento (Pre-inicio, Inicio, Engorde/Broiler), partidas de insumos médicos y costo inicial de adquisición de aves
   - **When** el administrador financiero accede a la vista de desglose
   - **Then** el sistema presenta una tabla estructurada por categorías donde cada fila detalla: Concepto, Cantidad, Unidad de medida, Precio unitario aplicado (COP) y Subtotal valorizado (COP)
   - **And** la suma de los subtotales de egreso coincide con exactitud al peso con el valor de Costos Operativos de la Liquidación

2. **Scenario**: Exportación exitosa a archivo Excel
   - **Given** se visualiza en pantalla el desglose de una Liquidación `ACTIVA`
   - **When** el usuario presiona "Exportar a Excel"
   - **Then** el sistema genera y descarga un archivo `.xlsx` que contiene el resultado final de sacrificio de Módulo 2, la Matriz de Ventas, los indicadores consolidados (Venta Bruta, Mortalidad del Lote, Costos Operativos, Utilidad Neta) y la tabla de desglose completa con los mismos valores de pantalla

3. **Scenario**: Consulta de desglose sobre una Liquidación anulada
   - **Given** una Liquidación en estado `ANULADA`
   - **When** el usuario consulta su desglose
   - **Then** el sistema muestra los datos en modo exclusivamente de solo lectura con un aviso visual destacado "LIQUIDACIÓN ANULADA", presenta el motivo, fecha, hora y responsable de la anulación registrados en M3-CU04, y mantiene deshabilitada la exportación a Excel

4. **Scenario**: Error durante la generación del archivo Excel
   - **Given** el usuario solicita la exportación y ocurre una falla de entrada/salida o de memoria
   - **When** se interrumpe la exportación
   - **Then** el sistema notifica el error mediante un mensaje claro y ofrece reintentar, sin alterar ni perder la vista de datos en pantalla

5. **Scenario**: Desglose de una Liquidación de siniestro total
   - **Given** una Liquidación `ACTIVA` de siniestro total, sin Matriz de Ventas asociada
   - **When** el usuario consulta su desglose
   - **Then** el sistema presenta las partidas de costo del ciclo y muestra la sección de ingresos con Venta Bruta igual a \$0 COP, indicando explícitamente que el lote no registró venta

---

### Edge Cases

- **El desglose no es un documento**: El desglose es una **proyección de solo lectura** derivada de la Liquidación y de sus partidas de costo. No se almacena como registro independiente ni tiene estado propio; por lo tanto no puede desincronizarse de la Liquidación que representa.
- **Trazabilidad de origen (≤ 3 clics)**: Cada línea del desglose debe indicar la referencia de su fuente sincronizada (ej. "Módulo 2 / Consumo de medicamento", "Módulo 1 / Costo total del lote"), permitiendo verificar el sustento del dato sin consulta en vivo al módulo de origen.
- **Formato numérico en Excel**: Los montos en COP se exportan con formato monetario estándar sin decimales en totales, preservando la fórmula de suma en las celdas de subtotal.
- **Exportación concurrente**: Varios usuarios exportando Liquidaciones al mismo tiempo no deben experimentar bloqueos ni corrupción de archivos.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar el desglose de la Liquidación agrupado en las siguientes categorías:
  - **Ingresos**: datos de la Matriz de Ventas (pollos vendidos, peso total kg, peso promedio kg, precio por kg y Venta Bruta). En siniestro total, la sección se presenta con Venta Bruta igual a \$0 COP.
  - **Alimento**: detalle por partida y tipo (Pre-inicio, Inicio, Engorde, etc.) con cantidad, precio aplicado por kg y subtotal en COP.
  - **Insumos Médicos**: detalle por consumo de medicamento o vacuna con cantidad en unidad base, unidad de medida, precio unitario histórico y subtotal en COP.
  - **Población Inicial**: costo total del lote informado por Módulo 1, no acumulativo.
- **FR-002**: La suma de los subtotales de todas las partidas de costo MUST ser idéntica al peso al campo `costosOperativosCop` de la Liquidación (cero discrepancias).
- **FR-003**: El sistema MUST proveer exportación a formato Excel (`.xlsx`) con la información del lote, el resultado final de sacrificio, la Matriz de Ventas, los indicadores consolidados y el desglose de costos.
- **FR-004**: Para Liquidaciones en estado `ANULADA`, el sistema MUST restringir la vista a solo lectura, presentar un encabezado visual prominente de anulación con los datos del registro de anulación (M3-CU04) y deshabilitar la exportación a Excel.
- **FR-005**: Ante fallas en la exportación, el sistema MUST capturar el error, informar al usuario y habilitar el reintento sin degradar la sesión activa.
- **FR-006**: El sistema MUST construir el desglose exclusivamente a partir de la Liquidación y de sus partidas de costo almacenadas en M3, sin requerir consultas a Módulo 1 ni a Módulo 2.
- **FR-007**: El sistema NO DEBE almacenar el desglose como entidad independiente. Es una proyección derivada calculada en el momento de la consulta.

### Key Entities

- **Liquidacion**: Documento de resultado consultado por esta especificación. Definida en M3-CU03.
- **PartidaCostoLote**: Partida de costo valorizada asociada al ciclo del lote, origen de cada fila del desglose. Definida en M3-CU03.
- **MatrizVentas**: Documento de ingreso, origen de la sección de ingresos del desglose. Definida en M3-CU02.
- **ArchivoExportacion**: Documento generado en formato `.xlsx`, con metadatos de exportación (`nombreArchivo`, `fechaHoraExportacion`, `usuarioGenerador`). No contiene datos financieros propios.

> **Nota**: La entidad `DesgloseLiquidacion` de la versión anterior de esta especificación se elimina. El desglose se modela como proyección derivada para evitar la duplicación de datos y el riesgo de descuadre frente a la Liquidación (FR-002).

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% de coincidencia numérica entre los subtotales del desglose en pantalla, el total de Costos Operativos de la Liquidación y el archivo Excel generado.
- **SC-002**: La generación y descarga del archivo Excel se completa en menos de 10 segundos para cualquier ciclo histórico.
- **SC-003**: Toda partida del desglose es trazable a su registro fuente en un máximo de 3 pasos de navegación.
- **SC-004**: 0% de exportaciones permitidas sobre Liquidaciones en estado `ANULADA`.
- **SC-005**: El 100% de las consultas de desglose se resuelve sin consultas a Módulo 1 ni a Módulo 2.
