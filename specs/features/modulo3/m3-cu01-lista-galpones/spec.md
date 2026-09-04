# Feature Specification: M3-CU01 – Consultar Lista de Galpones

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta y Selección de Galpones (Priority: P1)

Como administrador financiero, quiero consultar la lista consolidada de galpones y sus lotes con su identificación, nombre, estado operativo y población actual, para identificar la situación operativa de cada lote y seleccionar los que estén en condiciones de registrar ventas o generar liquidaciones.

**Why this priority**: Es el punto de entrada principal y obligatorio para todos los flujos operativos y financieros del Módulo 3. Sin esta vista, el usuario no puede seleccionar un lote para operar.

**Independent Test**: Se accede a la vista de lista de galpones y se verifica que el sistema muestra su copia local sincronizada de los datos cuya fuente oficial es el Módulo 1, bloqueando acciones sobre galpones no aptos. La consulta debe funcionar aunque el Módulo 1 no esté disponible en ese instante.

**Acceptance Scenarios**:

1. **Scenario**: Visualización de galpones con datos completos
   - **Given** existen galpones registrados en la copia local sincronizada, con origen en el Módulo 1
   - **When** el administrador financiero accede al listado de galpones
   - **Then** el sistema muestra cada galpón con su UUID, nombre, aforo máximo, estado actual (`Disponible`, `Vaciado Sanitario`, `Productivo`, `En Cosecha`, `Mantenimiento`, `Aislamiento`), y, cuando exista un lote activo, su UUID, nombre, fecha de ingreso, edad calculada en días, población inicial, población actual y fecha/hora de la última sincronización

2. **Scenario**: Selección de galpón en cosecha para registro de venta
   - **Given** se visualiza la lista y existe un galpón con lote activo en estado `En Cosecha`
   - **When** el usuario selecciona dicho galpón
   - **Then** el sistema habilita la opción para registrar matriz de ventas (M3-CU02)

3. **Scenario**: Selección de galpón en vaciado sanitario para liquidación
   - **Given** se visualiza la lista y existe un galpón en estado `Vaciado Sanitario` con una matriz de ventas `ACTIVA`
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema mantiene deshabilitado el registro de venta y habilita la opción para generar la liquidación del lote (M3-CU03)

4. **Scenario**: Galpón sin condiciones para venta o liquidación
   - **Given** un galpón listado en estado `Disponible`, `Productivo`, `Mantenimiento` o `Aislamiento`
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema muestra su información de forma informativa pero mantiene deshabilitadas las acciones de registro de venta y generación de liquidación, indicando que el galpón no se encuentra en fase de cosecha o de vaciado sanitario

5. **Scenario**: Ausencia total de datos sincronizados
   - **Given** no existen galpones en la copia local sincronizada
   - **When** el administrador accede a la lista
   - **Then** el sistema presenta un estado vacío con el mensaje explicativo "No hay galpones disponibles para consultar" y bloquea cualquier intento de registro

6. **Scenario**: Indisponibilidad temporal de Módulo 1 durante la sincronización
   - **Given** existe una copia local sincronizada previamente y Módulo 1 no está disponible cuando inicia una sincronización periódica
   - **When** el administrador consulta la lista de galpones
   - **Then** el sistema muestra la última copia local disponible, identifica su fecha/hora de actualización y reintenta la sincronización en segundo plano sin interrumpir la consulta ni los demás módulos

---

### Edge Cases

- **Galpón con población actual igual a 0 en estado Productivo/En Cosecha**: El sistema permite visualizarlo pero restringe el registro de ventas si no hay aves disponibles.
- **Transición de estado concurrente**: Si el estado del galpón cambia en el Módulo 1, el cambio se refleja en la siguiente sincronización. Las acciones en M3 se validan contra la copia local vigente, sin requerir una llamada en vivo al Módulo 1.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar desde su copia local sincronizada la lista de galpones con UUID, nombre, aforo máximo, estado operativo y fecha/hora de última sincronización. Cuando el galpón tenga lote activo, MUST mostrar además UUID y nombre de lote, fecha de ingreso, edad calculada en días, población inicial y población actual.
- **FR-002**: El sistema MUST tomar la población inicial, población actual y el estado operativo cuyo origen es el Módulo 1 como la fuente oficial de verdad. Módulo 1 actualiza la población actual con base en la mortalidad registrada y comunicada por Módulo 2. La consulta del usuario DEBE usar sincronización asíncrona y periódica, sin depender de una conexión en vivo con Módulo 1.
- **FR-003**: El sistema MUST admitir únicamente el siguiente catálogo unificado de estados operativos:
  - `Disponible`: Galpón sin lote activo (no accionable para venta o liquidación).
  - `Vaciado Sanitario`: Galpón sin aves después de la cosecha. No permite registrar venta; permite generar liquidación únicamente si existe una matriz de ventas `ACTIVA`.
  - `Productivo`: Lote en crecimiento/alimentación (consultable, no accionable para venta o liquidación).
  - `En Cosecha`: Lote en fase de recolección y venta (accionable únicamente para registro de ventas).
  - `Mantenimiento`: Galpón en reparaciones estructurales (no accionable para venta o liquidación).
  - `Aislamiento`: Galpón o lote con restricción sanitaria (no accionable para venta o liquidación).
- **FR-004**: El sistema MUST permitir al usuario seleccionar un galpón en estado `En Cosecha` para navegar hacia el registro de matriz de ventas (M3-CU02). Cuando el estado sea `Vaciado Sanitario` y exista una matriz de ventas `ACTIVA`, MUST permitir navegar hacia la liquidación del lote (M3-CU03).
- **FR-005**: El sistema MUST deshabilitar el registro de venta para estados diferentes de `En Cosecha` y la generación de liquidación para estados diferentes de `Vaciado Sanitario` o cuando no exista una matriz de ventas `ACTIVA`.
- **FR-006**: Ante la ausencia de registros de galpones, el sistema MUST mostrar un estado vacío claro y mantener bloqueada cualquier acción posterior.
- **FR-007**: El sistema MUST ejecutar la sincronización de galpones en segundo plano según un intervalo configurable. Si el Módulo 1 no está disponible, MUST conservar la última copia local completa, registrar el fallo y reintentar sin bloquear M3 ni los demás módulos.

### Key Entities

- **Galpón**: Unidad física de producción avícola. Atributos clave: `idGalpon` (UUID), `nombre`, `aforoMaximo`, `estado` (`Disponible` | `Vaciado Sanitario` | `Productivo` | `En Cosecha` | `Mantenimiento` | `Aislamiento`).
- **Lote**: Conjunto de aves alojado en un galpón durante un ciclo productivo. Atributos clave: `idLote` (UUID), `idGalpon` (UUID, relación con Galpón), `nombre`, `fechaIngreso`, `edadCalculadaDias` (derivada de la fecha de ingreso y la fecha actual), `poblacionInicial`, `poblacionActual`.
- **CopiaLocalGalponLote**: Réplica local completa de Galpón y su lote activo utilizada por M3, con `fechaHoraUltimaSincronizacion` y estado de sincronización.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de carga y renderizado de la lista de galpones es inferior a 3 segundos para operaciones de hasta 20 galpones simultáneos.
- **SC-002**: El 100% de los datos de galpón y lote reflejan con exactitud la información provista por el Módulo 1.
- **SC-003**: Cero accesos permitidos al registro de venta fuera de `En Cosecha` o a la generación de liquidación fuera de `Vaciado Sanitario` con matriz de ventas `ACTIVA`.
- **SC-004**: Los usuarios administradores pueden identificar el estado de cualquier galpón en menos de 5 segundos de interacción.
- **SC-005**: El 100% de las consultas de galpones se completa usando la copia local disponible, aun cuando Módulo 1 esté temporalmente indisponible.
