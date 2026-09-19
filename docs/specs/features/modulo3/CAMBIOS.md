# Registro de Cambios — Specs del Módulo 3

**Fecha**: 2026-09-18  
**Alcance**: Reestructuración de las especificaciones de `specs/features/modulo3/` de 5 a 10 casos de uso.

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

## 6. Pendiente: actualizar el diagrama

El diagrama de casos de uso debe reflejar la nueva estructura:

1. Agregar **Anular Documento Financiero**.
2. Agregar la elipse faltante del **resultado final de sacrificio** desde Módulo 2.
3. Renombrar "generar matriz de ventas" → **Generar Liquidación del Lote**.
4. Renombrar "Consultar historial de reportes financieros" → **Consultar Historial de Liquidaciones**.
5. Corregir la flecha invertida: el orden es `registrar → generar`, no al revés.
6. Colgar el **desglose** de la Liquidación, no de la lista de galpones.
7. Cambiar el verbo de las elipses de integración: "Reportar de…" → **"Consultar… al Módulo N"**, acorde al modelo de consulta.
8. Marcar las relaciones entre casos de uso con `«include»`. Hoy son asociaciones simples, que en UML solo aplican entre actor y caso de uso.
