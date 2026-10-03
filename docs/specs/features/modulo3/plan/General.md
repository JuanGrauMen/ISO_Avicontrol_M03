# General — Arquitectura y Stack Tecnológico

**Estado**: Referencia técnica transversal del Módulo 3  
**Fecha**: 02/10/2026  
**Diccionario de Dominio**: [Diccionario.md](../Diccionario.md)  

## Propósito

Este documento define la arquitectura, las tecnologías, la infraestructura de integración (@Scheduled y Kafka) y las reglas técnicas comunes de AviControl Módulo 3 (Liquidación de Lote y Análisis de Rentabilidad). Es la fuente única para estas decisiones: los planes de implementación funcionales deben enlazar este archivo y documentar únicamente su alcance, modelos, contratos y tareas de implementación.

## Stack Tecnológico

| Área | Tecnología adoptada | Uso en el proyecto |
| --- | --- | --- |
| Lenguaje | Java 21 (LTS) | Dominio, casos de uso, adaptadores y pruebas |
| Framework principal | Spring Boot 3.3.x | Configuración, ejecución y composición de la aplicación REST |
| Construcción | Maven (`pom.xml`) | Gestión de dependencias, compilación y empaquetado |
| API HTTP | Spring Web MVC | Controladores REST síncronos |
| Validación | Jakarta Bean Validation | Validación de DTOs en adaptadores de entrada |
| Seguridad | Spring Security | Autenticación JWT y autorización por rol (ej. `ROLE_FINANCIERO`) |
| Persistencia | Spring Data JPA + Hibernate | Repositorios y entidades JPA |
| Base de datos | PostgreSQL (prod) / H2 (dev/test) | Almacenamiento de copia local y liquidaciones |
| Migraciones | Flyway | Versionamiento y aplicación ordenada del esquema SQL |
| Mensajería inter-módulos | Apache Kafka + Spring Kafka | Integración asíncrona por eventos con Módulo 1 y Módulo 2 |
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

---

## Arquitectura Limpia por Capas

El Módulo 3 se implementa con **Arquitectura Limpia por Capas**. Las dependencias siempre apuntan hacia adentro: la infraestructura depende del servicio, y el servicio depende del dominio. El dominio no conoce nada de las capas externas.

```text
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

---

## Infraestructura de Sincronización e Integración

### A. Especificación de `@Scheduled` (Sincronización Periódica)
Módulo 3 ejecuta tareas en segundo plano para mantener actualizada la copia local desde M1 y M2:
- **Configuración Cron**: Expresión configurable vía `application.yml` (ej. `@Scheduled(cron = "${sync.cron.m1:0 */15 * * * *}")`).
- **Prevención de Concurrencia**: Control de ejecución única mediante flags de estado o cerrojos de tareas para evitar ejecuciones superpuestas.
- **Manejo de Excepciones**: Captura de errores de red o timeouts envolviendo la llamada en `try-catch`, registrando el fallo en la bitácora `registro_sincronizacion` sin detener el hilo planificador.
- **Bitácora**: Cada ejecución inserta un registro con `fuente` (`MODULO_1`/`MODULO_2`), `fechaHoraInicio`, `fechaHoraFin`, `resultado` (`EXITOSA`/`FALLIDA`) y `registrosActualizados`.

### B. Especificación de Apache Kafka (`spring-kafka`)
Para comunicación asíncrona orientada a eventos:
- **Nomenclatura de Tópicos**:
  - `avicontrol.m1.alertas-vaciado`: Escucha de alertas de vaciado sanitario emitidas por M1.
  - `avicontrol.m2.resultados-sacrificio`: Escucha de cierres de sacrificio emitidos por M2.
  - `avicontrol.m3.avisos-utilizacion`: Publicación de avisos salientes cuando M3 consume un resultado de sacrificio.
- **Estructura Estándar del Evento JSON**:
  ```json
  {
    "header": {
      "eventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "eventType": "ResultadoSacrificioPublicadoEvent",
      "timestamp": "2026-10-02T15:30:00Z",
      "sourceModule": "MODULO_2"
    },
    "payload": {
      "idResultado": "123e4567-e89b-12d3-a456-426614174000",
      "idLote": "98765432-e89b-12d3-a456-426614174000",
      "idGalpon": "456e7890-e89b-12d3-a456-426614174000",
      "cantidadFinalPollos": 8500,
      "pesoTotalKg": 23800.00
    }
  }
  ```
- **Consumidores y Reintentos**:
  - `groupId`: `avicontrol-modulo3-sync-group`
  - Clave de particionado (`partitionKey`): `idLote` o `idGalpon` para mantener orden estricto de eventos por lote.
  - Estrategia DLT (Dead Letter Topic): 3 reintentos con backoff exponencial. Si persiste el fallo, el evento se mueve a `avicontrol.m3.dlt` para inspecionar.

---

## Formato Estándar de Errores de API REST (RFC 7807 `ProblemDetail`)

Todas las respuestas de error en la API REST del Módulo 3 cumplen con la especificación estándar RFC 7807:

```json
{
  "type": "https://avicontrol.edu.co/errors/liquidacion-ya-existente",
  "title": "Conflicto de Liquidación",
  "status": 409,
  "detail": "El lote 98765432-e89b-12d3-a456-426614174000 ya cuenta con una liquidación en estado ACTIVA (ID: 1045).",
  "instance": "/api/v1/liquidaciones",
  "timestamp": "2026-10-02T16:00:00Z"
}
```

---

## Ciclo de Vida de la Liquidación (Máquina de Estados)

```text
       [ Solicitud POST /api/v1/liquidaciones ]
                         │
                         ▼
                    ┌─────────┐
                    │ CREADA  │ (Transitoria)
                    └────┬────┘
                         │ Validaciones exitosas
                         ▼
                    ┌─────────┐
                    │ ACTIVA  │ ◄── (Estado oficial e inmutable)
                    └────┬────┘
                         │
                         │ Invocación PUT /api/v1/liquidaciones/{id}/anulacion
                         ▼
                    ┌─────────┐
                    │ ANULADA │ ◄── (Estado terminal auditable;
                    └─────────┘      libera el lote para re-liquidar)
```

- **Inmutabilidad**: Una liquidación `ACTIVA` no puede ser editada ni sobrescrita.
- **Anulación**: Pasa a `ANULADA` registrando motivo y usuario. Libera el lote para permitir crear una nueva liquidación `ACTIVA`.

---

## Estrategia de Desarrollo Autónomo (`FakeAdapters`)

Para permitir el desarrollo y pruebas locales sin depender de que M1 o M2 estén levantados:
- En perfil `application-local.yml`, las interfaces `Modulo1Port` y `Modulo2SacrificioPort` se inyectan con implementaciones falsas (`Modulo1FakeAdapter`, `Modulo2SacrificioFakeAdapter`) que retornan datos estáticos precargados en memoria.
