# AGENTS.md — Guía para trabajar con IA en AVICONTROL Módulo 3

Este archivo define cómo trabajar en este repositorio. Léelo completo antes de cualquier tarea.

## 1. Contexto: AVICONTROL y los tres módulos

AVICONTROL es un sistema de gestión avícola dividido en tres módulos, desarrollados por **tres equipos distintos en tres repositorios distintos**. El documento raíz del profesor está en `docs/AVICONTROL.md`.

| Módulo | Responsabilidad | Repositorio |
| :--- | :--- | :--- |
| M1 — Gestión de Galpones e Infraestructura | Galpones, lotes, población, estados del galpón, costo del lote | https://github.com/andr4dev/AVICONTROL |
| M2 — Nutrición, Sanidad e Inventario Vivo | Dieta por etapa, mortalidad, medicamentos, inventario de bodega, sacrificio | https://github.com/OscarHernandez123/AviControlMod2 |
| **M3 — Liquidación de Lote y Análisis de Rentabilidad** | **Este repositorio.** Liquidación (con su Matriz de Venta Final), anulación, desglose, historial | https://github.com/JuanGrauMen/ISO_Avicontrol_M03 |

**Universidad del Magdalena — Ingeniería de Software.** Proyecto académico; el objetivo es que los cálculos sean auditables y que cada línea de código se trace a un spec.

## 2. Jerarquía de fuentes de verdad

Cuando dos documentos se contradicen, manda el de más arriba:

1. `docs/specs/features/modulo3/spec.md` — índice **normativo** del módulo: glosario, RT-01..RT-08, modelo de integración, CA-G, SC, decisiones D-01..D-07.
2. `docs/specs/features/modulo3/m3-cuNN-*/spec.md` — spec autocontenido de cada caso de uso.
3. `docs/specs/features/modulo3/plan/plan-v1.md` — **plan técnico general** del módulo: arquitectura, clases, contratos de los puertos, estrategia de pruebas, tareas por CU y supuestos sobre D-01..D-07. Define el *cómo*; nunca contradice a un spec.
4. `docs/specs/features/modulo3/CAMBIOS.md` — justificación de cada desviación respecto a versiones anteriores y al documento raíz.
5. `docs/AVICONTROL.md` — documento raíz del profesor. Define el alcance global de los tres módulos, **pero el spec de M3 se aparta de él deliberadamente** en los puntos siguientes. No "corrijas" el código hacia el documento raíz.

| Tema | Documento raíz | Spec M3 (manda) | Motivo (ver CAMBIOS.md) |
| :--- | :--- | :--- | :--- |
| Venta Bruta | `Pollos finales × Peso promedio × Precio/kg` | `pesoTotalKg × precioKg` | El peso promedio es un cociente con error de redondeo; el peso total es el hecho físico |
| Mortalidad | "Pérdida por Mortalidad" = `Población inicial − Pollos finales` | "Mortalidad del Lote" = `poblaciónInicial − poblaciónActual` (ambas de M1) | Restar vendidos mezcla muertos con descartes; no es pérdida monetaria |
| Costos Operativos | `Alimento + Medicina + Indirectos` | `Costo de Población + Alimento + Medicina`; indirectos **fuera de alcance** (RT-08) | El documento raíz lista el Costo de Población en 3.1 pero lo omite en 3.2 |
| Nombre del resultado | "reporte de rentabilidad" | **Liquidación** | Término "reporte" eliminado del glosario |
| Estados del galpón | 4 estados | 6 estados con la grafía de M1 (§7) | M1 es la fuente oficial (D-07) |

## 3. Stack y comandos

**Java 17 + Maven (jar) + JUnit 5.10.2** (surefire 3.2.5). Sin framework web, sin base de datos, sin linter ni formateador. M3 es hoy una **librería de dominio con pruebas unitarias**.

```bash
mvn test          # ejecuta todas las pruebas — obligatorio antes de dar por terminada una tarea
mvn clean test    # limpiar y testear
mvn package       # compila y empaqueta el jar
```

- **No añadas dependencias al `pom.xml`** (Spring, Lombok, Jackson, POI, etc.) sin acuerdo explícito del equipo. M1 y M2 usan Java 21 + Spring Boot 4.1.1 + PostgreSQL; M3 no, y la alineación es una decisión de equipo, no del agente.
- No subas el nivel de Java ni cambies versiones de plugins por tu cuenta.

## 4. Estructura del repo

```text
AGENTS.md                                  # este archivo
CLAUDE.md                                  # reglas de interacción con el usuario
README.md                                  # contexto académico, integrantes, enlace Figma
pom.xml
docs/
├── AVICONTROL.md                          # documento raíz del profesor (3 módulos)
├── diagramas/                             # Modulo3_v1.drawio (casos de uso) — editar solo si el usuario lo pide
└── specs/
    ├── templates/                         # spec-template.md, plan-template.md, sdd-guide.MD
    └── features/modulo3/
        ├── spec.md                        # ÍNDICE NORMATIVO: glosario, RT, CA-G, SC, D-01..D-07
        ├── plan/plan-v1.md                # PLAN TÉCNICO GENERAL de los 9 CU (arquitectura, clases, tareas), versionado
        ├── CAMBIOS.md                     # registro de cambios conceptuales y sus motivos
        ├── m3-cu01-lista-galpones/spec.md
        ├── m3-cu03-generar-liquidacion/spec.md
        ├── m3-cu04-anular-liquidacion/spec.md
        ├── m3-cu05-desglose-ventas-gastos/spec.md
        ├── m3-cu06-historial-liquidaciones/spec.md
        ├── m3-cu07-consultar-galpon-lote-m1/spec.md
        ├── m3-cu08-consultar-resultado-sacrificio-m2/spec.md
        ├── m3-cu09-consultar-alimento-requerido-m2/spec.md
        └── m3-cu10-consultar-consumo-medicamento-m2/spec.md
src/main/java/co/edu/unimagdalena/avicontrol/   # código (vacío por ahora)
src/test/java/co/edu/unimagdalena/avicontrol/   # pruebas (vacío por ahora)
```

Cada CU tiene su carpeta y su `spec.md` es autocontenido. **No existe M3-CU02**: se unificó en CU03 el 2026-09-21 (ver `CAMBIOS.md` §7); el ID queda vacante a propósito. El PLAN es **uno solo para el módulo** (`plan/plan-v1.md`, decidido el 2026-09-21; las versiones siguientes van en la misma carpeta como `plan-v2.md`, etc., y manda la de número más alto): cubre los 9 CU con una fase de tareas por cada uno. No se crean planes por CU. En el resto de este archivo, `plan.md` significa la versión vigente del plan.

## 5. Flujo SDD (Specification-Driven Development)

**No se escribe código sin spec y sin plan.** Las tres fases son secuenciales y cada una tiene un artefacto:

1. **SPEC** — leer `m3-cuNN-*/spec.md` completo, más el `spec.md` índice. Si el spec contiene `[NEEDS CLARIFICATION – D-0X]`, ver §8.
2. **PLAN** — leer la fase del CU en `docs/specs/features/modulo3/plan/plan-v1.md` (plan general escrito a partir de `docs/specs/templates/plan-template.md`). Ahí están las clases a crear, la cobertura FR → clase, el mapa acceptance scenario → `@Test`, el fixture `CasoDorado` y los supuestos sobre decisiones abiertas. Si al implementar el plan resulta insuficiente o incorrecto para ese CU, se corrige el `plan.md` (y se revisa con el equipo) antes de seguir; no se improvisa en código.
3. **Implementación** — Java 17 + pruebas JUnit 5 que ejecutan los acceptance scenarios tal cual están escritos. Se implementa un CU por rama, siguiendo las tareas `T0NN` de su fase.

**Orden de implementación:** `CU07 → CU10` (sincronización y copia local) → `CU01` → `CU03` → `CU04` → `CU05` y `CU06` en paralelo.

Sobre `docs/specs/templates/sdd-guide.MD`: es la guía metodológica general del curso. Sus recomendaciones de herramientas y modelos (Cursor, Fury, GPT, etc.) **no aplican** a este repo; sí aplican su definición de SPEC/PLAN y la revisión crítica del plan.

## 6. Reglas normativas (resumen — el detalle está en `spec.md`)

**Glosario.** Cada término tiene un único significado; está prohibido introducir sinónimos.
- **Liquidación**: **único Documento Financiero** del módulo. Se *genera* en CU03 a partir del resultado final de sacrificio (M2), el precio/kg ingresado por el administrador y los costos de M1 y M2.
- **Matriz de Venta Final** (o Matriz de Ventas): la **tabla de presentación** de una Liquidación (Venta Bruta, Mortalidad, Costos Operativos, Utilidad Neta + datos de venta), como en `docs/AVICONTROL.md` §3.2. **No es entidad ni documento**; no se registra aparte ni tiene estado.
- **Desglose**: proyección de solo lectura sobre una Liquidación. **No es entidad**, no se persiste.
- **Documento Financiero**: término reservado para la Liquidación. Ciclo `ACTIVA → ANULADA`.
- **Anulación**: único mecanismo de corrección (CU04). Nunca edita ni borra.
- Siniestro total (población actual 0): se liquida sin resultado de sacrificio ni precio/kg, con Venta Bruta = 0.

**RT-01..RT-08.** COP con `HALF_UP` en totales enteros (precios unitarios admiten decimales) · porcentajes a 2 decimales · inmutabilidad de `ACTIVA` · anulación con motivo + fecha/hora/usuario · RT-05 y RT-06 **eliminadas** (regulaban la anulación Matriz/Liquidación; los números se conservan) · M3 valoriza (recibe cantidad + precio unitario, nunca totales consolidados) · costos indirectos fuera de alcance.

**Modelo de integración.** M3 **consulta** a M1/M2 en segundo plano hacia una **copia local sincronizada** con fecha/hora de sincronización. Ninguna operación funcional hace consultas en vivo. M3 **no escribe** en M1/M2, salvo el aviso de uso del resultado de sacrificio (M3-CU08.FR-006). Solo CU07–CU10 definen accesos externos.

**Fórmulas.**
- `ventaBruta = pesoTotalKg × precioKg` — nunca sobre peso promedio.
- `mortalidadAves = poblaciónInicial − poblaciónActual`; `%mortalidad = mortalidadAves / poblaciónInicial × 100` (2 decimales).
- `costoAlimento = Σ (kg consumidos de cada recepción × precio histórico por kg)`; igual para medicina por unidad base. **Nunca** se promedian precios entre recepciones.
- `costosOperativos = costoPoblación + costoAlimento + costoMedicina`; `utilidadNeta = ventaBruta − costosOperativos`.

**Criterios globales.** CA-G01..CA-G08 y SC-001..SC-007 del `spec.md` (CA-G09 eliminado). CA-G04 (una sola Liquidación `ACTIVA` por lote) debe tener prueba explícita.

## 7. Qué exponen hoy M1 y M2 (revisado 2026-09-21)

Ningún módulo tiene código funcional todavía; todo proviene de sus **specs**. No existe contrato HTTP, esquema JSON ni autenticación acordada entre módulos. Esta sección es **informativa**: si contradice al `spec.md`, manda el `spec.md`, y hay que abrir la discusión con el otro equipo.

**M1** (`avicontrol/docs/Casos de uso.md`, `spec-consultar-galpón-lote/`, `spec-recibirVaciadoSanitario/`)
- Reconoce a M3 como actor del CU compartido *Consultar galpón y/o lote*.
- **Galpón**: `UUID`, `Nombre`, `Aforo máximo`, `Estado`. **Lote**: `UUID`, `Nombre`, `Población inicial`, `Población actual`, `Fecha de ingreso`, `Costo total` (**entero COP, sin decimales**), `galpon_id`. Lote activo = el de fecha de ingreso más reciente.
- **Estados del galpón (6, grafía de M1)**: `Disponible` · `Productivo` · `En cosecha` · `Vaciado sanitario` · `Mantenimiento` · `Aislamiento`.
- `→ Vaciado sanitario` solo desde `En cosecha` o `Aislamiento`, y solo por alerta de M2 (`UUID alerta`, `UUID galpón`, `UUID lote`, fecha/hora). Población 0 **no** inicia el vaciado → **D-04**. Alertas de mortalidad solo en `Productivo` → **D-06**.
- M1 solo define una pantalla de consulta; no declara servicio para M3 y su `pom.xml` no tiene servidor web → **D-02**.

**M2** (`docs/specs/001, 002, 006, 013, 022`)
- **Resultado de sacrificio** (006): `cantidad final` (entero > 0), `peso total kg` (> 0, **máx. 2 decimales**), `estado de utilización`. Único por lote. Se entrega a M3 "como una única unidad de información". M2 permite corregirlo **hasta que M3 lo use**, y luego lo bloquea: M2 depende del aviso de M3 → **D-03**.
- **Alimento**: M2 ofrece dos caminos incompatibles → **D-01**: (a) spec 001, consumo real valorizado `Σ kg consumidos × precio neto histórico por kg` por recepción (coincide con RT-07 y M3-CU09); (b) spec 022, servicio HTTP con requerimiento **proyectado** por etapa y `costoUnitarioKg` **vigente** (no cumple RT-07).
- **Medicamentos** (002/013): `Σ cantidad aplicada × precio neto histórico por unidad base` (`gr`, `ml`, `unidad`).
- Ambos llegan con `precio neto` + `impuesto` separados; M2 delega el tratamiento del impuesto a M3 → **D-05**.
- M3 **no** consume mortalidad de M2; la toma como `Población inicial − Población actual` de M1.

## 8. Decisiones pendientes (D-01..D-07) — cómo actuar

El `spec.md` índice lista siete decisiones bloqueantes con otros módulos; los specs las marcan con `[NEEDS CLARIFICATION – D-0X]`. Tras revisar los repos (§7), todas siguen abiertas salvo D-07, que prácticamente se resuelve adoptando la grafía de M1.

Cuando una tarea toque una decisión abierta:

1. **No inventes la respuesta ni la resuelvas en código.** Tampoco la elimines del spec.
2. **Implementa lo que no depende de ella** y deja el punto de decisión aislado en una única clase/método con un supuesto explícito documentado en el `plan.md`. Ejemplo: para D-05, calcular sobre valor neto y dejar el impuesto como partida separada para poder sumarlo después.
3. **Pregunta al equipo** solo si ningún supuesto razonable permite avanzar; en ese caso di exactamente qué decisión te bloquea y qué opciones ves.
4. Si el equipo resuelve una decisión: actualizar el spec del CU (quitar la marca), el `spec.md` índice (tabla de decisiones), `CAMBIOS.md` (motivo) y §7 de este archivo.

## 9. Convenciones de código

**Idioma.** Código (paquetes, clases, métodos, variables, nombres de test) en **inglés**. Documentación, specs, planes, commits y mensajes de error visibles al usuario en **español**. Los nombres de atributos de las Key Entities del spec (`idLote`, `pesoTotalKg`, `precioKgCop`, `ventaBrutaCop`) se conservan **tal cual** para trazabilidad, aunque mezclen idiomas.

**Sin comentarios** en el código salvo que se pida. El nombre debe explicar el propósito.

**Dinero y números.**
- Todo valor monetario es `BigDecimal`; nunca `double`/`float` para COP.
- Los totales se presentan con `setScale(0, RoundingMode.HALF_UP)`; los porcentajes con `setScale(2, RoundingMode.HALF_UP)`.
- Cantidades de aves son `int`/`long`; kilogramos y precios unitarios son `BigDecimal` con la escala que traen de origen.

**Identificadores.** Los UUID de galpón, lote, alerta y resultado de sacrificio son `java.util.UUID`; M3 no los genera, los recibe de M1/M2. Los IDs propios de M3 (`idVenta`, `idLiquidacion`) sí los genera M3.

**Estructura de paquetes** (el detalle clase por clase está en `plan.md`, sección *Project Structure*):

```text
co.edu.unimagdalena.avicontrol
├── domain/          # entidades del glosario: Liquidacion, partidas de costo, estados, fórmulas, elegibilidad
├── sync/            # bitácora, estado de sincronización, planificador; sync/m1 y sync/m2: copia local y puertos (CU07–CU10)
├── application/     # casos de uso CU01, CU03–CU06 como servicios, con sus vistas de solo lectura y excepciones
└── shared/          # Rounding (único lugar de setScale)
```

- Cada entidad del glosario es **una clase** con **ese nombre** (`Liquidacion`). No crear `MatrizVentas`, `Reporte`, `Venta`, `Desglose` como entidades: la Matriz de Venta Final es una vista de `Liquidacion`.
- Los accesos a M1/M2 se modelan como **interfaces (puertos)** en `sync/`; en tests se implementan con dobles en memoria que simulan los tres escenarios de CA-G07: datos completos, datos faltantes, ausencia total.

**Pruebas.**
- Una clase de test por CU: `CU03GenerarLiquidacionTest`, más tests unitarios por clase de dominio si hace falta.
- Cada acceptance scenario del spec es un `@Test` con `@DisplayName("CU03-AS-02 – <título del scenario>")`, en el mismo orden que el spec. Así CA-G02 es verificable con `mvn test`.
- Usar los **valores exactos** del spec (el "Caso Dorado": 9.000 iniciales, 8.500 vendidos, 23.800 kg, etc.). No inventar fixtures paralelos.
- No hay pruebas de UI, de concurrencia (SC-004), de tiempos (SC-002) ni de exportación a Excel (CA-G06) en esta etapa: **fuera de alcance del jar actual**; no añadir dependencias para simularlas.

## 10. Git y ramas

- `main` es la rama estable; `develop` integra. **Nunca commitear directo a `main`.**
- Ramas por tipo: `feature/<cuNN-descripcion>` para código, `docs/<descripcion>` para specs/planes/documentos. Ejemplo real: `docs/actualizar-cu01-integracion-m1-m2`.
- Commits en español, en imperativo, con prefijo: `docs:`, `docs(ui):`, `feat(cu03):`, `test(cu03):`, `fix:`, `refactor:`, `merge:`. Una idea por commit.
- Se integra por Pull Request hacia `develop`, y de `develop` a `main`.
- No hacer `commit`, `push`, `merge` ni crear PRs sin que el usuario lo pida.

## 11. Qué NO hacer

- No trabajar en nada que el usuario no haya pedido directamente (ver `CLAUDE.md`).
- No escribir código sin spec y sin `plan.md` aprobado.
- No modificar un `spec.md` sin registrar el motivo en `CAMBIOS.md`.
- No crear entidades, estados ni términos fuera del glosario. No usar "reporte", "informe", "venta" como sustitutos de Liquidación.
- No implementar edición de Liquidaciones `ACTIVA` (RT-03): solo anulación.
- No reintroducir la Matriz de Ventas como documento o entidad independiente (ver `CAMBIOS.md` §7).
- No calcular Venta Bruta con peso promedio ni mortalidad con pollos vendidos.
- No promediar precios entre recepciones de M2.
- No añadir dependencias ni cambiar el nivel de Java en `pom.xml`.
- No editar `docs/diagramas/` ni `docs/AVICONTROL.md`.
- No resolver decisiones D-01..D-07 por cuenta propia.

## 12. Checklist de fin de tarea

- [ ] `mvn test` ejecutado y todas las pruebas pasan (pegar el resumen de surefire).
- [ ] La fase del CU en `plan.md` está completa (tareas `T0NN` tachadas) y el código cubre todos los FR listados en su tabla de cobertura.
- [ ] Cada acceptance scenario del spec tiene su `@Test` con `@DisplayName("CUNN-AS-XX …")` y pasa con los valores del spec.
- [ ] Glosario, RT-01..RT-08 y modelo de integración respetados; sin entidades ni sinónimos nuevos.
- [ ] Fórmulas correctas: Venta Bruta sobre peso total; Mortalidad sobre poblaciones de M1; sin promedios de precio.
- [ ] `BigDecimal` + `HALF_UP` en todo valor monetario y porcentaje.
- [ ] Decisiones D-0X tocadas: supuesto documentado en `plan.md`, sin resolverlas en código.
- [ ] Si cambió algo conceptual: `CAMBIOS.md` actualizado; si cambió algo de M1/M2: §7 de este archivo actualizado.
- [ ] Código en inglés sin comentarios; docs en español; sin cambios en `pom.xml` ni en `docs/diagramas/`.
- [ ] Trabajo en rama `feature/*` o `docs/*`, nunca en `main`.
