# General — Arquitectura y Stack Tecnológico

**Estado**: Referencia técnica transversal del Módulo 3  
**Fecha**: 28/09/2026

## Propósito

Este documento define la arquitectura, las tecnologías y las reglas técnicas comunes de AviControl Módulo 3 (Liquidación de Lote y Análisis de Rentabilidad). Es la fuente única para estas decisiones: los planes técnicos por caso de uso deben enlazar este archivo y documentar únicamente su alcance, modelos, contratos y tareas de implementación.

## Stack Tecnológico

| Área | Tecnología adoptada | Uso en el proyecto |
| --- | --- | --- |
| Lenguaje | Java 21 (LTS) | Dominio, casos de uso, adaptadores y pruebas |
| Framework principal | Spring Boot 3.3.x | Configuración, ejecución y composición de la aplicación REST |
| Construcción | Maven (`pom.xml`) | Gestión de dependencias, compilación y empaquetado |
| API HTTP | Spring Web MVC | Controladores REST síncronos |
| Validación | Jakarta Bean Validation | Validación de DTOs en adaptadores de entrada |
| Seguridad | Spring Security | Autenticación JWT y autorización por rol |
| Persistencia | Spring Data JPA + Hibernate | Repositorios y entidades JPA |
| Base de datos | PostgreSQL (prod) / H2 (dev/test) | Almacenamiento de copia local y liquidaciones |
| Migraciones | Flyway | Versionamiento y aplicación ordenada del esquema SQL |
| Mensajería inter-módulos | Apache Kafka + Spring Kafka | Integración asíncrona con Módulo 1 y Módulo 2 |
| Exportación de reportes | Apache POI `poi-ooxml:5.2.5` | Generación de reportes Excel (CU05) |
| Serialización | Jackson | JSON para API REST y contratos de integración |
| Documentación API | OpenAPI 3 (`springdoc-openapi`) | Descripción interactiva de endpoints en `/swagger-ui.html` |
| Pruebas unitarias | JUnit 5 y Mockito | Reglas de dominio y casos de uso aislados |
| Pruebas de integración | Spring Boot Test + Testcontainers | PostgreSQL y Kafka reales durante las pruebas |
| Utilidades | Lombok | Reducción de código repetitivo |
| Calidad y arquitectura | Spotless y ArchUnit | Formato uniforme y verificación de límites entre capas |
| Observabilidad | Spring Boot Actuator y Micrometer | Salud (`/actuator/health`) y métricas operativas |
| Entorno local | Docker Compose | PostgreSQL y Kafka para desarrollo local |
| Abstracción de tiempo | `java.time.Clock` | Fecha/hora inyectada para auditabilidad y pruebas deterministas |

> **Nota sobre herramientas de construcción**: El Módulo 2 usa Gradle y el Módulo 3 usa Maven. Esto no representa ninguna incompatibilidad: la comunicación entre módulos ocurre exclusivamente a través de la red (HTTP REST y Kafka), por lo que la herramienta de build de cada módulo es un detalle interno sin impacto en la integración.

Las versiones de librerías administradas por Spring Boot se obtienen de su BOM. Solo se fija versión explícita cuando no esté administrada o exista razón técnica documentada.

## Arquitectura Limpia por Capas

El Módulo 3 se implementa con **Arquitectura Limpia por Capas**. Las dependencias siempre apuntan hacia adentro: la infraestructura depende del servicio, y el servicio depende del dominio. El dominio no conoce nada de las capas externas.

```
┌─────────────────────────────────────────┐
│            infrastructure/              │
│  ┌──────────────────────────────────┐   │
│  │   Spring · LiquidacionController │   │
│  │   LiquidacionDataAdapter         │   │
│  │  ┌───────────────────────────┐   │   │
│  │  │       service/            │   │   │
│  │  │   LiquidacionService      │   │   │
│  │  │  ┌─────────────────────┐  │   │   │
│  │  │  │      domain/        │  │   │   │
│  │  │  │  entities:          │  │   │   │
│  │  │  │    Liquidacion      │  │   │   │
│  │  │  │  repository:        │  │   │   │
│  │  │  │    LiquidacionRepo  │  │   │   │
│  │  │  └─────────────────────┘  │   │   │
│  │  └───────────────────────────┘   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Capa `domain/`

Contiene el modelo de negocio puro: entidades (`Liquidacion`, `Galpon`, `Lote`, etc.), enums, value objects, reglas de redondeo e interfaces de repositorio.

- **Regla estricta**: Java puro. No importa Spring, JPA, Jackson, Kafka ni clases de las otras capas.

### Capa `service/` (Aplicación)

Contiene los servicios de aplicación que implementan los casos de uso, orquestan el dominio y coordinan la sincronización periódica con M1 y M2. Define el límite transaccional y no conoce detalles de HTTP, JPA ni Kafka.

### Capa `infrastructure/`

Contiene los adaptadores concretos:

- **`adapter/rest/`**: Controladores REST API (entrada HTTP).
- **`adapter/client/`**: Clientes REST/Kafka para consumir y publicar eventos con Módulo 1 y Módulo 2.
- **`adapter/persistence/`**: Entidades JPA (`@Entity`), repositorios Spring Data y mappers hacia el dominio.
- **`config/`**: Configuración de Spring Security (JWT), OpenAPI, Scheduler, Clock, Actuator.
- **`exception/`**: Manejador global de excepciones (`@ControllerAdvice`).

## Regla de Dependencias

```
infrastructure  ──►  service  ──►  domain
```

- **Dominio**: No depende de ninguna capa.
- **Servicio**: Depende solo del dominio.
- **Infraestructura**: Depende del servicio y del dominio para implementar los adaptadores.

Esta regla es verificada automáticamente por **ArchUnit** en cada build.

## Integración entre Módulos

La comunicación entre el Módulo 3 y los demás módulos del sistema AviControl se realiza por dos mecanismos:

| Mecanismo | Dirección | Uso |
| --- | --- | --- |
| **REST HTTP** (`RestClient`) | M3 → M1, M3 → M2 | Sincronización periódica de datos locales (galpones, lotes, sacrificios, alimentos, medicamentos) |
| **Kafka** (`spring-kafka`) | M2 → M3, M3 → M2 | Publicación y consumo de eventos asíncronos (ej. `ResultadoSacrificioPublicadoEvent`, `LiquidacionGeneradaEvent`) |

> La arquitectura de integración (REST vs. Kafka por canal) puede evolucionar a medida que se acuerden los contratos con los equipos de Módulo 1 y Módulo 2. Los puertos del dominio están diseñados para ser independientes del mecanismo de transporte.
