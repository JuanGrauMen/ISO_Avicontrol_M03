# Feature Specification: M3-CU03 – Generar Liquidación del Lote

**Created**: 2026-08-31  
**Actualizado**: 2026-09-21  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consolidación y Liquidación Financiera del Lote (Priority: P1)

Como administrador financiero, quiero generar la Liquidación económica definitiva de un lote ingresando el precio por kilogramo y dejando que el sistema calcule automáticamente la Venta Bruta, la Mortalidad del Lote, los Costos Operativos y la Utilidad Neta a partir del resultado final de sacrificio de Módulo 2 y los costos sincronizados del ciclo, para conocer la rentabilidad real y auditable del ciclo avícola presentada como Matriz de Venta Final.

**Why this priority**: Es el entregable central del Módulo 3 y el cálculo económico final del sistema. Determina si el lote generó ganancias o pérdidas reales.

**Independent Test**: Con una alerta de vaciado sanitario que identifica el lote, un resultado final de sacrificio sincronizado desde Módulo 2 —o población actual igual a 0 por mortalidad total—, datos locales sincronizados completos y partidas de costo valorizadas del ciclo, el administrador ingresa un precio por kg válido, se ejecuta la generación de la Liquidación y se comprueba que los cuatro indicadores cuadran con precisión de 100% contra el cálculo matemático verificado.

**Acceptance Scenarios**:

1. **Scenario**: Generación exitosa de Liquidación definitiva (Caso Dorado)
   - **Given** una alerta de vaciado sanitario que conserva los UUID de un galpón y su lote, con Población Inicial sincronizada de 9.000 aves y Población Actual sincronizada de 8.500 aves (origen M1), un resultado final de sacrificio sincronizado desde M2 de 8.500 pollos y 23.800 kg totales, y Costos Operativos valorizados de \$85.000.000 COP (alimento, medicina y costo inicial de pollitos)
   - **When** el administrador financiero ingresa un precio de \$4.500 COP/kg y solicita generar la Liquidación
   - **Then** el sistema calcula y presenta la Matriz de Venta Final:
     - **Venta Bruta**: \$107.100.000 COP (`23.800 kg × 4.500`)
     - **Mortalidad del Lote**: 500 aves (`9.000 − 8.500`), equivalente al 5,56% (`(500 / 9.000) × 100`)
     - **Costos Operativos**: \$85.000.000 COP
     - **Utilidad Neta**: \$22.100.000 COP (`107.100.000 − 85.000.000`)
     - Datos de venta: 8.500 pollos vendidos, 23.800 kg, peso promedio 2,8 kg, \$4.500 COP/kg
     - Estado de la Liquidación: `ACTIVA`, con fecha, hora y usuario responsable

2. **Scenario**: Bloqueo por lote sin resultado final de sacrificio sincronizado
   - **Given** una alerta de vaciado sanitario que identifica un lote con población actual mayor a 0 y sin resultado final de sacrificio válido sincronizado desde Módulo 2
   - **When** se intenta generar la Liquidación del lote
   - **Then** el sistema bloquea la acción e informa: "El lote no cuenta con resultado final de sacrificio sincronizado desde Módulo 2. No es posible liquidar"

3. **Scenario**: Rechazo por precio por kg no positivo
   - **Given** un lote liquidable con resultado final de sacrificio válido
   - **When** el administrador ingresa un precio por kg menor o igual a 0 COP
   - **Then** el sistema rechaza el formulario, no genera la Liquidación e indica explícitamente el campo con valor no permitido

4. **Scenario**: Bloqueo por partidas de costo sin precio configurado
   - **Given** un lote liquidable con partidas de alimento o medicina sincronizadas desde Módulo 2 que carecen de precio unitario aplicable
   - **When** se solicita generar la Liquidación
   - **Then** el sistema detiene el proceso, no genera registros preliminares ni parciales, y muestra la lista detallada de partidas pendientes de valorización

5. **Scenario**: Ausencia total de partidas de costo sincronizadas
   - **Given** un lote liquidable para el cual M3 no cuenta con partidas de alimento ni de medicina sincronizadas desde Módulo 2
   - **When** se intenta generar la Liquidación
   - **Then** el sistema notifica "No hay partidas de costo sincronizadas para este ciclo" y mantiene bloqueada la Liquidación

6. **Scenario**: Mortalidad del 100% (siniestro total)
   - **Given** una alerta de vaciado sanitario que identifica un lote con 9.000 aves iniciales, población actual sincronizada igual a 0 por mortalidad total, ningún resultado final de sacrificio y partidas de costo del ciclo por \$40.000.000 COP
   - **When** se procesa la Liquidación de cierre
   - **Then** el sistema genera una Liquidación de siniestro total sin exigir precio por kg, mostrando Venta Bruta = \$0 COP, pollos vendidos = 0, Mortalidad del Lote = 9.000 aves (100%), Costos Operativos = \$40.000.000 COP y Utilidad Neta = −\$40.000.000 COP

7. **Scenario**: Intento de Liquidación antes del vaciado sanitario
   - **Given** un galpón en estado `En Cosecha` con resultado final de sacrificio sincronizado
   - **When** el administrador intenta generar la Liquidación
   - **Then** el sistema bloquea la acción e informa que la Liquidación se habilita cuando Módulo 1 registre la alerta de vaciado sanitario tras retirar todas las aves

8. **Scenario**: Intento de segunda Liquidación activa sobre el mismo lote
   - **Given** un lote que ya cuenta con una Liquidación en estado `ACTIVA`
   - **When** el administrador intenta generar una nueva Liquidación sobre el mismo lote
   - **Then** el sistema deniega la acción, informa que el lote ya está liquidado y ofrece consultar la Liquidación existente o iniciar su anulación (M3-CU04)

9. **Scenario**: Generación tras anulación de la Liquidación anterior
   - **Given** un lote cuya Liquidación previa fue anulada formalmente mediante M3-CU04 por un error de precio
   - **When** el administrador genera una nueva Liquidación con el precio por kg corregido
   - **Then** el sistema crea la nueva Liquidación en estado `ACTIVA` y conserva la anterior en estado `ANULADA` para auditoría

10. **Scenario**: Indisponibilidad temporal de los módulos de origen
    - **Given** Módulo 1 o Módulo 2 no responden y M3 cuenta con los datos locales sincronizados requeridos para el lote
    - **When** el administrador solicita generar la Liquidación
    - **Then** el sistema genera la Liquidación usando esos datos, conserva la fecha y hora de sincronización de cada fuente en el snapshot financiero y no realiza consultas en vivo a los módulos de origen

---

### Edge Cases

- **Matriz de Venta Final**: Es la presentación de la Liquidación (glosario del módulo). No se registra por separado ni tiene estado propio; cualquier referencia a "registrar la matriz" es una referencia a generar la Liquidación.
- **Precisión de la Venta Bruta**: La Venta Bruta se calcula sobre el **peso total en kilogramos**, no sobre el peso promedio. El peso promedio es un valor derivado de presentación y no interviene en ningún cálculo monetario; usarlo introduciría error de redondeo cuando el cociente `pesoTotalKg / pollosVendidos` no es exacto.
- **Precio con decimales**: El precio por kg admite valores con decimales (ej. \$4.500,50 COP). La cantidad vendida y el peso total provienen del resultado final de sacrificio de Módulo 2; M3 no los digita ni los modifica.
- **Regla de redondeo monetario**: Las cifras monetarias totales se expresan como enteros aplicando redondeo `HALF_UP` (regla transversal RT-01).
- **Porcentaje de mortalidad**: Se calcula sobre la población inicial oficial de Módulo 1 y se presenta con 2 decimales (RT-02).
- **Mortalidad frente a aves no vendidas**: La mortalidad se calcula como la diferencia entre población inicial y población actual, ambas de Módulo 1. **No** se calcula restando los pollos vendidos, porque esa diferencia mezcla aves muertas con aves no comercializadas (descartes, decomisos) y produce una mortalidad sobreestimada.
- **Inmutabilidad absoluta (snapshot financiero)**: Una vez generada una Liquidación `ACTIVA`, sus valores quedan congelados, incluido el precio por kg ingresado. Si con posterioridad Módulo 1 o Módulo 2 corrigen sus cifras, la Liquidación existente no se altera; toda corrección exige anulación formal (M3-CU04) y una nueva Liquidación.
- **Liquidación en galpón equivocado**: No se permite edición silenciosa. El usuario debe anular formalmente la Liquidación mediante M3-CU04 y generarla sobre el lote correcto.
- **Exclusión de costos indirectos**: Administración, arrendamientos, nómina general y servicios quedan fuera del cálculo (RT-08).

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST calcular la **Venta Bruta** aplicando la fórmula:  
  `Venta Bruta = Peso Total kg (origen M2, M3-CU08) × Precio por kg (ingresado en M3)`, con redondeo `HALF_UP` a enteros.  
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
- **FR-005**: El sistema MUST exigir una alerta de vaciado sanitario registrada por Módulo 1, que conserve los UUID del galpón y del lote, como condición previa obligatoria para liquidar. Además MUST existir un resultado final de sacrificio válido sincronizado desde Módulo 2 (M3-CU08), excepto cuando la población actual sincronizada del lote sea 0 por mortalidad total; en ese caso MUST permitir una Liquidación de siniestro total sin resultado de sacrificio ni precio por kg, con `ventaBruta = 0` y `pollosVendidos = 0`.  
  [NEEDS CLARIFICATION – D-04: Módulo 1 solo admite la transición a `Vaciado sanitario` desde `En cosecha`, por lo que un lote con mortalidad total en estado `Productivo` no alcanza la precondición de este requisito. Ruta de estado pendiente de acuerdo con Módulo 1.]
- **FR-006**: El sistema MUST impedir la Liquidación y listar las partidas faltantes si alguna partida de alimento o medicina del ciclo carece de precio aplicable. No se permiten liquidaciones preliminares ni parciales.
- **FR-007**: El sistema MUST impedir la existencia de más de una Liquidación en estado `ACTIVA` por lote.
- **FR-008**: El sistema MUST guardar la Liquidación con estado `ACTIVA`, fecha y hora de generación y usuario responsable. Toda Liquidación `ACTIVA` es inmutable; su corrección se ejecuta exclusivamente mediante el flujo de anulación definido en **M3-CU04 – Anular Liquidación**. El lote solo admite una nueva Liquidación cuando la anterior haya sido anulada formalmente.
- **FR-009**: El sistema MUST generar la Liquidación exclusivamente con datos locales sincronizados cuyo origen sea Módulo 1 (M3-CU07) o Módulo 2 (M3-CU08, M3-CU09, M3-CU10), y MUST conservar en el snapshot financiero la fecha y hora de sincronización de cada fuente. La generación NO DEBE requerir consultas en vivo a los módulos de origen. Si no existen las copias locales requeridas, el sistema MUST bloquear la generación e informar que no hay datos disponibles.
- **FR-010**: Al persistir una Liquidación `ACTIVA` que consume un resultado final de sacrificio, el sistema MUST marcar localmente ese resultado como utilizado y MUST emitir el aviso de utilización definido en **M3-CU08**.
- **FR-011**: El sistema MUST capturar manualmente, como único dato de entrada del administrador financiero, el **precio por kilogramo** en pesos colombianos (decimal positivo mayor que 0). MUST rechazar un precio menor o igual a cero.
- **FR-012**: El sistema MUST tomar la cantidad final de pollos vendidos y el peso total exclusivamente del resultado final de sacrificio sincronizado (M3-CU08), MUST calcular `pesoPromedioKg = pesoTotalKg / pollosVendidos` solo con fines de presentación y NO DEBE editar ninguno de los valores operativos de origen.
- **FR-013**: El sistema MUST presentar la Liquidación generada como **Matriz de Venta Final**: los cuatro indicadores (Venta Bruta, Mortalidad del Lote, Costos Operativos, Utilidad Neta) y los datos de venta que los sustentan (pollos vendidos, peso total, peso promedio, precio por kg).

### Key Entities

- **Liquidacion**: Resultado financiero consolidado del lote y único Documento Financiero del módulo. Atributos clave: `idLiquidacion`, `idLote`, `idGalpon`, `idResultadoSacrificio` (nulo únicamente en siniestro total), `pollosVendidos` (origen M2), `pesoTotalKg` (origen M2), `pesoPromedioKg` (calculado, solo presentación), `precioKgCop` (ingresado; nulo en siniestro total), `ventaBrutaCop`, `mortalidadAves`, `porcentajeMortalidad`, `costosOperativosCop`, `utilidadNetaCop`, `estado` (`ACTIVA` | `ANULADA`), `fechaHoraGeneracion`, `usuarioResponsable`.
- **ResultadoFinalSacrificio**: Resultado operativo originado en Módulo 2 y obtenido por M3 mediante consulta (M3-CU08). Atributos sincronizados: `idResultado`, `idLote`, `idGalpon`, `cantidadFinalPollos`, `pesoTotalKg`, `fechaRegistro`, `fechaHoraSincronizacion`, `estadoUtilizacionLocal`.
- **PartidaCostoLote**: Partida de costo sincronizada y valorizada asociada al ciclo del lote. Atributos: `idPartida`, `idLote`, `categoria` (`ALIMENTO` | `MEDICINA` | `POBLACION`), `concepto`, `cantidad`, `unidadMedida`, `precioUnitarioCop`, `subtotalCop`, `fuenteOrigen`, `referenciaOrigen`.
- **SnapshotDatosOrigen**: Datos sincronizados utilizados para una Liquidación. Atributos: `idLiquidacion`, `idLote`, `fuente` (`MODULO_1` | `MODULO_2`), `fechaHoraSincronizacion` y referencias a las poblaciones, al resultado de sacrificio y a las partidas de costo empleadas.
- **AlertaVaciadoSanitario**: Evento registrado por Módulo 1 que conserva `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento` después de desvincular el lote del galpón; es la referencia que identifica al lote liquidable.

> **Nota**: La entidad `MatrizVentas` del antiguo M3-CU02 se elimina. Sus atributos de venta (`pollosVendidos`, `pesoTotalKg`, `pesoPromedioKg`, `precioKgCop`) pasan a `Liquidacion`. La Matriz de Venta Final es la presentación de esta entidad, no una entidad propia.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las Liquidaciones de prueba coincide con exactitud al peso contra el cálculo aritmético manual verificado.
- **SC-002**: El tiempo total de procesamiento y despliegue de la Liquidación en pantalla es inferior a 5 segundos tras pulsar "Generar Liquidación".
- **SC-003**: 0% de Liquidaciones generadas con partidas de costo sin valorizar, sin resultado final de sacrificio válido o con precio por kg no positivo, excepto las Liquidaciones de siniestro total autorizadas con población actual igual a 0.
- **SC-004**: 100% de inmutabilidad: ninguna Liquidación `ACTIVA` cambia su valor ante modificaciones posteriores de los datos de origen sin anulación previa.
- **SC-005**: El 100% de las Liquidaciones generadas conserva la fecha y hora de sincronización de cada fuente utilizada en el cálculo y el resultado final de sacrificio consumido.
- **SC-006**: 0% de lotes con más de una Liquidación en estado `ACTIVA`.
