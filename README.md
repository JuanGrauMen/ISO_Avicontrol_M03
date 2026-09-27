# AVICONTROL — Módulo 3: Liquidación de Lote y Análisis de Rentabilidad

Sistema para la consolidación de ventas, cálculo de costos operativos, liquidación definitiva de lotes y análisis de rentabilidad financiera por ciclo productivo avícola.

## 🎓 Información Académica

- **Institución:** Universidad del Magdalena
- **Programa:** Ingeniería de Sistemas
- **Asignatura:** Ingeniería de Software (Grupo 2)
- **Docente:** Ing. Juan Manuel Rodríguez Pineda

## 👥 Integrantes del Grupo

- Juan Grau
- Jorge Meléndez
- Andrés Lara
- Daniela Jiménez
- Juan Muñoz

---

## 🎨 Diseño y Prototipos de Interfaz (Figma)

- 🔗 **Prototipo Interactivo en Figma:** [AVICONTROL — Prototipo Interactivo en Figma](https://www.figma.com/proto/bJDPtrQoTxztWIuiC9BrC7/alternativo?node-id=921-1377&starting-point-node-id=921%3A1377&t=d9Aw9XBsMwRcOffT-1)

---

## 📌 Alcance del Módulo 3

El Módulo 3 se encarga de determinar la rentabilidad real de cada lote avícola al cierre de su ciclo de producción.

### Casos de Uso del Módulo:

#### 🧑‍💼 Funcionales (Administrador Financiero)
1. **M3-CU01 – Consultar Lista de Galpones (P1):** Consulta del estado operativo y población viva actual de los galpones.
2. **M3-CU03 – Generar Liquidación del Lote (P1):** Ingreso del precio/kg y cálculo de Venta Bruta, Mortalidad, Costos Operativos y Utilidad Neta, presentados como Matriz de Venta Final.
3. **M3-CU04 – Anular Liquidación (P1):** Flujo formal de anulación de liquidaciones con justificación y auditoría.
4. **M3-CU05 – Consultar Desglose de Ventas y Gastos (P2):** Detalle auditado por partida y exportación a formato Excel (.xlsx).
5. **M3-CU06 – Consultar Historial de Liquidaciones (P3):** Consulta histórica con filtros por fecha y galpón para análisis comparativo entre ciclos.

#### 🔌 Integración Externa (Módulos 1 y 2)
6. **M3-CU07 – Consultar Galpón y Lote al Módulo 1 (P1)**
7. **M3-CU08 – Consultar Resultado Final de Sacrificio al Módulo 2 (P1)**
8. **M3-CU09 – Consultar Alimento Requerido al Módulo 2 (P1)**
9. **M3-CU10 – Consultar Consumo de Medicamento al Módulo 2 (P1)**

---

## 📐 Metodología de Desarrollo

Este proyecto sigue el marco de **Specification-Driven Development (SDD)**:
1. **Fase 1 (SPEC):** Especificaciones atómicas y autocontenidas (`specs/features/modulo3/`).
2. **Fase 2 (PLAN):** Diseño técnico de arquitectura, modelos y contratos de integración.
3. **Fase 3 (Implementación):** Desarrollo en Java 17, Maven y pruebas unitarias con JUnit 5.
