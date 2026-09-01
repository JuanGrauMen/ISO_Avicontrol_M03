# Feature Specification: M3-CU01 – Consultar Lista de Galpones

**Created**: 2026-08-31  
**Módulo**: 3 – Liquidación de Lote y Análisis de Rentabilidad (AVICONTROL)  
**Rol Principal**: Administrador Financiero  

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consulta y Selección de Galpones (Priority: P1)

Como administrador financiero, quiero consultar la lista consolidada de galpones con su identificación, estado operativo y población viva actual, para identificar la situación operativa de cada lote y seleccionar galpones aptos para registrar ventas o consultar liquidaciones.

**Why this priority**: Es el punto de entrada principal y obligatorio para todos los flujos operativos y financieros del Módulo 3. Sin esta vista, el usuario no puede seleccionar un lote para operar.

**Independent Test**: Se accede a la vista de lista de galpones y se verifica que el sistema lista los galpones con sus datos reportados por el Módulo 1 (fuente oficial), bloqueando acciones sobre galpones no aptos y mostrando alertas ante discrepancias de población.

**Acceptance Scenarios**:

1. **Scenario**: Visualización de galpones con datos completos
   - **Given** existen galpones registrados y reportados por el Módulo 1
   - **When** el administrador financiero accede al listado de galpones
   - **Then** el sistema muestra cada galpón con su Identificador único, Estado actual (`Vaciado Sanitario`, `Productivo`, `En Cosecha`, `Mantenimiento`) y Población viva actual tomada del Módulo 1

2. **Scenario**: Selección de galpón accionable para flujo de ventas/liquidación
   - **Given** se visualiza la lista y existe un galpón en estado `En Cosecha` o `Productivo`
   - **When** el usuario selecciona dicho galpón
   - **Then** el sistema habilita las opciones para registrar matriz de ventas (M3-CU02) o consultar/generar liquidación (M3-CU03)

3. **Scenario**: Galpón en estado no accionable
   - **Given** un galpón listado en estado `Vaciado Sanitario` o `Mantenimiento`
   - **When** el usuario interactúa con dicho galpón
   - **Then** el sistema muestra su información de forma informativa pero mantiene deshabilitadas las acciones de registro de venta y generación de liquidación, indicando que el lote no está en fase productiva/cosecha

4. **Scenario**: Ausencia total de datos reportados por Módulo 1
   - **Given** el Módulo 1 no tiene galpones registrados o el servicio no responde
   - **When** el administrador accede a la lista
   - **Then** el sistema presenta un estado vacío con el mensaje explicativo "No hay galpones reportados por el sistema de galpones (Módulo 1)" y bloquea cualquier intento de registro

5. **Scenario**: Alerta de discrepancia de población entre Módulos
   - **Given** un galpón cuya población viva reportada por Módulo 1 difiere de la registrada en los reportes diarios de Módulo 2
   - **When** se visualiza el galpón en la lista
   - **Then** el sistema muestra como oficial la población de Módulo 1 y despliega un indicador visual de advertencia informativa sobre la discrepancia con Módulo 2

---

### Edge Cases

- **Galpón con población viva igual a 0 en estado Productivo/En Cosecha**: El sistema permite visualizarlo pero restringe el registro de ventas si no hay aves vivas disponibles.
- **Pérdida temporal de conexión con el Módulo 1**: El sistema muestra un mensaje de error amigable indicando la indisponibilidad del servicio de origen y evita mostrar datos parciales o corruptos.
  
!!! tip "warning"
- tener en cuenta para cambios.
  
- **Transición de estado concurrente**: Si el estado del galpón cambia en el Módulo 1 mientras el usuario tiene la lista abierta, la acción se revalida antes de abrir el formulario de venta o liquidación.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST obtener y mostrar la lista de galpones con los siguientes campos: Identificador de Galpón / Lote, Estado Operativo y Población Viva Actual.
- **FR-002**: El sistema MUST tomar la población inicial, población viva y el estado operativo exclusivamente del Módulo 1 como la fuente oficial de verdad.
- **FR-003**: El sistema MUST admitir únicamente el siguiente catálogo unificado de estados operativos:
  - `Vaciado Sanitario`: Galpón en desinfección y preparación (no accionable).
  - `Productivo`: Lote en crecimiento/alimentación (accionable para consulta).
  - `En Cosecha`: Lote en fase de recolección y venta (accionable para registro de ventas).
  - `Mantenimiento`: Galpón en reparaciones estructurales (no accionable).
- **FR-004**: El sistema MUST permitir al usuario seleccionar un galpón accionable para navegar hacia el registro de matriz de ventas (M3-CU02) o hacia la liquidación del lote (M3-CU03).
- **FR-005**: El sistema MUST deshabilitar los botones de acción operativa para galpones en estados `Vaciado Sanitario` y `Mantenimiento`.
- **FR-006**: Si existe reporte de población en Módulo 2 que difiera de Módulo 1, el sistema MUST utilizar el dato de Módulo 1 y desplegar una alerta visual de inconsistencia sin interrumpir la consulta.
- **FR-007**: Ante la ausencia de registros de galpones, el sistema MUST mostrar un estado vacío claro y mantener bloqueada cualquier acción posterior.

### Key Entities

- **Galpón / Lote**: Representa la unidad productiva física y el ciclo de crianza. Atributos clave: `idGalpon`, `idLote`, `estado` (`Vaciado Sanitario` | `Productivo` | `En Cosecha` | `Mantenimiento`), `poblacionInicial`, `poblacionVivaActual`.
- **AlertaInconsistencia**: Indicador generado cuando `poblacionViva(Módulo 1) != poblacionReportada(Módulo 2)`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El tiempo de carga y renderizado de la lista de galpones es inferior a 3 segundos para operaciones de hasta 20 galpones simultáneos.
- **SC-002**: El 100% de los datos de identificación, estado y población viva reflejan con exactitud la información provista por el Módulo 1.
- **SC-003**: Cero accesos permitidos a formularios de venta o liquidación sobre galpones en estado `Vaciado Sanitario` o `Mantenimiento`.
- **SC-004**: Los usuarios administradores pueden identificar el estado de cualquier galpón en menos de 5 segundos de interacción.
