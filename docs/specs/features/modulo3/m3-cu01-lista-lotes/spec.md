# Feature Specification: M3-CU01 – Consultar Lista de Lotes

**Created**: 2026-08-31  
**Actualizado**: 2026-10-03  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta y Selección de Lotes (Priority: P1)

Como administrador financiero, quiero consultar la lista consolidada de lotes con su identificación, el galpón que ocupan o ocuparon y su etapa financiera, para identificar qué lotes están pendientes de liquidar y operar sobre ellos.

**Why this priority**: Es el punto de entrada obligatorio de todos los flujos financieros del Módulo 3. Toda acción financiera recae sobre un lote; sin esta vista, el usuario no puede seleccionar un lote sobre el cual operar.

**Independent Test**: Se accede a la vista de lista de lotes y se verifica que el sistema muestra su copia local sincronizada de los datos cuya fuente oficial es Módulo 1, con la etapa de cada lote derivada por M3, bloqueando acciones sobre lotes no aptos. La consulta debe funcionar aunque Módulo 1 no responda en ese instante.

**Acceptance Scenarios**:

1. **Scenario**: Visualización de lotes con datos completos
   - **Given** existen lotes en la copia local sincronizada, con origen en Módulo 1 (M3-CU07)
   - **When** el administrador financiero accede al listado de lotes
   - **Then** el sistema muestra cada lote con su UUID, nombre, fecha de ingreso, el nombre y UUID del galpón que ocupa u ocupó y su etapa (`Productivo`, `En Cosecha`, `Aislamiento`, `Por Liquidar`, `Liquidado`), junto con la fecha y hora de la última sincronización con Módulo 1 y con Módulo 2. El sistema permite filtrar la lista por etapa.

2. **Scenario**: Selección de lote en cosecha para seguimiento del cierre
   - **Given** se visualiza la lista y existe un lote en etapa `En Cosecha`
   - **When** el usuario selecciona dicho lote
   - **Then** el sistema indica que el cierre operativo está a cargo de Módulo 2 y mantiene deshabilitadas las acciones financieras hasta que Módulo 1 registre el vaciado sanitario

3. **Scenario**: Selección de lote por liquidar
   - **Given** se visualiza la lista y existe un lote en etapa `Por Liquidar`
   - **When** el usuario interactúa con dicho lote
   - **Then** el sistema habilita la generación de la Liquidación (M3-CU03) si existe un resultado final de sacrificio sincronizado desde Módulo 2 o si la población actual sincronizada es 0 por mortalidad total

4. **Scenario**: Lote sin condiciones para operar
   - **Given** un lote listado en etapa `Productivo` o `Aislamiento`
   - **When** el usuario interactúa con dicho lote
   - **Then** el sistema muestra su información de forma informativa y presenta la acción de generar la Liquidación deshabilitada, indicando que el lote no ha cerrado su ciclo productivo

5. **Scenario**: Ausencia total de datos sincronizados
   - **Given** no existen lotes en la copia local sincronizada
   - **When** el administrador accede a la lista
   - **Then** el sistema presenta un estado vacío con el mensaje "No hay lotes disponibles para consultar" y bloquea cualquier intento de generar una Liquidación

6. **Scenario**: Indisponibilidad temporal de Módulo 1 durante la sincronización
   - **Given** existe una copia local sincronizada previa y Módulo 1 no responde cuando M3 ejecuta una consulta periódica
   - **When** el administrador consulta la lista de lotes
   - **Then** el sistema muestra la última copia local disponible, identifica su fecha y hora de actualización y reintenta la consulta en segundo plano sin interrumpir la vista ni los demás módulos

7. **Scenario**: Filtrado por etapa sin coincidencias
   - **Given** se visualiza la lista de lotes y el usuario selecciona una etapa (`Productivo`, `En Cosecha`, `Aislamiento`, `Por Liquidar`, `Liquidado`)
   - **When** ningún lote de la copia local corresponde a esa etapa
   - **Then** el sistema muestra el mensaje "No se encontraron resultados"

8. **Scenario**: Búsqueda por nombre o UUID sin coincidencias
   - **Given** se visualiza la lista de lotes y el usuario ingresa un término de búsqueda por nombre de lote, nombre de galpón o UUID
   - **When** ningún registro de la copia local coincide con el término ingresado
   - **Then** el sistema muestra el mensaje "No se encontraron resultados"

9. **Scenario**: Paginación de la lista
   - **Given** la copia local sincronizada contiene más de 6 lotes
   - **When** el administrador financiero accede al listado
   - **Then** el sistema muestra únicamente los primeros 6 registros, sin importar el total disponible, y presenta al pie de la lista de resultados la cantidad mostrada frente al total (por ejemplo, "6 de 7 lotes"); el usuario avanza a los registros restantes mediante una acción explícita de paginación

10. **Scenario**: Lote liquidado
    - **Given** se visualiza la lista y existe un lote en etapa `Liquidado`
    - **When** el usuario interactúa con dicho lote
    - **Then** el sistema habilita la consulta de la Liquidación existente (M3-CU03.FR-014), desde la cual se accede al Desglose (M3-CU05) y a la anulación (M3-CU04); la fila no ofrece la generación de una nueva Liquidación


11. **Scenario**: Lote en siniestro total
    - **Given** un lote cuya población actual sincronizada es 0 por mortalidad total
    - **When** el administrador financiero visualiza la lista de lotes
    - **Then** el sistema muestra en la fila del lote, junto a su etapa, el distintivo "Siniestro total" y, si el lote está en etapa `Por Liquidar`, habilita la generación de la Liquidación sin resultado final de sacrificio (M3-CU03.FR-005). [NEEDS CLARIFICATION – D-04: un lote con mortalidad total en `Productivo` no alcanza hoy el vaciado sanitario en Módulo 1]
---

#### Edge Cases

- **Galpón sin lote**: Un galpón en `Disponible` o `Mantenimiento`, o en `Aislamiento` sin lote vinculado, no produce ninguna fila. Su estado se conserva en la copia local (M3-CU07), pero no se lista.
- **Lote con población actual igual a 0 en `Productivo` o `En Cosecha`**: El sistema permite visualizarlo pero no habilita la Liquidación, porque Módulo 1 aún no ha registrado el vaciado sanitario. [NEEDS CLARIFICATION – D-04]
- **Transición de estado concurrente**: Si el estado del galpón cambia en Módulo 1, el cambio se refleja en la siguiente sincronización. Las acciones en M3 se validan contra la copia local vigente, sin requerir consulta en vivo a Módulo 1.
- **Población actual distinta de cero tras el vaciado sanitario**: Módulo 1 no exige población cero para transicionar a `Vaciado Sanitario`, por lo que un lote `Por Liquidar` puede tener población actual mayor a 0. Esa diferencia representa aves vendidas, no una inconsistencia, y no bloquea las acciones financieras.
- **Galpón reocupado**: Si el galpón de un lote `Por Liquidar` o `Liquidado` recibe un lote nuevo, la etapa del lote anterior no cambia: se deriva de la alerta de vaciado sanitario y de la Liquidación, no del estado actual del galpón (FR-013).
- **Liquidación anulada**: Si la única Liquidación de un lote pasa a `ANULADA` (M3-CU04), el lote vuelve a la etapa `Por Liquidar`.
- **Filtro y búsqueda combinados**: Si el usuario aplica un filtro por etapa y además ingresa un término de búsqueda, ambos criterios se combinan; el mensaje de FR-010 aplica igual si la combinación no retorna resultados.
- **Paginación tras aplicar filtro o búsqueda**: Al cambiar el filtro por etapa o el término de búsqueda, el sistema MUST reiniciar la paginación en la primera página.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar desde su copia local sincronizada la lista de lotes con UUID y nombre del lote, su fecha de ingreso, el nombre y UUID del galpón que ocupa u ocupó, su etapa (FR-013) y la fecha y hora de la última sincronización con Módulo 1 y con Módulo 2 (RT-09). El aforo máximo, la edad en días, las poblaciones inicial y actual y el porcentaje de mortalidad NO se muestran en esta lista: se conservan en la copia local (M3-CU07) para los cálculos de M3-CU03, y la mortalidad del lote se presenta en la Liquidación. El sistema MUST permitir filtrar la lista por etapa. Cuando la población actual sincronizada del lote sea 0 por mortalidad total, la fila MUST mostrar, junto a su etapa, el distintivo **"Siniestro total"**; la población no se muestra como cifra.
- **FR-002**: El sistema MUST tomar el estado operativo del galpón, la alerta de vaciado sanitario, la población inicial y la población actual cuyo origen es Módulo 1 como fuente oficial de verdad para derivar la etapa del lote y para habilitar o bloquear las acciones financieras. Módulo 1 actualiza la población actual con base en la mortalidad que le comunica Módulo 2. La vista del usuario DEBE alimentarse de la copia local mantenida por M3-CU07, sin depender de una conexión en vivo con Módulo 1.
- **FR-003**: El sistema MUST admitir únicamente el siguiente catálogo de estados operativos **del galpón**, tal como los recibe de Módulo 1 (M3-CU07.FR-011). Este catálogo no se muestra en la lista; es el insumo de FR-013:
  - `Disponible`: galpón sin lote.
  - `Vaciado Sanitario`: galpón sin aves después de la cosecha; Módulo 1 desvincula el lote y conserva sus UUID en la alerta de vaciado sanitario.
  - `Productivo`: lote en crecimiento.
  - `En Cosecha`: lote en fase de cierre operativo a cargo de Módulo 2.
  - `Mantenimiento`: galpón en reparaciones estructurales, sin lote.
  - `Aislamiento`: galpón o lote con restricción sanitaria.
  
  [NEEDS CLARIFICATION – D-07: la grafía exacta de cada estado debe acordarse con Módulo 1, que actualmente los escribe en minúscula ("En cosecha", "Vaciado sanitario").]
- **FR-004**: Para un lote en etapa `Por Liquidar`, el sistema MUST identificar el lote mediante los UUID históricos de la alerta de vaciado sanitario y MUST ofrecer, **en la propia fila del lote y sin pantalla de detalle intermedia**, la acción de generar la Liquidación (M3-CU03) si existe resultado final de sacrificio sincronizado o si la población actual sincronizada es 0. Para un lote en etapa `Liquidado`, MUST ofrecer en la propia fila la acción "Ver liquidación", que abre la Liquidación `ACTIVA` del lote (M3-CU03.FR-014). La lista NO ofrece acceso directo al Desglose ni a la anulación: ambos se alcanzan desde la Liquidación.
- **FR-005**: Cada fila MUST mostrar **una única acción**, determinada por la etapa del lote (FR-013), y NO DEBE ocultarla cuando no aplique:
  - `Por Liquidar` → "Generar liquidación", habilitada si existe resultado final válido o si se trata de un siniestro total autorizado con población actual igual a 0; deshabilitada en caso contrario.
  - `Liquidado` → "Ver liquidación", habilitada.
  - `Productivo`, `En Cosecha` o `Aislamiento` → "Generar liquidación", deshabilitada.
- **FR-006**: Ante la ausencia de registros de lotes, el sistema MUST mostrar un estado vacío claro y mantener bloqueada cualquier acción posterior.
- **FR-007**: El sistema MUST alimentarse exclusivamente de la copia local mantenida por el proceso de sincronización definido en **M3-CU07** y de las Liquidaciones registradas por M3. Esta especificación NO DEBE definir accesos propios a Módulo 1.
- **FR-008**: El sistema MUST permitir filtrar la lista de lotes por una única etapa de FR-013, o por la opción "Todos".
- **FR-009**: El sistema MUST permitir buscar lotes por nombre de lote, nombre de galpón o UUID, sobre la copia local sincronizada.
- **FR-010**: Cuando el filtro por etapa o la búsqueda no retornen ningún lote, el sistema MUST mostrar el mensaje "No se encontraron resultados". Este mensaje es distinto del definido en FR-006, que aplica exclusivamente cuando la copia local no contiene ningún lote registrado.
- **FR-011**: El sistema MUST mostrar como máximo 6 lotes por página, sin importar cuántos registros resulten del filtro o búsqueda aplicados. Si el total de registros es mayor a 6, el sistema MUST paginar y mostrar únicamente los primeros 6 en la página inicial.
- **FR-012**: El sistema MUST indicar, al pie de la lista de resultados, la cantidad de lotes mostrados en la página actual frente al total de registros disponibles (por ejemplo, "6 de 7 lotes").
- **FR-013**: El sistema MUST derivar la **etapa** de cada lote a partir de la copia local (M3-CU07) y de las Liquidaciones de M3, aplicando las reglas en este orden; la primera que se cumple determina la etapa:
  1. Existe alerta de vaciado sanitario para el lote y existe una Liquidación `ACTIVA` del lote → `Liquidado`.
  2. Existe alerta de vaciado sanitario para el lote y no existe Liquidación `ACTIVA` → `Por Liquidar`.
  3. El lote está vinculado a un galpón en `Productivo` → `Productivo`.
  4. El lote está vinculado a un galpón en `En Cosecha` → `En Cosecha`.
  5. El lote está vinculado a un galpón en `Aislamiento` → `Aislamiento`.
  
  La etapa es una derivación de M3: no se recibe de Módulo 1, no se le comunica y no altera la copia local. Un galpón que no tiene lote vinculado ni alerta pendiente no genera fila.

### Key Entities

- **Lote**: Conjunto de aves alojado en un galpón durante un ciclo productivo. Atributos clave: `idLote` (UUID), `idGalpon` (UUID), `nombre`, `fechaIngreso`, `edadCalculadaDias` (derivada), `poblacionInicial`, `poblacionActual`, `costoTotalCop`.
- **Galpón**: Unidad física de producción avícola. Atributos clave: `idGalpon` (UUID), `nombre`, `aforoMaximo`, `estado` (`Disponible` | `Vaciado Sanitario` | `Productivo` | `En Cosecha` | `Mantenimiento` | `Aislamiento`).
- **Etapa del Lote**: Valor derivado por FR-013 (`Productivo` | `En Cosecha` | `Aislamiento` | `Por Liquidar` | `Liquidado`). No se almacena.
- **CopiaLocalGalponLote**: Réplica local de Galpón y su lote activo utilizada por M3, con `fechaHoraUltimaSincronizacion` y estado de la última sincronización. Es mantenida por M3-CU07.
- **AlertaVaciadoSanitario**: Evento originado en Módulo 2 y registrado por Módulo 1 al finalizar la cosecha. Conserva `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento`, y permite a M3 identificar el lote que fue desvinculado del galpón.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de carga y renderizado de la lista es inferior a 3 segundos para hasta 20 lotes simultáneos.
- **SC-002**: El 100% de los datos de galpón y lote mostrados refleja con exactitud la información obtenida de Módulo 1 en la última sincronización.
- **SC-003**: Cero accesos permitidos a la generación de la Liquidación fuera de la etapa `Por Liquidar`; la Liquidación sin resultado de sacrificio solo se permite en siniestro total con población actual igual a 0.
- **SC-004**: Los usuarios pueden identificar la etapa de cualquier lote en menos de 5 segundos de interacción.
- **SC-005**: El 100% de las consultas de lotes se completa usando la copia local disponible, aun cuando Módulo 1 esté temporalmente indisponible.
