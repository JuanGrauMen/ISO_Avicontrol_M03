# Registro de Cambios — Specs del Módulo 3

**Fecha**: 2026-09-18 (§1–§6) · 2026-09-21 (§7)  
**Alcance**: Reestructuración de las especificaciones de `specs/features/modulo3/` de 5 a 10 casos de uso (§1–§6) y unificación de la Matriz de Ventas en la Liquidación (§7).

---

## Cómo aplicar

El contenido de esta carpeta reemplaza íntegramente `specs/features/modulo3/`.

Carpetas que **desaparecen** (su contenido se trasladó y corrigió):

- `m3-cu03-generar-matriz-ventas/` → `m3-cu03-generar-liquidacion/`
- `m3-cu04-desglose-ventas-gastos/` → `m3-cu05-desglose-ventas-gastos/`
- `m3-cu05-historial-reportes/` → `m3-cu06-historial-liquidaciones/`

Conviene borrar las tres carpetas viejas en el mismo commit, para que Git registre los renombrados y no queden specs duplicados.

---

## 1. Problema conceptual resuelto: glosario normativo

Se agregó un **glosario** al `spec.md` índice con definiciones únicas y excluyentes. Cada término nombra exactamente un artefacto y cada artefacto tiene exactamente un nombre.

- **Matriz de Ventas** — documento de **ingreso**: qué se vendió y a qué precio. Responde *¿cuánto entró?*
- **Liquidación** — documento de **resultado**: ingreso menos costos. Responde *¿cuánto quedó?*
- **Desglose** — **proyección de solo lectura** sobre la Liquidación. No es documento ni entidad.
- **Documento Financiero** — abstracción que agrupa a Matriz de Ventas y Liquidación.
- **Anulación** — único mecanismo de corrección admitido.

Se incluye además una tabla de distinción obligatoria entre Matriz de Ventas y Liquidación, y la regla que las relaciona: *la Liquidación consume la Matriz de Ventas; la Matriz no conoce a la Liquidación.*

### Términos eliminados del módulo

| Término | Motivo | Reemplazo |
| :--- | :--- | :--- |
| "Reporte financiero" | Sinónimo no declarado de Liquidación; nunca definido | **Liquidación** |
| "Generar Matriz de Ventas" (nombre de CU03) | El caso de uso generaba una Liquidación, no una matriz | **Generar Liquidación del Lote** |
| "Pérdida por Mortalidad" | Se calculaba en aves y porcentaje, no en pesos; no es una pérdida monetaria | **Mortalidad del Lote** |

---

## 2. Casos de uso nuevos

### M3-CU04 — Anular Documento Financiero
La anulación estaba enterrada en dos specs distintas (CU02.FR-006/FR-007 y CU03.FR-008) con entidades separadas. Es una acción del Administrador Financiero con actor, precondición, confirmación y auditoría propias, por lo que pasa a ser caso de uso independiente.

Al extraerla aparecieron dos reglas que ninguna spec definía:

- **RT-05 — Orden de anulación**: no puede anularse una Matriz de Ventas con Liquidación `ACTIVA` asociada. Primero la Liquidación, después la Matriz.
- **RT-06 — Independencia**: anular una Liquidación no anula la Matriz de Ventas.

Sin RT-05, anular solo la Liquidación deja el lote atrapado: la matriz errónea sigue `ACTIVA`, CU02 impide registrar otra, y toda nueva Liquidación reproduce el mismo error.

### M3-CU07 a M3-CU10 — Integración
Las cuatro consultas externas se documentan como casos de uso propios. Antes vivían como supuestos dispersos dentro de CU01 y CU03.

| Spec | Fuente | Qué obtiene |
| :--- | :--- | :--- |
| M3-CU07 | Módulo 1 | Galpones, lotes, población inicial y actual, costo total, alertas de vaciado |
| M3-CU08 | Módulo 2 | Resultado final de sacrificio + aviso de utilización |
| M3-CU09 | Módulo 2 | Partidas de alimento del ciclo |
| M3-CU10 | Módulo 2 | Consumos de medicamento con desglose por recepción |

**M3-CU08 no estaba en el diagrama.** El diagrama dibuja alimento y medicina desde Módulo 2, pero omite el resultado final de sacrificio, que es el único insumo físico de la Matriz de Ventas. Sin él, CU02 no puede ejecutarse. Hay que agregar esa elipse.

---

## 3. Correcciones de fondo

### Fórmula de Venta Bruta
**Antes**: `pollosVendidos × pesoPromedioKg × precioKg`  
**Ahora**: `pesoTotalKg × precioKg`

El peso promedio es un valor **calculado**. En el Caso Dorado da exacto (23.800 / 8.500 = 2,8) pero con 23.801 kg el cociente es periódico y el resultado cambia según la precisión que se conserve. Multiplicar sobre el peso total es matemáticamente equivalente y no pierde nada. El peso promedio queda como dato de presentación.

### Fórmula de Mortalidad
**Antes**: `poblaciónInicial − pollosVendidos`  
**Ahora**: `poblaciónInicial − poblaciónActual`

La fórmula anterior mezclaba aves muertas con aves no comercializadas (descartes, decomisos) y sobreestimaba la mortalidad. Ambas poblaciones provienen de Módulo 1, que mantiene la población actual con base en la mortalidad que le reporta Módulo 2.

### Siniestro total
CU03.FR-005 ahora declara explícitamente `pollosVendidos = 0` y `ventaBruta = 0` cuando no existe Matriz de Ventas. Antes el escenario existía pero el requisito no definía el valor.

### Entidades eliminadas
| Entidad | Motivo |
| :--- | :--- |
| `DesgloseLiquidacion` | Almacenar el desglose aparte permite que deje de cuadrar con la Liquidación, que es justo lo que CU05.FR-002 prohíbe. Pasa a ser proyección derivada. |
| `HistorialLiquidacion` | Es una consulta filtrada sobre `Liquidacion`, no una entidad de dominio. |
| `CriteriosFiltroHistorial` | Es un conjunto de parámetros de búsqueda. |
| `RegistroAnulacionVenta` y `RegistroAnulacionLiquidacion` | Unificadas en `RegistroAnulacion` (M3-CU04). |
| `PrecioInsumo` | Creaba una segunda fuente para el costo de pollitos, que ya viene de Módulo 1. Los precios vienen ahora con cada partida sincronizada. |

### Tabla de trazabilidad
Regenerada contra la numeración `FR-xxx` vigente. La anterior usaba nomenclatura `RF-xxx` y apuntaba a numeraciones previas al refactor de septiembre.

---

## 4. Modelo de integración declarado

Se agregó al índice una sección normativa: **M3 consulta; los módulos de origen no envían información a M3.** M3 sincroniza en segundo plano hacia una copia local, y ninguna operación funcional requiere consulta en vivo.

**Única excepción**: el aviso de utilización del resultado final de sacrificio (M3-CU08.FR-006). Módulo 2 condiciona la corrección de ese resultado a que M3 no lo haya usado, así que necesita enterarse. Es asíncrono, idempotente, y su fallo nunca revierte una Liquidación ya generada.

---

## 5. Decisiones pendientes con otros módulos

Marcadas como `NEEDS CLARIFICATION` en cada spec y listadas en el índice. Son los bloqueantes reales del módulo.

| # | Decisión | Contraparte |
| :--- | :--- | :--- |
| D-01 | Criterio de costeo del alimento: compras, consumo real o proyección | Módulo 2 |
| D-02 | Exposición formal de los datos de Módulo 1 hacia Módulo 3 | Módulo 1 |
| D-03 | Mecanismo del aviso de utilización del resultado de sacrificio | Módulo 2 |
| D-04 | Ruta de estado para liquidar un lote con mortalidad total | Módulo 1 |
| D-05 | ¿Los costos operativos incluyen impuesto o son sobre valor neto? | Módulo 2 |
| D-06 | Mortalidad ocurrida fuera del estado `Productivo` | Módulo 1 |
| D-07 | Grafía exacta del catálogo de estados compartido | Módulo 1 |

### Sustento de D-01
Módulo 2 sostiene dos criterios contradictorios entre sus propias specs: costeo por consumo real con precio histórico, y costeo por requerimiento proyectado con precio vigente. Además **no cuenta con ninguna spec que registre el alimento efectivamente suministrado**, a diferencia de lo que sí hace con medicamentos. Por eso M3-CU09 se redactó en términos neutros ("partida de alimento"): la elección de la fuente no altera el resto de la especificación.

### Sustento de D-04
Módulo 1 acepta la alerta de vaciado sanitario únicamente desde el estado `En cosecha`, y declara explícitamente que la población actual en cero no inicia el vaciado. Un lote con mortalidad total en `Productivo` nunca alcanza el estado que habilita la Liquidación, lo que hace inalcanzable el escenario de siniestro total de M3-CU03.

---

## 6. Pendiente: actualizar el diagrama *(revisado el 2026-09-21 — ver §7)*

El diagrama de casos de uso debe reflejar la nueva estructura:

1. Agregar **Anular Documento Financiero**.
2. Agregar la elipse faltante del **resultado final de sacrificio** desde Módulo 2.
3. Renombrar "generar matriz de ventas" → **Generar Liquidación del Lote**.
4. Renombrar "Consultar historial de reportes financieros" → **Consultar Historial de Liquidaciones**.
5. Corregir la flecha invertida: el orden es `registrar → generar`, no al revés.
6. Colgar el **desglose** de la Liquidación, no de la lista de galpones.
7. Cambiar el verbo de las elipses de integración: "Reportar de…" → **"Consultar… al Módulo N"**, acorde al modelo de consulta.
8. Marcar las relaciones entre casos de uso con `«include»`. Hoy son asociaciones simples, que en UML solo aplican entre actor y caso de uso.

Estado al 2026-09-21: se aplicaron 2 y 7 en `docs/diagramas/Modulo3_v1.drawio` (resultado final de sacrificio agregado; integraciones como "Consultar … al Módulo N" con M1 y M2 como `«system»`). Los puntos 1, 3, 4 y 5 quedaron sin objeto tras §7. Siguen pendientes 6 y 8.

---

## 7. Unificación de la Matriz de Ventas en la Liquidación (2026-09-21)

### Motivo

El documento raíz de AVICONTROL (`docs/AVICONTROL.md`, §3.2 *"Matriz de Venta Final (Ingreso Neto)"*) define la Matriz de Ventas como la **tabla de resultado** que el usuario ve: Venta Bruta, Pérdida por Mortalidad, Costos Operativos y Utilidad Neta. Las specs del 2026-09-18 la habían convertido en un **documento de ingreso intermedio** (M3-CU02) que se registraba antes de liquidar, con estado propio, anulación propia y reglas de orden frente a la Liquidación (RT-05, RT-06). Ese paso intermedio no existe en el documento raíz ni en el diagrama de casos de uso del módulo, y solo servía para capturar el precio por kilogramo.

### Qué cambia

| Elemento | Antes | Ahora |
| :--- | :--- | :--- |
| **Matriz de Ventas** | Documento de ingreso, entidad `MatrizVentas`, estado `ACTIVA`/`ANULADA` | **Matriz de Venta Final**: presentación tabular de la Liquidación. No es entidad ni documento |
| **M3-CU02 – Registrar Matriz de Ventas** | Caso de uso propio | **Eliminado.** El precio por kg se ingresa en M3-CU03 (FR-011). La carpeta desaparece; el ID CU02 queda vacante para no renumerar CU03–CU10 |
| **M3-CU03** | Consumía una Matriz `ACTIVA` | Consume directamente el resultado final de sacrificio (M3-CU08) + precio por kg ingresado. Nuevos FR-011 a FR-013; `Liquidacion` absorbe `pollosVendidos`, `pesoTotalKg`, `pesoPromedioKg`, `precioKgCop`, `idResultadoSacrificio` |
| **M3-CU04** | *Anular Documento Financiero* (Matriz o Liquidación) | **Anular Liquidación.** Carpeta `m3-cu04-anular-liquidacion/`. Se eliminan `DocumentoFinanciero`, `tipoDocumento` y los escenarios de orden de anulación |
| **RT-05, RT-06** | Orden e independencia de anulación entre Matriz y Liquidación | **Eliminadas.** Se conservan los números para no renumerar RT-07 y RT-08 |
| **CA-G09** | Orden de anulación | **Eliminado** |
| **CA-G01, CA-G02, CA-G04, CA-G08, SC-006** | Contaban 10 CU y dos tipos de documento | Cuentan 9 CU y un único documento |
| **M3-CU01, M3-CU05, M3-CU08** | Referencias a "registrar la Matriz" y a `MatrizVentas` | Referencias a "generar la Liquidación" y a los datos de venta de `Liquidacion` |
| **Documento Financiero** (glosario) | Agrupaba Matriz y Liquidación | Reservado para la Liquidación |

### Renombres para coincidir con los diagramas de M1 y M2

Los casos de uso de integración adoptan el nombre del caso de uso que el módulo de origen expone hacia M3:

| ID | Antes | Ahora | Contraparte |
| :--- | :--- | :--- | :--- |
| M3-CU07 | Consultar Población y Costo del Lote al Módulo 1 | **Consultar Galpón y Lote al Módulo 1** | M1: *Consultar galpón y/o lote* |
| M3-CU09 | Consultar Alimento del Lote al Módulo 2 | **Consultar Alimento Requerido al Módulo 2** | M2: *Consultar alimento requerido por galpón* |
| M3-CU10 | Consultar Medicina Consumida al Módulo 2 | **Consultar Consumo de Medicamento al Módulo 2** | M2: *Registrar consumo medicamento* |

Las carpetas se renombran en consecuencia. El nuevo nombre de M3-CU09 **no cierra D-01**; el contenido de la spec sigue en términos neutros.

### Qué no cambia

- Fórmulas de Venta Bruta (sobre peso total), Mortalidad (sobre poblaciones de M1) y Utilidad Neta.
- Modelo de integración (M3 consulta; única escritura: aviso de utilización, M3-CU08).
- Decisiones D-01 a D-07: todas siguen abiertas.
- Siniestro total: sigue liquidándose sin resultado de sacrificio y ahora también sin precio por kg.

