# Épica: Liquidación de Lote y Análisis de Rentabilidad (Módulo 3 – AVICONTROL)

**Created**: 2026-08-31  
**Actualizado**: 2026-09-18  
**Origen**: Casos de uso – `docs/diagramas/M3/Diagrama_M3_Avicontrol.drawio`  
**Modelo**: 1 especificación = 1 caso de uso (autocontenidas según `spec-template.md`)  

Este documento es el **índice del módulo**: coordina las especificaciones atómicas, sus prioridades y dependencias, y fija el **glosario** y las **reglas transversales** que todas las especificaciones deben respetar. Cada especificación dentro de su subcarpeta es 100% autocontenida y autónoma.

---

## Glosario del Módulo *(normativo)*

Estos términos tienen un único significado en todo el módulo. Ninguna especificación puede usarlos con otro sentido ni introducir sinónimos.

- **Matriz de Ventas**: Documento de **ingreso**. Registra el hecho comercial de un lote —cuántas aves se vendieron, con qué peso total y a qué precio por kilogramo— y del cual se deriva la Venta Bruta. Responde a la pregunta **¿cuánto entró?**. No contiene costos ni utilidad. Es única por lote, inmutable y solo se corrige por anulación formal (M3-CU04).

- **Liquidación**: Documento de **resultado**. Confronta el ingreso del lote con los costos operativos de su ciclo para determinar la utilidad neta. Responde a la pregunta **¿cuánto quedó?**. Es única por lote, inmutable y solo se corrige por anulación formal (M3-CU04).

- **Desglose de Ventas y Gastos**: **Proyección de solo lectura** derivada de una liquidación, que presenta sus partidas línea por línea. No es un documento independiente, no se almacena por separado y no tiene estado propio.

- **Documento Financiero**: Término que agrupa a la **Matriz de Ventas** y a la **Liquidación**. Ambos comparten ciclo de vida (`ACTIVA` → `ANULADA`), exigencia de motivo para anular y conservación para auditoría.

- **Anulación**: Único mecanismo admitido para corregir un Documento Financiero. Cambia su estado a `ANULADA`, lo conserva íntegro para auditoría y rehabilita el lote para emitir un documento nuevo del mismo tipo. Nunca edita ni elimina el documento original.

- **Snapshot financiero**: Conjunto congelado de datos de origen (Módulo 1 y Módulo 2) con el que se calculó una liquidación, incluyendo la fecha y hora de sincronización de cada fuente.

- **Copia local sincronizada**: Réplica que M3 mantiene de los datos cuya fuente oficial es Módulo 1 o Módulo 2, actualizada por consulta periódica en segundo plano.

### Distinción obligatoria entre Matriz de Ventas y Liquidación

| | Matriz de Ventas | Liquidación |
| :--- | :--- | :--- |
| Qué es | Un **hecho externo** que ocurrió | Un **acto analítico** interno |
| Verbo | Se **registra** | Se **genera** (se calcula) |
| Contiene | Solo ingresos | Ingresos **menos** costos |
| Conoce los costos | **No** | **Sí**, los consolida |
| ¿Obligatoria? | **No** — no existe en siniestro total | **Sí** — todo lote cerrado se liquida |
| Insumos | Resultado final de sacrificio (M2) + precio por kg (M3) | Matriz de Ventas + costos (M1 y M2) |
| Cardinalidad | 0 o 1 `ACTIVA` por lote | 0 o 1 `ACTIVA` por lote |
| Especificación | M3-CU02 | M3-CU03 |

**Relación entre ambos**: la Liquidación **consume** la Matriz de Ventas; la Matriz de Ventas **no conoce** la existencia de la Liquidación. Por eso pueden existir por separado: matriz sin liquidación (el lote se vendió pero aún no hay precios de insumos cargados) y liquidación sin matriz (siniestro total, no hubo venta).

### Términos eliminados

- **"Reporte financiero"**: eliminado del módulo. Era sinónimo no declarado de *Liquidación*. Toda referencia se reemplaza por **Liquidación**.
- **"Generar Matriz de Ventas"** como nombre de caso de uso: eliminado. El caso de uso que consolida el resultado financiero genera una **Liquidación** y se denomina **M3-CU03 – Generar Liquidación del Lote**.

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
- **RT-03 – Inmutabilidad**: Ningún Documento Financiero en estado `ACTIVA` puede editarse. Toda corrección se ejecuta mediante M3-CU04.
- **RT-04 – Auditoría de anulación**: Toda anulación exige motivo no vacío y registra fecha, hora y usuario responsable.
- **RT-05 – Orden de anulación**: No puede anularse una Matriz de Ventas que tenga una Liquidación `ACTIVA` asociada. **Primero la Liquidación, después la Matriz.**
- **RT-06 – Independencia de anulación**: Anular una Liquidación **no** anula la Matriz de Ventas asociada. Son documentos con ciclos de anulación separados.
- **RT-07 – Valorización**: M3 recibe de los módulos de origen **hechos físicos valorizados** (cantidad + precio unitario). La multiplicación, la consolidación y el cálculo de la utilidad son responsabilidad exclusiva de M3. Ningún módulo externo entrega cifras en pesos ya consolidadas por lote.
- **RT-08 – Costos indirectos**: Administración, arrendamientos, nómina general y servicios quedan fuera del alcance del módulo.

---

## Mapa de Especificaciones

### Casos de uso internos (actor: Administrador Financiero)

| ID | Nombre | Prioridad | Depende de | Especificación |
| :--- | :--- | :--- | :--- | :--- |
| M3-CU01 | Consultar Lista de Galpones | P1 | M3-CU07 (punto de entrada) | [m3-cu01-lista-galpones/spec.md](m3-cu01-lista-galpones/spec.md) |
| M3-CU02 | Registrar Matriz de Ventas | P1 | M3-CU01 · M3-CU08 | [m3-cu02-registrar-matriz-ventas/spec.md](m3-cu02-registrar-matriz-ventas/spec.md) |
| M3-CU03 | Generar Liquidación del Lote | P1 | M3-CU02 · M3-CU07 · M3-CU09 · M3-CU10 | [m3-cu03-generar-liquidacion/spec.md](m3-cu03-generar-liquidacion/spec.md) |
| M3-CU04 | Anular Documento Financiero | P1 | M3-CU02 o M3-CU03 | [m3-cu04-anular-documento-financiero/spec.md](m3-cu04-anular-documento-financiero/spec.md) |
| M3-CU05 | Consultar Desglose de Ventas y Gastos | P2 | M3-CU03 | [m3-cu05-desglose-ventas-gastos/spec.md](m3-cu05-desglose-ventas-gastos/spec.md) |
| M3-CU06 | Consultar Historial de Liquidaciones | P3 | M3-CU03 | [m3-cu06-historial-liquidaciones/spec.md](m3-cu06-historial-liquidaciones/spec.md) |

### Casos de uso de integración (actores: Módulo 1 y Módulo 2)

| ID | Nombre | Prioridad | Fuente | Especificación |
| :--- | :--- | :--- | :--- | :--- |
| M3-CU07 | Consultar Población y Costo del Lote al Módulo 1 | P1 | Módulo 1 | [m3-cu07-consultar-poblacion-costo-m1/spec.md](m3-cu07-consultar-poblacion-costo-m1/spec.md) |
| M3-CU08 | Consultar Resultado Final de Sacrificio al Módulo 2 | P1 | Módulo 2 | [m3-cu08-consultar-resultado-sacrificio-m2/spec.md](m3-cu08-consultar-resultado-sacrificio-m2/spec.md) |
| M3-CU09 | Consultar Alimento del Lote al Módulo 2 | P1 | Módulo 2 | [m3-cu09-consultar-alimento-m2/spec.md](m3-cu09-consultar-alimento-m2/spec.md) |
| M3-CU10 | Consultar Medicina Consumida al Módulo 2 | P1 | Módulo 2 | [m3-cu10-consultar-medicina-m2/spec.md](m3-cu10-consultar-medicina-m2/spec.md) |

## Cadena de Dependencias

```text
     M3-CU07 (M1) ──────────────┐
     M3-CU08 (M2) ──┐           │
     M3-CU09 (M2) ──┼──────┐    │
     M3-CU10 (M2) ──┘      │    │
                           ▼    ▼
CU01 ──► CU02 ──────────► CU03 ──► CU05
  │        │               │
  │        └──► CU04 ◄─────┘
  │                        │
  └────────────────────────┴──► CU06
```

Orden sugerido de implementación: **CU07 a CU10** (sincronización) → **CU01** → **CU02** → **CU03** → **CU04** → **CU05 y CU06 en paralelo**.

---

## Criterios de Aceptación Globales del Módulo

El módulo se considera ACEPTADO cuando se cumplen TODOS:

- **CA-G01**: Los 10 casos de uso operan de extremo a extremo; los seis internos con el rol Administrador Financiero y los cuatro de integración mediante consulta a Módulo 1 y Módulo 2.
- **CA-G02**: Cada escenario de aceptación de los specs CU01–CU10 pasa tal cual está escrito.
- **CA-G03**: Los cálculos de al menos 3 lotes de prueba coinciden 100% con liquidación manual firmada por el stakeholder.
- **CA-G04**: Es imposible generar una segunda Matriz de Ventas o Liquidación `ACTIVA` para el mismo lote.
- **CA-G05**: Todo número en pantalla puede explicarse navegando hasta su registro origen en un máximo de 3 interacciones.
- **CA-G06**: La exportación a Excel refleja exactamente lo mostrado en pantalla.
- **CA-G07**: Los flujos funcionan correctamente con Módulos 1 y 2 simulando: datos completos, datos faltantes y ausencia total de datos.
- **CA-G08**: Ningún Documento Financiero `ACTIVA` puede editarse; el 100% de las correcciones pasa por M3-CU04 con motivo, fecha, hora y responsable.
- **CA-G09**: Se cumple el orden de anulación de RT-05 en el 100% de los intentos; el sistema bloquea la anulación de una Matriz con Liquidación `ACTIVA`.

## Criterios de Éxito Medibles del Módulo

- **SC-001**: 100% de liquidaciones de prueba coinciden exactamente con cálculo manual verificado.
- **SC-002**: Del clic en "lista de galpones" a la liquidación en pantalla transcurren menos de 30 segundos.
- **SC-003**: Cada cifra es trazable a su origen en ≤ 3 interacciones de navegación.
- **SC-004**: 10 usuarios concurrentes completan consultas dentro de los tiempos definidos en cada spec sin errores.
- **SC-005**: En el primer ciclo real, el 90% de los lotes vendidos quedan liquidados sin reprocesos manuales.
- **SC-006**: Cero casos de Documentos Financieros modificables tras su emisión; toda corrección pasa por anulación registrada.
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
| US1 → US5 | Specs atómicos CU01 → CU06 |
| FR-001 | CU01.FR-001 |
| FR-002 | CU06.FR-001 |
| FR-003 a FR-005 | CU02.FR-001 a FR-005 |
| FR-006 a FR-012 | CU03.FR-001 a FR-007 |
| FR-013 (anulación) | **CU04** (especificación propia) |
| FR-014, FR-015 | CU05.FR-001 a FR-003 |
| FR-016 | CU02.FR-006 y CU03.FR-008 |
| FR-017 (moneda) | RT-01 (este documento) |
| NFR-001 a NFR-005 | Criterios de Éxito y FRs en cada spec |
| E-01 a E-09 | Manejo de Errores y Escenarios integrados en cada spec |
| CA-01 a CA-07 | CA-G01 a CA-G09 (este documento) |
| Integración con M1 y M2 (antes implícita) | **CU07, CU08, CU09, CU10** (especificaciones propias) |
