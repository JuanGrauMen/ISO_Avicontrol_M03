# Épica: Liquidación de Lote y Análisis de Rentabilidad (Módulo 3 – AVICONTROL)

**Created**: 2026-08-31  
**Actualizado**: 2026-09-21  
**Origen**: Casos de uso – `docs/diagramas/Modulo3_v1.drawio`  
**Modelo**: 1 especificación = 1 caso de uso (autocontenidas según `spec-template.md`)  

Este documento es el **índice del módulo**: coordina las especificaciones atómicas, sus prioridades y dependencias, y fija el **glosario** y las **reglas transversales** que todas las especificaciones deben respetar. Cada especificación dentro de su subcarpeta es 100% autocontenida y autónoma.

---

## Glosario del Módulo *(normativo)*

Estos términos tienen un único significado en todo el módulo. Ninguna especificación puede usarlos con otro sentido ni introducir sinónimos.

- **Liquidación**: Documento de **resultado** y único Documento Financiero del módulo. Toma el resultado final de sacrificio de Módulo 2 (cantidad final y peso total), el precio por kilogramo ingresado por el Administrador Financiero y los costos operativos del ciclo (Módulo 1 y Módulo 2), y determina la Venta Bruta, la Mortalidad del Lote, los Costos Operativos y la Utilidad Neta. Responde a la pregunta **¿cuánto entró y cuánto quedó?**. Es única por lote, inmutable y solo se corrige por anulación formal (M3-CU04).

- **Matriz de Venta Final** (o **Matriz de Ventas**): **Presentación tabular** de la Liquidación, tal como la define el documento raíz de AVICONTROL (§3.2): Venta Bruta, Mortalidad del Lote, Costos Operativos y Utilidad Neta, con los datos de venta que los sustentan (pollos vendidos, peso total, peso promedio, precio por kg). **No es un documento independiente** ni una entidad: es la forma en que se muestra una Liquidación. No se registra por separado, no tiene estado propio y no puede existir sin su Liquidación.

- **Desglose de Ventas y Gastos**: **Proyección de solo lectura** derivada de una Liquidación, que presenta sus partidas línea por línea. No es un documento independiente, no se almacena por separado y no tiene estado propio.

- **Documento Financiero**: Término reservado para la **Liquidación**. Se conserva en el glosario porque las reglas de ciclo de vida (`ACTIVA` → `ANULADA`), exigencia de motivo para anular y conservación para auditoría se enuncian sobre él.

- **Anulación**: Único mecanismo admitido para corregir una Liquidación. Cambia su estado a `ANULADA`, la conserva íntegra para auditoría y rehabilita el lote para generar una Liquidación nueva. Nunca edita ni elimina la Liquidación original.

- **Snapshot financiero**: Conjunto congelado de datos de origen (Módulo 1 y Módulo 2) con el que se calculó una Liquidación, incluyendo la fecha y hora de sincronización de cada fuente y el precio por kilogramo ingresado.

- **Copia local sincronizada**: Réplica que M3 mantiene de los datos cuya fuente oficial es Módulo 1 o Módulo 2, actualizada por consulta periódica en segundo plano.

### Términos eliminados

- **"Reporte financiero"**: eliminado del módulo. Era sinónimo no declarado de *Liquidación*. Toda referencia se reemplaza por **Liquidación**.
- **"Generar Matriz de Ventas"** como nombre de caso de uso: eliminado. El caso de uso que consolida el resultado financiero genera una **Liquidación** y se denomina **M3-CU03 – Generar Liquidación del Lote**.
- **"Matriz de Ventas" como documento de ingreso** (antiguo M3-CU02): eliminado el 2026-09-21. La versión anterior la trataba como un documento intermedio que se registraba antes de liquidar; el documento raíz de AVICONTROL la define como la tabla de resultado que ve el usuario. El precio por kilogramo se ingresa ahora directamente en M3-CU03. Ver `CAMBIOS.md` §7.
- **"Anular Documento Financiero"** como nombre de caso de uso: reemplazado por **M3-CU04 – Anular Liquidación**, único documento anulable.

---

## Modelo de Integración *(normativo)*

**M3 consulta; los módulos de origen no envían información a M3.**

1. M3 obtiene todos los datos externos mediante **consultas propias** a Módulo 1 y Módulo 2, ejecutadas por un proceso de sincronización en segundo plano con intervalo configurable.
2. M3 persiste el resultado de esas consultas en su **copia local sincronizada**, junto con la fecha y hora de cada sincronización.
3. **Ninguna operación funcional de M3 requiere una consulta en vivo.** Si Módulo 1 o Módulo 2 no responden, M3 opera con la última copia local y muestra su antigüedad.
4. M3 **no escribe** en Módulo 1 ni en Módulo 2, con una única excepción declarada en M3-CU08 (aviso de utilización del resultado final de sacrificio).

Las especificaciones M3-CU07 a M3-CU10 documentan cada punto de consulta. Ninguna otra especificación del módulo puede definir un acceso externo por su cuenta.

---

## Reglas Transversales *(normativas)*

- **RT-01 – Moneda**: Todo valor monetario se expresa en pesos colombianos (COP). Los totales se presentan como enteros aplicando redondeo `HALF_UP`. Los precios unitarios admiten decimales.
- **RT-02 – Porcentajes**: Se presentan con 2 cifras decimales.
- **RT-03 – Inmutabilidad**: Ninguna Liquidación en estado `ACTIVA` puede editarse. Toda corrección se ejecuta mediante M3-CU04.
- **RT-04 – Auditoría de anulación**: Toda anulación exige motivo no vacío y registra fecha, hora y usuario responsable.
- **RT-05 – (eliminada 2026-09-21)**: Regulaba el orden de anulación entre Matriz de Ventas y Liquidación. Sin objeto al existir un único Documento Financiero. Se conserva el número para no renumerar las referencias de las demás reglas.
- **RT-06 – (eliminada 2026-09-21)**: Regulaba la independencia de anulación entre Matriz de Ventas y Liquidación. Sin objeto por la misma razón.
- **RT-07 – Valorización**: M3 recibe de los módulos de origen **hechos físicos valorizados** (cantidad + precio unitario). La multiplicación, la consolidación y el cálculo de la utilidad son responsabilidad exclusiva de M3. Ningún módulo externo entrega cifras en pesos ya consolidadas por lote.
- **RT-08 – Costos indirectos**: Administración, arrendamientos, nómina general y servicios quedan fuera del alcance del módulo.
- **RT-09 – Visibilidad de la sincronización**: Toda pantalla del módulo muestra la fecha y hora de la última sincronización exitosa con Módulo 1 y con Módulo 2, y advierte cuando alguno de ellos no responde. La forma visual del indicador es una decisión de diseño; su presencia no.

### Supuestos transversales

- **Identidad del usuario**: El módulo asume un **Administrador Financiero ya autenticado**. La autenticación y la gestión de roles son externas a M3 y no forman parte de ninguna especificación de este módulo. El "usuario responsable" que registran M3-CU03 y M3-CU04 es la identidad que ese contexto externo provee.

---

## Mapa de Especificaciones

### Casos de uso internos (actor: Administrador Financiero)

| ID | Nombre | Prioridad | Depende de | Especificación |
| :--- | :--- | :--- | :--- | :--- |
| M3-CU01 | Consultar Lista de Galpones | P1 | M3-CU07 (punto de entrada) | [m3-cu01-lista-galpones/spec.md](m3-cu01-lista-galpones/spec.md) |
| M3-CU02 | *(eliminado — unificado en M3-CU03)* | — | — | — |
| M3-CU03 | Generar Liquidación del Lote | P1 | M3-CU01 · M3-CU07 · M3-CU08 · M3-CU09 · M3-CU10 | [m3-cu03-generar-liquidacion/spec.md](m3-cu03-generar-liquidacion/spec.md) |
| M3-CU04 | Anular Liquidación | P1 | M3-CU03 | [m3-cu04-anular-liquidacion/spec.md](m3-cu04-anular-liquidacion/spec.md) |
| M3-CU05 | Consultar Desglose de Ventas y Gastos | P2 | M3-CU03 | [m3-cu05-desglose-ventas-gastos/spec.md](m3-cu05-desglose-ventas-gastos/spec.md) |
| M3-CU06 | Consultar Historial de Liquidaciones | P3 | M3-CU03 | [m3-cu06-historial-liquidaciones/spec.md](m3-cu06-historial-liquidaciones/spec.md) |

### Casos de uso de integración (actores: Módulo 1 y Módulo 2)

| ID | Nombre | Prioridad | Fuente | Especificación |
| :--- | :--- | :--- | :--- | :--- |
| M3-CU07 | Consultar Galpón y Lote al Módulo 1 | P1 | Módulo 1 | [m3-cu07-consultar-galpon-lote-m1/spec.md](m3-cu07-consultar-galpon-lote-m1/spec.md) |
| M3-CU08 | Consultar Resultado Final de Sacrificio al Módulo 2 | P1 | Módulo 2 | [m3-cu08-consultar-resultado-sacrificio-m2/spec.md](m3-cu08-consultar-resultado-sacrificio-m2/spec.md) |
| M3-CU09 | Consultar Alimento Requerido al Módulo 2 | P1 | Módulo 2 | [m3-cu09-consultar-alimento-requerido-m2/spec.md](m3-cu09-consultar-alimento-requerido-m2/spec.md) |
| M3-CU10 | Consultar Consumo de Medicamento al Módulo 2 | P1 | Módulo 2 | [m3-cu10-consultar-consumo-medicamento-m2/spec.md](m3-cu10-consultar-consumo-medicamento-m2/spec.md) |

Los nombres de M3-CU07, M3-CU09 y M3-CU10 coinciden con los casos de uso que Módulo 1 y Módulo 2 exponen hacia M3 en sus propios diagramas (*Consultar galpón/lote*, *Consultar alimento requerido por galpón*, *Registrar consumo medicamento*). El nombre de M3-CU09 no cierra la decisión D-01: el criterio de costeo sigue pendiente.

## Cadena de Dependencias

```text
     M3-CU07 (M1) ──────────────┐
     M3-CU08 (M2) ──┐           │
     M3-CU09 (M2) ──┼──────┐    │
     M3-CU10 (M2) ──┘      │    │
                           ▼    ▼
CU01 ────────────────────► CU03 ──► CU05
  │                         │
  │                         ├──► CU04
  │                         │
  └─────────────────────────┴──► CU06
```

Orden sugerido de implementación: **CU07 a CU10** (sincronización) → **CU01** → **CU03** → **CU04** → **CU05 y CU06 en paralelo**.

---

## Criterios de Aceptación Globales del Módulo

El módulo se considera ACEPTADO cuando se cumplen TODOS:

- **CA-G01**: Los 9 casos de uso operan de extremo a extremo; los cinco internos con el rol Administrador Financiero y los cuatro de integración mediante consulta a Módulo 1 y Módulo 2.
- **CA-G02**: Cada escenario de aceptación de los specs CU01, CU03–CU10 pasa tal cual está escrito.
- **CA-G03**: Los cálculos de al menos 3 lotes de prueba coinciden 100% con liquidación manual firmada por el stakeholder.
- **CA-G04**: Es imposible generar una segunda Liquidación `ACTIVA` para el mismo lote.
- **CA-G05**: Todo número en pantalla puede explicarse navegando hasta su registro origen en un máximo de 3 interacciones.
- **CA-G06**: La exportación a Excel refleja exactamente lo mostrado en pantalla.
- **CA-G07**: Los flujos funcionan correctamente con Módulos 1 y 2 simulando: datos completos, datos faltantes y ausencia total de datos.
- **CA-G08**: Ninguna Liquidación `ACTIVA` puede editarse; el 100% de las correcciones pasa por M3-CU04 con motivo, fecha, hora y responsable.
- **CA-G09**: *(eliminado 2026-09-21)* Verificaba el orden de anulación de RT-05. Sin objeto al existir un único Documento Financiero.

## Criterios de Éxito Medibles del Módulo

- **SC-001**: 100% de liquidaciones de prueba coinciden exactamente con cálculo manual verificado.
- **SC-002**: Del clic en "lista de galpones" a la liquidación en pantalla transcurren menos de 30 segundos.
- **SC-003**: Cada cifra es trazable a su origen en ≤ 3 interacciones de navegación.
- **SC-004**: 10 usuarios concurrentes completan consultas dentro de los tiempos definidos en cada spec sin errores.
- **SC-005**: En el primer ciclo real, el 90% de los lotes vendidos quedan liquidados sin reprocesos manuales.
- **SC-006**: Cero casos de Liquidaciones modificables tras su emisión; toda corrección pasa por anulación registrada.
- **SC-007**: El 100% de las operaciones funcionales se completa sin consultas en vivo a Módulo 1 o Módulo 2.

---

## Decisiones Pendientes con Otros Módulos

Estas decisiones bloquean la implementación y deben resolverse en mesa conjunta con los equipos de Módulo 1 y Módulo 2. Cada una está marcada como `NEEDS CLARIFICATION` en la especificación correspondiente.

| # | Decisión | Afecta | Contraparte |
| :--- | :--- | :--- | :--- |
| D-01 | Criterio de costeo del alimento: compras, consumo real o requerimiento proyectado | M3-CU03, M3-CU09 | Módulo 2 |
| D-02 | Exposición formal de población inicial, población actual, costo del lote, estado y alerta de vaciado sanitario | M3-CU07 | Módulo 1 |
| D-03 | Mecanismo de aviso de utilización del resultado final de sacrificio | M3-CU08 | Módulo 2 |
| D-04 | Ruta de estado para liquidar un lote con mortalidad total | M3-CU03, M3-CU07 | Módulo 1 |
| D-05 | Tratamiento del impuesto informado por Módulo 2: ¿los costos operativos se calculan sobre valor neto o con impuesto? | M3-CU03, M3-CU09, M3-CU10 | Módulo 2 |
| D-06 | Registro de mortalidad ocurrida fuera del estado `Productivo` | M3-CU03, M3-CU07 | Módulo 1 |
| D-07 | Grafía exacta del catálogo de estados compartido | M3-CU01, M3-CU07 | Módulo 1 |

---

## Trazabilidad con la Especificación Consolidada Anterior

| Elemento anterior | Ubicación actual |
| :--- | :--- |
| US1 → US5 | Specs atómicos CU01, CU03 → CU06 |
| FR-001 | CU01.FR-001 |
| FR-002 | CU06.FR-001 |
| FR-003 a FR-005 (antiguo CU02) | CU03.FR-001, FR-005, FR-011 a FR-013 |
| FR-006 a FR-012 | CU03.FR-001 a FR-007 |
| FR-013 (anulación) | **CU04** (especificación propia) |
| FR-014, FR-015 | CU05.FR-001 a FR-003 |
| FR-016 | CU03.FR-008 |
| FR-017 (moneda) | RT-01 (este documento) |
| NFR-001 a NFR-005 | Criterios de Éxito y FRs en cada spec |
| E-01 a E-09 | Manejo de Errores y Escenarios integrados en cada spec |
| CA-01 a CA-07 | CA-G01 a CA-G08 (este documento) |
| Integración con M1 y M2 (antes implícita) | **CU07, CU08, CU09, CU10** (especificaciones propias) |
| M3-CU02 – Registrar Matriz de Ventas (versión 2026-09-18) | Unificado en **CU03** (2026-09-21) |
