# PENDIENTES.md — lista personal de Andrés

Cosas que voy dejando para después. Se van tachando cuando se resuelven.

## Por leer / revisar

- [ ] Leer `AGENTS.md` completo (versión del 2026-09-21, ya con la unificación de la Matriz).
- [ ] Leer `CLAUDE.md`.
- [ ] Revisar las dos propuestas que no vienen de ningún spec y decidir si se quedan:
  - [ ] Estructura de paquetes (`AGENTS.md` §9: `domain/`, `sync/`, `application/`, `shared/`).
  - [ ] Convención de ramas y commits (`AGENTS.md` §10).
- [ ] Revisar los specs reescritos el 2026-09-21: `spec.md`, CU03, CU04 (completos) y los ajustes en CU01, CU05, CU08.
- [ ] Abrir `docs/diagramas/Modulo3_v1.drawio` en draw.io y acomodar la estética (el nuevo óvalo de sacrificio quedó en `y=990` y el de M1 bajó a `y=1150`).

## Diagrama vs. specs (lo que no se tocó)

- [ ] Desglose: en el diagrama cuelga de *Consultar Lista de Galpones*; en el spec CU05 depende de CU03 (los datos salen de la Liquidación). Decidir si se mueve la flecha en el diagrama o se deja.
- [ ] Las flechas entre casos de uso no tienen `«include»`/`«extend»` (CAMBIOS.md §6, punto 8).
- [ ] Actualizar el diagrama con el nombre real del CU04 (*Anular Liquidación*, hoy dice "Anular liquidacion" sin tilde) y normalizar mayúsculas de los óvalos.

## Primer prototipo de pantallas (decidido el 2026-09-21)

- Pantallas: P1 Lista de Galpones, P2 Generar Liquidación, P3 diálogo Anular, P4 Desglose, P5 Historial. Contenido de cada una: en su spec.
- Decisiones ya llevadas a los specs: usuario fijo sin login (`spec.md` supuestos), acciones en la fila sin detalle intermedio (CU01 FR-004/005), Anular solo desde Generar (CU04 FR-002b), indicador de sincronización en todas las pantallas (RT-09).
- [ ] A criterio del diseñador: layout de la Matriz de Venta Final en P2 y forma visual del indicador de sincronización.
- [ ] **Exportar a Excel (CU05) queda fuera del primer prototipo**; el spec lo sigue exigiendo para la versión final.
- [ ] Cotejar el Figma existente (README) contra el inventario: es anterior a la unificación y seguramente tiene "Registrar Matriz de Ventas".

## Por decidir

- [ ] `docs/AVICONTROL.md` tiene todo el contenido envuelto en un bloque ```` ```markdown ```` y finales de línea CRLF; GitHub lo muestra como código plano. ¿Se limpia o se deja tal cual lo entregó el profesor?
- [ ] Decisiones D-01..D-07 con los equipos de M1 y M2 (ver `docs/specs/features/modulo3/spec.md` y `AGENTS.md` §7–§8). Ninguna se cerró con la unificación; D-01 sigue abierta aunque CU09 ya se llame "Alimento Requerido".
- [ ] Avisar a los equipos de M1 y M2 de los renombres de CU07/CU09/CU10 para que sus diagramas y los nuestros coincidan.
- [ ] `docs/diagramas/Modulo3_v1.drawio.png` (3.5 MB, sin trackear): ¿se sube, se reemplaza por una exportación más liviana o se borra?

## Git

- [x] Todo lo del 2026-09-21 está commiteado en la rama `docs/unificar-matriz-en-liquidacion` y subido.
- [ ] **PR #7 abierto hacia `develop`**: https://github.com/JuanGrauMen/ISO_Avicontrol_M03/pull/7 — pendiente de revisión y merge por el equipo.
- [ ] Decidir si `docs/diagramas/Modulo3_v1.drawio.png` (3,5 MB) se queda en el repo; si no, retirarlo antes del merge.
- [ ] Tras el merge a `develop`, PR de `develop` a `main`.
