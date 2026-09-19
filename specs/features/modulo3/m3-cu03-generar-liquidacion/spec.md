# Feature Specification: M3-CU03 – Generar Liquidación del Lote

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consolidación y Liquidación Financiera del Lote (Priority: P1)

Como administrador financiero, quiero generar la Liquidación económica definitiva de un lote calculando automáticamente la Venta Bruta, la Mortalidad del Lote, los Costos Operativos y la Utilidad Neta, para conocer la rentabilidad real y auditable del ciclo avícola.

**Why this priority**: Es el entregable central del Módulo 3 y el cálculo económico final del sistema. Determina si el lote generó ganancias o pérdidas reales.

**Independent Test**: Con una alerta de vaciado sanitario que identifica el lote, una Matriz de Ventas `ACTIVA` valorizada por M3 a partir del resultado final de sacrificio de Módulo 2 —o población actual igual a 0 por mortalidad total—, datos locales sincronizados completos y partidas de costo valorizadas del ciclo, se ejecuta la generación de la Liquidación y se comprueba que los cuatro indicadores cuadran con precisión de 100% contra el cálculo matemático verificado.

**Acceptance Scenarios**:

1. **Scenario**: Generación exitosa de Liquidación definitiva (Caso Dorado)
   - **Given** una alerta de vaciado sanitario que conserva los UUID de un galpón y su lote, con Población Inicial sincronizada de 9.000 aves y Población Actual sincronizada de 8.500 aves (origen M1), Matriz de Ventas `ACTIVA` valorizada por M3 desde un resultado final de M2 de 8.500 pollos y 23.800 kg totales a un precio de \$4.500 COP/kg, y Costos Operativos valorizados de \$85.000.000 COP (alimento, medicina y costo inicial de pollitos)
   - **When** el administrador financiero solicita generar la Liquidación
   - **Then** el sistema calcula y presenta:
     - **Venta Bruta**: \$107.100.000 COP (`23.800 kg × 4.500`)
     - **Mortalidad del Lote**: 500 aves (`9.000 − 8.500`), equivalente al 5,56% (`(500 / 9.000) × 100`)
     - **Costos Operativos**: \$85.000.000 COP
     - **Utilidad Neta**: \$22.100.000 COP (`107.100.000 − 85.000.000`)
     - Estado de la Liquidación: `ACTIVA`, con fecha, hora y usuario responsable

2. **Scenario**: Bloqueo por lote sin Matriz de Ventas registrada
   - **Given** una alerta de vaciado sanitario que identifica un lote con población actual mayor a 0 y sin Matriz de Ventas en estado `ACTIVA`
   - **When** se intenta generar la Liquidación del lote
   - **Then** el sistema bloquea la acción e informa: "El lote no cuenta con matriz de ventas activa. Debe registrarla (M3-CU02) antes de liquidar"

3. **Scenario**: Bloqueo por partidas de costo sin precio configurado
   - **Given** un lote con Matriz de Ventas `ACTIVA` pero con partidas de alimento o medicina sincronizadas desde Módulo 2 que carecen de precio unitario aplicable
   - **When** se solicita generar la Liquidación
   - **Then** el sistema detiene el proceso, no genera registros preliminares ni parciales, y muestra la lista detallada de partidas pendientes de valorización

4. **Scenario**: Ausencia total de partidas de costo sincronizadas
   - **Given** un lote con Matriz de Ventas `ACTIVA` para el cual M3 no cuenta con partidas de alimento ni de medicina sincronizadas desde Módulo 2
   - **When** se intenta generar la Liquidación
   - **Then** el sistema notifica "No hay partidas de costo sincronizadas para este ciclo" y mantiene bloqueada la Liquidación

5. **Scenario**: Mortalidad del 100% (siniestro total)
   - **Given** una alerta de vaciado sanitario que identifica un lote con 9.000 aves iniciales, población actual sincronizada igual a 0 por mortalidad total, ninguna Matriz de Ventas `ACTIVA` y partidas de costo del ciclo por \$40.000.000 COP
   - **When** se procesa la Liquidación de cierre
   - **Then** el sistema genera una Liquidación de siniestro total sin Matriz de Ventas, mostrando Venta Bruta = \$0 COP, Mortalidad del Lote = 9.000 aves (100%), Costos Operativos = \$40.000.000 COP y Utilidad Neta = −\$40.000.000 COP

6. **Scenario**: Intento de Liquidación antes del vaciado sanitario
   - **Given** un galpón en estado `En Cosecha` con una Matriz de Ventas `ACTIVA`
   - **When** el administrador intenta generar la Liquidación
   - **Then** el sistema bloquea la acción e informa que la Liquidación se habilita cuando Módulo 1 registre la alerta de vaciado sanitario tras retirar todas las aves

7. **Scenario**: Intento de segunda Liquidación activa sobre el mismo lote
   - **Given** un lote que ya cuenta con una Liquidación en estado `ACTIVA`
   - **When** el administrador intenta generar una nueva Liquidación sobre el mismo lote
   - **Then** el sistema deniega la acción, informa que el lote ya está liquidado y ofrece consultar la Liquidación existente o iniciar su anulación (M3-CU04)

8. **Scenario**: Indisponibilidad temporal de los módulos de origen
   - **Given** Módulo 1 o Módulo 2 no responden y M3 cuenta con los datos locales sincronizados requeridos para el lote
   - **When** el administrador solicita generar la Liquidación
   - **Then** el sistema genera la Liquidación usando esos datos, conserva la fecha y hora de sincronización de cada fuente en el snapshot financiero y no realiza consultas en vivo a los módulos de origen

---

### Edge Cases

- **Precisión de la Venta Bruta**: La Venta Bruta se calcula sobre el **peso total en kilogramos**, no sobre el peso promedio. El peso promedio es un valor derivado de presentación y no interviene en ningún cálculo monetario; usarlo introduciría error de redondeo cuando el cociente `pesoTotalKg / pollosVendidos` no es exacto.
- **Regla de redondeo monetario**: Las cifras monetarias totales se expresan como enteros aplicando redondeo `HALF_UP` (regla transversal RT-01).
- **Porcentaje de mortalidad**: Se calcula sobre la población inicial oficial de Módulo 1 y se presenta con 2 decimales (RT-02).
- **Mortalidad frente a aves no vendidas**: La mortalidad se calcula como la diferencia entre población inicial y población actual, ambas de Módulo 1. **No** se calcula restando los pollos vendidos, porque esa diferencia mezcla aves muertas con aves no comercializadas (descartes, decomisos) y produce una mortalidad sobreestimada.
- **Inmutabilidad absoluta (snapshot financiero)**: Una vez generada una Liquidación `ACTIVA`, sus valores quedan congelados. Si con posterioridad Módulo 1 o Módulo 2 corrigen sus cifras, la Liquidación existente no se altera; toda corrección exige anulación formal (M3-CU04) y una nueva Liquidación.
- **Exclusión de costos indirectos**: Administración, arrendamientos, nómina general y servicios quedan fuera del cálculo (RT-08).

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST calcular la **Venta Bruta** a partir de la Matriz de Ventas aplicando la fórmula:  
  `Venta Bruta = Peso Total kg (origen M2) × Precio por kg (M3)`, con redondeo `HALF_UP` a enteros.  
  El peso promedio por ave NO DEBE intervenir en este cálculo.
- **FR-002**: El sistema MUST calcular la **Mortalidad del Lote** aplicando las fórmulas:
  - `Mortalidad Absoluta = Población Inicial − Población Actual` (ambas sincronizadas desde Módulo 1)
  - `Porcentaje Mortalidad = (Mortalidad Absoluta / Población Inicial) × 100`, con 2 decimales.
  
  En una Liquidación de siniestro total la población actual es 0, por lo que la mortalidad absoluta equivale a la población inicial y el porcentaje es 100,00%.
- **FR-003**: El sistema MUST consolidar los **Costos Operativos** exclusivamente como la suma valorizada de:
  - **Costo de Alimento**: partidas de alimento del ciclo sincronizadas desde Módulo 2 (M3-CU09), valorizadas por M3 como `cantidad × precio unitario`.  
    [NEEDS CLARIFICATION – D-01: el criterio de la cantidad (comprada, consumida o requerida según proyección) está pendiente de acuerdo con Módulo 2. Ver M3-CU09.]
  - **Insumos Médicos**: consumos de medicamento del ciclo sincronizados desde Módulo 2 (M3-CU10), valorizados por M3 como `cantidad en unidad base × precio histórico de la recepción correspondiente`, sin promediar precios entre recepciones.
  - **Costo de Población**: costo total del lote informado por Módulo 1 (M3-CU07), no acumulativo. Es la única fuente admitida para este valor.
  
  [NEEDS CLARIFICATION – D-05: tratamiento del impuesto informado por Módulo 2 pendiente de definición.]
- **FR-004**: El sistema MUST calcular la **Utilidad Neta** aplicando la fórmula:  
  `Utilidad Neta = Venta Bruta − Costos Operativos`.
- **FR-005**: El sistema MUST exigir una alerta de vaciado sanitario registrada por Módulo 1, que conserve los UUID del galpón y del lote, como condición previa obligatoria para liquidar. Además MUST existir una Matriz de Ventas `ACTIVA`, excepto cuando la población actual sincronizada del lote sea 0 por mortalidad total; en ese caso MUST permitir una Liquidación de siniestro total sin Matriz de Ventas, con `ventaBruta = 0` y `pollosVendidos = 0`.  
  [NEEDS CLARIFICATION – D-04: Módulo 1 solo admite la transición a `Vaciado sanitario` desde `En cosecha`, por lo que un lote con mortalidad total en estado `Productivo` no alcanza la precondición de este requisito. Ruta de estado pendiente de acuerdo con Módulo 1.]
- **FR-006**: El sistema MUST impedir la Liquidación y listar las partidas faltantes si alguna partida de alimento o medicina del ciclo carece de precio aplicable. No se permiten liquidaciones preliminares ni parciales.
- **FR-007**: El sistema MUST impedir la existencia de más de una Liquidación en estado `ACTIVA` por lote.
- **FR-008**: El sistema MUST guardar la Liquidación con estado `ACTIVA`, fecha y hora de generación y usuario responsable. Toda Liquidación `ACTIVA` es inmutable; su corrección se ejecuta exclusivamente mediante el flujo de anulación definido en **M3-CU04 – Anular Documento Financiero**.
- **FR-009**: El sistema MUST generar la Liquidación exclusivamente con datos locales sincronizados cuyo origen sea Módulo 1 (M3-CU07) o Módulo 2 (M3-CU08, M3-CU09, M3-CU10), y MUST conservar en el snapshot financiero la fecha y hora de sincronización de cada fuente. La generación NO DEBE requerir consultas en vivo a los módulos de origen.
- **FR-010**: Al persistir una Liquidación `ACTIVA` que consume una Matriz de Ventas, el sistema MUST marcar localmente el resultado final de sacrificio asociado como utilizado y MUST emitir el aviso de utilización definido en **M3-CU08**.

### Key Entities

- **Liquidacion**: Resultado financiero consolidado del lote. Atributos clave: `idLiquidacion`, `idLote`, `idVenta` (nulo únicamente en siniestro total), `ventaBrutaCop`, `mortalidadAves`, `porcentajeMortalidad`, `costosOperativosCop`, `utilidadNetaCop`, `estado` (`ACTIVA` | `ANULADA`), `fechaHoraGeneracion`, `usuarioResponsable`.
- **PartidaCostoLote**: Partida de costo sincronizada y valorizada asociada al ciclo del lote. Atributos: `idPartida`, `idLote`, `categoria` (`ALIMENTO` | `MEDICINA` | `POBLACION`), `concepto`, `cantidad`, `unidadMedida`, `precioUnitarioCop`, `subtotalCop`, `fuenteOrigen`, `referenciaOrigen`.
- **SnapshotDatosOrigen**: Datos sincronizados utilizados para una Liquidación. Atributos: `idLiquidacion`, `idLote`, `fuente` (`MODULO_1` | `MODULO_2`), `fechaHoraSincronizacion` y referencias a las poblaciones y partidas de costo empleadas.
- **AlertaVaciadoSanitario**: Evento registrado por Módulo 1 que conserva `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento` después de desvincular el lote del galpón; es la referencia que identifica al lote liquidable.

> **Nota**: La entidad `RegistroAnulacionLiquidacion` de la versión anterior de esta especificación se traslada a **M3-CU04** como `RegistroAnulacion`, unificada con la anulación de Matriz de Ventas.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las Liquidaciones de prueba coincide con exactitud al peso contra el cálculo aritmético manual verificado.
- **SC-002**: El tiempo total de procesamiento y despliegue de la Liquidación en pantalla es inferior a 5 segundos tras pulsar "Generar Liquidación".
- **SC-003**: 0% de Liquidaciones generadas con partidas de costo sin valorizar o sin Matriz de Ventas, excepto las Liquidaciones de siniestro total autorizadas con población actual igual a 0.
- **SC-004**: 100% de inmutabilidad: ninguna Liquidación `ACTIVA` cambia su valor ante modificaciones posteriores de los datos de origen sin anulación previa.
- **SC-005**: El 100% de las Liquidaciones generadas conserva la fecha y hora de sincronización de cada fuente utilizada en el cálculo.
- **SC-006**: 0% de lotes con más de una Liquidación en estado `ACTIVA`.
