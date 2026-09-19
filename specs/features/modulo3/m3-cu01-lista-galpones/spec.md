# Feature Specification: M3-CU01 – Consultar Lista de Galpones

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta y Selección de Galpones (Priority: P1)

Como administrador financiero, quiero consultar la lista consolidada de galpones y sus lotes con su identificación, nombre, estado operativo y población, para identificar la situación de cada lote y seleccionar los que estén en condiciones de registrar una Matriz de Ventas o generar una Liquidación.

**Why this priority**: Es el punto de entrada obligatorio de todos los flujos financieros del Módulo 3. Sin esta vista, el usuario no puede seleccionar un lote sobre el cual operar.

**Independent Test**: Se accede a la vista de lista de galpones y se verifica que el sistema muestra su copia local sincronizada de los datos cuya fuente oficial es Módulo 1, bloqueando acciones sobre galpones no aptos. La consulta debe funcionar aunque Módulo 1 no responda en ese instante.

**Acceptance Scenarios**:

1. **Scenario**: Visualización de galpones con datos completos
   - **Given** existen galpones en la copia local sincronizada, con origen en Módulo 1 (M3-CU07)
   - **When** el administrador financiero accede al listado de galpones
   - **Then** el sistema muestra cada galpón con su UUID, nombre, aforo máximo, estado actual (`Disponible`, `Vaciado Sanitario`, `Productivo`, `En Cosecha`, `Mantenimiento`, `Aislamiento`) y, cuando exista un lote activo, su UUID, nombre, fecha de ingreso, edad calculada en días, población inicial, población actual y fecha y hora de la última sincronización

2. **Scenario**: Selección de galpón en cosecha para seguimiento del cierre
   - **Given** se visualiza la lista y existe un galpón con lote activo en estado `En Cosecha`
   - **When** el usuario selecciona dicho galpón
   - **Then** el sistema indica que el cierre operativo está a cargo de Módulo 2 y mantiene deshabilitadas las acciones financieras hasta que Módulo 1 registre el vaciado sanitario

3. **Scenario**: Selección de galpón en vaciado sanitario para operar
   - **Given** se visualiza la lista y existe un galpón en estado `Vaciado Sanitario` con una alerta de vaciado sanitario que conserva los UUID del lote y del galpón
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema habilita el registro de la Matriz de Ventas (M3-CU02) si existe un resultado final de sacrificio sincronizado desde Módulo 2 y no existe Matriz `ACTIVA`; habilita la generación de la Liquidación (M3-CU03) si ya existe Matriz `ACTIVA` o si la población actual sincronizada es 0 por mortalidad total

4. **Scenario**: Galpón sin condiciones para operar
   - **Given** un galpón listado en estado `Disponible`, `Productivo`, `Mantenimiento` o `Aislamiento`
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema muestra su información de forma informativa pero mantiene deshabilitadas las acciones de registro de Matriz de Ventas y de generación de Liquidación, indicando que el galpón no se encuentra en fase de cosecha ni de vaciado sanitario

5. **Scenario**: Ausencia total de datos sincronizados
   - **Given** no existen galpones en la copia local sincronizada
   - **When** el administrador accede a la lista
   - **Then** el sistema presenta un estado vacío con el mensaje "No hay galpones disponibles para consultar" y bloquea cualquier intento de registro

6. **Scenario**: Indisponibilidad temporal de Módulo 1 durante la sincronización
   - **Given** existe una copia local sincronizada previa y Módulo 1 no responde cuando M3 ejecuta una consulta periódica
   - **When** el administrador consulta la lista de galpones
   - **Then** el sistema muestra la última copia local disponible, identifica su fecha y hora de actualización y reintenta la consulta en segundo plano sin interrumpir la vista ni los demás módulos

---

### Edge Cases

- **Galpón con población actual igual a 0 en estado `Productivo` o `En Cosecha`**: El sistema permite visualizarlo pero restringe el registro de Matriz de Ventas, ya que no hay aves comercializables.
- **Transición de estado concurrente**: Si el estado del galpón cambia en Módulo 1, el cambio se refleja en la siguiente sincronización. Las acciones en M3 se validan contra la copia local vigente, sin requerir consulta en vivo a Módulo 1.
- **Población actual distinta de cero en `Vaciado Sanitario`**: Módulo 1 no exige población cero para transicionar a `Vaciado Sanitario`, por lo que un galpón puede encontrarse en ese estado con población actual mayor a 0. Esa diferencia representa aves vendidas, no una inconsistencia, y no bloquea las acciones financieras.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar desde su copia local sincronizada la lista de galpones con UUID, nombre, aforo máximo, estado operativo y fecha y hora de última sincronización. Cuando el galpón tenga lote activo, MUST mostrar además UUID y nombre del lote, fecha de ingreso, edad calculada en días, población inicial y población actual.
- **FR-002**: El sistema MUST tomar la población inicial, la población actual y el estado operativo cuyo origen es Módulo 1 como fuente oficial de verdad. Módulo 1 actualiza la población actual con base en la mortalidad que le comunica Módulo 2. La vista del usuario DEBE alimentarse de la copia local mantenida por M3-CU07, sin depender de una conexión en vivo con Módulo 1.
- **FR-003**: El sistema MUST admitir únicamente el siguiente catálogo unificado de estados operativos:
  - `Disponible`: galpón sin lote activo (no accionable financieramente).
  - `Vaciado Sanitario`: galpón sin aves después de la cosecha. Permite registrar la Matriz de Ventas a partir de un resultado final válido de Módulo 2 y, después, generar la Liquidación; también permite liquidar sin Matriz únicamente en mortalidad total con población actual igual a 0.
  - `Productivo`: lote en crecimiento (consultable, no accionable financieramente).
  - `En Cosecha`: lote en fase de cierre operativo a cargo de Módulo 2 (consultable, no accionable financieramente).
  - `Mantenimiento`: galpón en reparaciones estructurales (no accionable financieramente).
  - `Aislamiento`: galpón o lote con restricción sanitaria (no accionable financieramente).
  
  [NEEDS CLARIFICATION – D-07: la grafía exacta de cada estado debe acordarse con Módulo 1, que actualmente los escribe en minúscula ("En cosecha", "Vaciado sanitario").]
- **FR-004**: Cuando el estado sea `Vaciado Sanitario`, el sistema MUST identificar el lote mediante los UUID históricos de la alerta de vaciado sanitario. MUST permitir navegar al registro de la Matriz de Ventas (M3-CU02) si existe resultado final de sacrificio sincronizado y no existe Matriz `ACTIVA`; MUST permitir navegar a la generación de la Liquidación (M3-CU03) si existe Matriz `ACTIVA` o si la población actual sincronizada es 0.
- **FR-005**: El sistema MUST deshabilitar las acciones financieras en estados diferentes de `Vaciado Sanitario`; MUST deshabilitar el registro de la Matriz de Ventas cuando no exista resultado final válido y la generación de la Liquidación cuando no exista Matriz `ACTIVA`, excepto el siniestro total autorizado con población actual igual a 0.
- **FR-006**: Ante la ausencia de registros de galpones, el sistema MUST mostrar un estado vacío claro y mantener bloqueada cualquier acción posterior.
- **FR-007**: El sistema MUST alimentarse exclusivamente de la copia local mantenida por el proceso de sincronización definido en **M3-CU07**. Esta especificación NO DEBE definir accesos propios a Módulo 1.

### Key Entities

- **Galpón**: Unidad física de producción avícola. Atributos clave: `idGalpon` (UUID), `nombre`, `aforoMaximo`, `estado` (`Disponible` | `Vaciado Sanitario` | `Productivo` | `En Cosecha` | `Mantenimiento` | `Aislamiento`).
- **Lote**: Conjunto de aves alojado en un galpón durante un ciclo productivo. Atributos clave: `idLote` (UUID), `idGalpon` (UUID), `nombre`, `fechaIngreso`, `edadCalculadaDias` (derivada), `poblacionInicial`, `poblacionActual`, `costoTotalCop`.
- **CopiaLocalGalponLote**: Réplica local de Galpón y su lote activo utilizada por M3, con `fechaHoraUltimaSincronizacion` y estado de la última sincronización. Es mantenida por M3-CU07.
- **AlertaVaciadoSanitario**: Evento originado en Módulo 2 y registrado por Módulo 1 al finalizar la cosecha. Conserva `idAlerta`, `idGalpon`, `idLote` y `fechaHoraEvento`, y permite a M3 identificar el lote que fue desvinculado del galpón.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de carga y renderizado de la lista es inferior a 3 segundos para hasta 20 galpones simultáneos.
- **SC-002**: El 100% de los datos de galpón y lote refleja con exactitud la información obtenida de Módulo 1 en la última sincronización.
- **SC-003**: Cero accesos permitidos al registro de la Matriz de Ventas o a la generación de la Liquidación fuera de `Vaciado Sanitario`; la Liquidación sin Matriz solo se permite en siniestro total con población actual igual a 0.
- **SC-004**: Los usuarios pueden identificar el estado de cualquier galpón en menos de 5 segundos de interacción.
- **SC-005**: El 100% de las consultas de galpones se completa usando la copia local disponible, aun cuando Módulo 1 esté temporalmente indisponible.
