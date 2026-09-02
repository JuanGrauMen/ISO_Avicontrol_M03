# Feature Specification: M3-CU01 – Consultar Lista de Galpones

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta y Selección de Galpones (Priority: P1)

Como administrador financiero, quiero consultar la lista consolidada de galpones con su identificación, estado operativo y población viva actual, para identificar la situación operativa de cada lote y seleccionar galpones aptos para registrar ventas o consultar liquidaciones.

**Why this priority**: Es el punto de entrada principal y obligatorio para todos los flujos operativos y financieros del Módulo 3. Sin esta vista, el usuario no puede seleccionar un lote para operar.

**Independent Test**: Se accede a la vista de lista de galpones y se verifica que el sistema muestra su copia local sincronizada de los datos cuya fuente oficial es el Módulo 1, bloqueando acciones sobre galpones no aptos y mostrando alertas ante discrepancias de población. La consulta debe funcionar aunque el Módulo 1 no esté disponible en ese instante.

**Acceptance Scenarios**:

1. **Scenario**: Visualización de galpones con datos completos
   - **Given** existen galpones registrados en la copia local sincronizada, con origen en el Módulo 1
   - **When** el administrador financiero accede al listado de galpones
   - **Then** el sistema muestra cada galpón con su Identificador único, Estado actual (`Vaciado Sanitario`, `Productivo`, `En Cosecha`, `Mantenimiento`), Población viva actual y fecha/hora de la última sincronización

2. **Scenario**: Selección de galpón accionable para flujo de ventas/liquidación
   - **Given** se visualiza la lista y existe un galpón en estado `En Cosecha` o `Productivo`
   - **When** el usuario selecciona dicho galpón
   - **Then** el sistema habilita las opciones para registrar matriz de ventas (M3-CU02) o consultar/generar liquidación (M3-CU03)

3. **Scenario**: Galpón en estado no accionable
   - **Given** un galpón listado en estado `Vaciado Sanitario` o `Mantenimiento`
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema muestra su información de forma informativa pero mantiene deshabilitadas las acciones de registro de venta y generación de liquidación, indicando que el lote no está en fase productiva/cosecha

4. **Scenario**: Ausencia total de datos sincronizados
   - **Given** no existen galpones en la copia local sincronizada
   - **When** el administrador accede a la lista
   - **Then** el sistema presenta un estado vacío con el mensaje explicativo "No hay galpones disponibles para consultar" y bloquea cualquier intento de registro

5. **Scenario**: Alerta de discrepancia de población entre Módulos
   - **Given** un galpón cuya población viva reportada por Módulo 1 difiere de la registrada en los reportes diarios de Módulo 2
   - **When** se visualiza el galpón en la lista
   - **Then** el sistema muestra como oficial la población de Módulo 1 y despliega un indicador visual de advertencia informativa sobre la discrepancia con Módulo 2

6. **Scenario**: Indisponibilidad temporal de Módulo 1 durante la sincronización
   - **Given** existe una copia local sincronizada previamente y Módulo 1 no está disponible cuando inicia una sincronización periódica
   - **When** el administrador consulta la lista de galpones
   - **Then** el sistema muestra la última copia local disponible, identifica su fecha/hora de actualización y reintenta la sincronización en segundo plano sin interrumpir la consulta ni los demás módulos

---

### Edge Cases

- **Galpón con población viva igual a 0 en estado Productivo/En Cosecha**: El sistema permite visualizarlo pero restringe el registro de ventas si no hay aves vivas disponibles.
- **Pérdida temporal de conexión con el Módulo 1**: La vista no realiza una consulta en vivo. El sistema conserva y muestra la última copia local completa, indica la fecha/hora de su última sincronización y programa reintentos en segundo plano; no borra ni muestra datos parciales.
  
!!! tip "warning"
- tener en cuenta para cambios.
  
- **Transición de estado concurrente**: Si el estado del galpón cambia en el Módulo 1, el cambio se refleja en la siguiente sincronización. Las acciones en M3 se validan contra la copia local vigente, sin requerir una llamada en vivo al Módulo 1.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar desde su copia local sincronizada la lista de galpones con los siguientes campos: Identificador de Galpón / Lote, Estado Operativo, Población Viva Actual y Fecha/Hora de Última Sincronización.
- **FR-002**: El sistema MUST tomar la población inicial, población viva y el estado operativo cuyo origen es el Módulo 1 como la fuente oficial de verdad, mediante sincronización asíncrona y periódica; la consulta del usuario no DEBE depender de una conexión en vivo con dicho módulo.
- **FR-003**: El sistema MUST admitir únicamente el siguiente catálogo unificado de estados operativos:
  - `Vaciado Sanitario`: Galpón en desinfección y preparación (no accionable).
  - `Productivo`: Lote en crecimiento/alimentación (accionable para consulta).
  - `En Cosecha`: Lote en fase de recolección y venta (accionable para registro de ventas).
  - `Mantenimiento`: Galpón en reparaciones estructurales (no accionable).
- **FR-004**: El sistema MUST permitir al usuario seleccionar un galpón accionable para navegar hacia el registro de matriz de ventas (M3-CU02) o hacia la liquidación del lote (M3-CU03).
- **FR-005**: El sistema MUST deshabilitar los botones de acción operativa para galpones en estados `Vaciado Sanitario` y `Mantenimiento`.
- **FR-006**: Si existe reporte de población en Módulo 2 que difiera de Módulo 1, el sistema MUST utilizar el dato de Módulo 1 y desplegar una alerta visual de inconsistencia sin interrumpir la consulta.
- **FR-007**: Ante la ausencia de registros de galpones, el sistema MUST mostrar un estado vacío claro y mantener bloqueada cualquier acción posterior.
- **FR-008**: El sistema MUST ejecutar la sincronización de galpones en segundo plano según un intervalo configurable. Si el Módulo 1 no está disponible, MUST conservar la última copia local completa, registrar el fallo y reintentar sin bloquear M3 ni los demás módulos.

### Key Entities

- **Galpón / Lote**: Representa la unidad productiva física y el ciclo de crianza. Atributos clave: `idGalpon`, `idLote`, `estado` (`Vaciado Sanitario` | `Productivo` | `En Cosecha` | `Mantenimiento`), `poblacionInicial`, `poblacionVivaActual`.
- **AlertaInconsistencia**: Indicador generado cuando `poblacionViva(Módulo 1) != poblacionReportada(Módulo 2)`.
- **CopiaLocalGalpon**: Réplica local completa de los datos de Galpón / Lote utilizados por M3, con `fechaHoraUltimaSincronizacion` y estado de sincronización.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de carga y renderizado de la lista de galpones es inferior a 3 segundos para operaciones de hasta 20 galpones simultáneos.
- **SC-002**: El 100% de los datos de identificación, estado y población viva reflejan con exactitud la información provista por el Módulo 1.
- **SC-003**: Cero accesos permitidos a formularios de venta o liquidación sobre galpones en estado `Vaciado Sanitario` o `Mantenimiento`.
- **SC-004**: Los usuarios administradores pueden identificar el estado de cualquier galpón en menos de 5 segundos de interacción.
- **SC-005**: El 100% de las consultas de galpones se completa usando la copia local disponible, aun cuando Módulo 1 esté temporalmente indisponible.
