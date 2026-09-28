# General — Arquitectura y Stack Tecnológico

**Estado**: Referencia técnica transversal del Módulo 3  
**Fecha**: 28/09/2026  

## Propósito

Este documento define la arquitectura, las tecnologías y las reglas técnicas comunes de AviControl Módulo 3 (Liquidación de Lote y Análisis de Rentabilidad). Es la fuente única para estas decisiones: los planes técnicos de cada caso de uso (specs) deben enlazar este archivo y documentar únicamente su alcance, modelos, reglas específicas y tareas de implementación.

## Stack Tecnológico

| Área | Tecnología adoptada | Uso en el proyecto |
| --- | --- | --- |
| Lenguaje | Java 21 (LTS) | Dominio, servicios, adaptadores y pruebas |
| Framework principal | Spring Boot 3.3.x | Configuración, ejecución y composición de la aplicación |
| Construcción | Maven (`pom.xml`) | Gestión de dependencias, compilación, pruebas y empaquetado |
| API HTTP | Spring Web MVC | Controladores REST API síncronos (`API/`) |
| Eventos entre módulos | Apache Kafka (`spring-kafka`) | Integración asíncrona y transmisión de eventos entre módulos |
| Validación | Jakarta Bean Validation | Validación de DTOs en la capa de entrada `API` |
| Seguridad | Spring Security | Autenticación JWT, protección de endpoints y autorización por rol (`ROLE_FINANCIERO`) |
| Persistencia | Spring Data JPA e Hibernate | Adaptadores de persistencia relacional (`adapters/`) |
| Base de datos | PostgreSQL (prod) / H2 (dev/test) | Almacenamiento de copia local (M1/M2) y registros de liquidación |
| Migraciones | Flyway (`flyway-core`) | Versionamiento y aplicación ordenada del esquema SQL |
| Exportación de reportes | Apache POI `poi-ooxml:5.2.5` | Generación de reportes en Excel de desglose de ventas y gastos (CU05) |
| Serialización | Jackson | JSON para la API REST y contratos de integración con Kafka |
| Documentación API | OpenAPI 3 (`springdoc-openapi`) | Descripción verificable e interactiva de endpoints y DTOs |
| Pruebas unitarias | JUnit 5 y Mockito | Reglas de dominio y servicios aislados |
| Pruebas HTTP / Integración | Spring Boot Test y MockRestServiceServer | Contratos de API REST, clientes M1/M2 y controladores |
| Pruebas de integración | Testcontainers (PostgreSQL) y H2 | Pruebas de integración contra base de datos real / H2 en memoria |
| Utilidades | Lombok | Reducción de código repetitivo sin ocultar reglas de negocio |
| Calidad y arquitectura | Spotless y ArchUnit | Formato uniforme de código y verificación de límites arquitectónicos |
| Observabilidad | Spring Boot Actuator y Micrometer | Salud (`/actuator/health`), métricas y diagnóstico operativo |
| Entorno local | Docker Compose | PostgreSQL y Apache Kafka para desarrollo y pruebas locales |

## Arquitectura Limpia por Capas (Clean Architecture)

El Módulo 3 adopta una **Arquitectura Limpia por Capas** basada en círculos concéntricos. La capa de dominio está en el centro, rodeada por los servicios de aplicación y, en el círculo exterior, la infraestructura (adaptadores, API REST, Spring Boot y Kafka).

### Diagrama de Capas Concéntricas

```text
       ┌────────────────────────────────────────────────────────┐
       │                 INFRASTRUCTURE                         │
       │  ┌──────────────────────────────────────────────────┐  │
       │  │                   SERVICE                        │  │
       │  │  ┌────────────────────────────────────────────┐  │  │
       │  │  │                 DOMAIN                     │  │  │
       │  │  │  ┌──────────────────┐ ┌─────────────────┐  │  │  │
       │  │  │  │     entities     │ │   repository    │  │  │  │
       │  │  │  │  (Liquidacion)   │ │  (interfaces)   │  │  │  │
       │  │  │  └──────────────────┘ └─────────────────┘  │  │  │
       │  │  └────────────────────────────────────────────┘  │  │
       │  │             LiquidacionService                   │  │
       │  └──────────────────────────────────────────────────┘  │
       │        LiquidacionDataAdapter  │  LiquidacionController │
       │        Spring Boot             │  Apache Kafka          │
       └────────────────────────────────────────────────────────┘
```

### Estructura de Paquetes en el Proyecto (`module3`)

```text
src/main/java/co/edu/unimagdalena/avicontrol/
├── domain/
│   ├── entities/               # Entidades de dominio puras (Liquidacion, Galpon, Lote, etc.)
│   └── repository/             # Interfaces de repositorios (LiquidacionRepository, GalponRepository, etc.)
├── service/                    # Servicios de aplicación con reglas de negocio (LiquidacionService, etc.)
└── infraestructure/
    ├── adapters/               # Adaptadores de persistencia JPA y comunicación M1/M2 (LiquidacionDataAdapter, etc.)
    └── API/                    # Controladores REST API (LiquidacionController), Spring Security y Kafka Producers/Consumers
```

### Descripción de las Capas

1. **`domain` (Círculo Central)**:
   - **`entities`**: Contiene las entidades puras de negocio (`Liquidacion`, `Galpon`, `Lote`, etc.) y las reglas numéricas/redondeo. No depende de Spring, JPA ni librerías externas.
   - **`repository`**: Interfaces que definen los contratos de persistencia y consulta (`LiquidacionRepository` con métodos `save`, `update`, `delete`, `find`).

2. **`service` (Círculo Intermedio)**:
   - Contiene la lógica de aplicación y los casos de uso (`LiquidacionService` con `generarLiquidacion()`, `anularLiquidacion()`, etc.).
   - Coordina las entidades del dominio y hace uso de las interfaces de `repository` sin conocer los detalles de la base de datos o el transporte HTTP/Kafka.

3. **`infraestructure` (Círculo Exterior)**:
   - **`adapters`**: Implementaciones concretas de los repositorios (`LiquidacionDataAdapter` usando Spring Data JPA / Hibernate) y adaptadores de clientes externos.
   - **`API`**: Punto de entrada del sistema. Contiene los controladores REST (`LiquidacionController`), endpoints OpenAPI, interceptores de Spring Security y escuchadores/productores de eventos de Apache Kafka.

## Regla de Dependencias

```text
infraestructure  ──►  service  ──►  domain
```

- **Límites de dependencias**:
  - `domain` no depende de ninguna otra capa.
  - `service` depende únicamente de `domain`.
  - `infraestructure` depende de `service` y `domain` para exponer la API y conectar la persistencia.
- **Verificación automatizada**: ArchUnit valida en el build que `domain` no importe paquetes de `service` o `infraestructure`.
