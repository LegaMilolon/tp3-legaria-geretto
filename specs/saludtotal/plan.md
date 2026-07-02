# Implementation Plan: SaludTotal — Historia Clínica Unificada

**Branch**: `feat/saludtotal-backend` | **Date**: 2026-07-01 | **Specs**: [specs/saludtotal/](.)

**Input**: Cinco specs clarificadas: Historia Clínica Digital, Seguridad, Gestión de Pacientes, Panel de Administración, Monitoreo Biométrico Predictivo.

## Summary

Backend REST API de un sistema de historia clínica unificada para una red de clínicas. Médicos registran evoluciones de pacientes desde tablets. El sistema incluye gestión de usuarios con RBAC (3 roles), autenticación JWT stateless, auditoría de accesos y un módulo opcional de monitoreo biométrico predictivo con score simulado por batch diario. Stack: Java 21 + Spring Boot + Hibernate/MySQL 8 + JWT + Gradle.

## Technical Context

**Language/Version**: Java 21

**Primary Dependencies**: Spring Boot (latest stable), Spring Data JPA (Hibernate), Spring Security, springdoc-openapi (Swagger Bearer JWT), Lombok, jjwt (JWT), JUnit 5, Mockito

**Storage**: MySQL 8 — esquema autogenerado por Hibernate (`ddl-auto=create`), datos semilla via `DataInitializer`

**Testing**: JUnit 5 + Mockito (unitarios en capa Service); tests de integración para auth y endpoints críticos

**Target Platform**: Servidor Linux / JVM 21

**Project Type**: REST API backend (web-service)

**Performance Goals**: Búsqueda de pacientes < 2 s para 10.000 registros; alertas biométricas < 5 min post-batch

**Constraints**: JWT stateless (sin sesiones); sin hardcoding de credenciales; contraseñas BCrypt; `ddl-auto=create` solo en dev

**Scale/Scope**: Prototipo universitario — ~10.000 pacientes de prueba, ~3 roles, ~50 usuarios del sistema

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Estado | Notas |
|---|---|---|
| Java 21 | ✅ | Stack confirmado |
| Spring Boot latest | ✅ | Ninguna versión alternativa introducida |
| Hibernate / Spring Data JPA | ✅ | Sin scripts SQL manuales; `ddl-auto=create` en dev |
| MySQL 8 | ✅ | Collation `utf8mb4_unicode_ci` para búsqueda tolerante a tildes |
| Spring Security + JWT stateless | ✅ | Claims: `dni`, `email`, `rol`; sin cookies de sesión |
| Lombok | ✅ | Para reducción de boilerplate en entidades y DTOs |
| springdoc-openapi + Swagger Bearer | ✅ | Todos los endpoints documentados |
| JUnit 5 + Mockito | ✅ | Tests unitarios en capa Service |
| Gradle Groovy DSL | ✅ | `build.gradle` (no Kotlin DSL) |
| Sin hardcoding | ✅ | `${DB_USER}`, `${DB_PASS}`, `${JWT_SECRET}` via env vars |
| BCrypt passwords | ✅ | `password` nunca en respuestas API |
| Estructura controller→service→repository→entity | ✅ | Ver Project Structure |
| DTOs para API (no exponer entidades) | ✅ | Paquete `dto/` separado |

**Resultado**: ✅ Sin violaciones. Continuar a Phase 1.

## Project Structure

### Documentation (este proyecto)

```text
specs/saludtotal/
├── plan.md              ← este archivo
├── research.md          ← decisiones técnicas y resolución de dudas
├── data-model.md        ← entidades JPA + relaciones + diagrama ER
├── quickstart.md        ← instrucciones para levantar el proyecto
└── contracts/
    ├── auth.md
    ├── pacientes.md
    ├── evoluciones.md
    ├── usuarios.md
    └── biometrico.md
```

### Source Code (repository root)

```text
src/
├── main/
│   ├── java/com/saludtotal/
│   │   ├── SaludTotalApplication.java
│   │   ├── config/
│   │   │   ├── SecurityConfig.java          # Spring Security + JWT filter chain
│   │   │   ├── OpenApiConfig.java           # Swagger con Bearer JWT
│   │   │   └── DataInitializer.java         # Semilla de datos (CommandLineRunner)
│   │   ├── controller/
│   │   │   ├── AuthController.java          # POST /api/auth/login
│   │   │   ├── PacienteController.java
│   │   │   ├── EvolucionController.java
│   │   │   ├── UsuarioController.java
│   │   │   └── BiometricoController.java
│   │   ├── service/
│   │   │   ├── AuthService.java
│   │   │   ├── PacienteService.java
│   │   │   ├── EvolucionService.java
│   │   │   ├── UsuarioService.java
│   │   │   ├── BiometricoService.java
│   │   │   └── ScoreBatchService.java       # @Scheduled batch diario 00:00
│   │   ├── repository/
│   │   │   ├── PacienteRepository.java
│   │   │   ├── EvolucionRepository.java
│   │   │   ├── UsuarioRepository.java
│   │   │   ├── EventoAuditoriaRepository.java
│   │   │   ├── SesionBiometricaRepository.java
│   │   │   ├── EventoTipeoRepository.java
│   │   │   ├── ScoreRiesgoRepository.java
│   │   │   └── AlertaBiometricaRepository.java
│   │   ├── entity/
│   │   │   ├── Paciente.java
│   │   │   ├── EvolucionClinica.java
│   │   │   ├── AdjuntoClinico.java
│   │   │   ├── Usuario.java
│   │   │   ├── EventoAuditoria.java
│   │   │   ├── SesionBiometrica.java
│   │   │   ├── EventoTipeo.java
│   │   │   ├── ScoreRiesgo.java
│   │   │   └── AlertaBiometrica.java
│   │   ├── dto/
│   │   │   ├── request/
│   │   │   │   ├── LoginRequest.java
│   │   │   │   ├── PacienteRequest.java
│   │   │   │   ├── EvolucionRequest.java
│   │   │   │   ├── UsuarioRequest.java
│   │   │   │   └── ResetPasswordRequest.java
│   │   │   └── response/
│   │   │       ├── LoginResponse.java       # { token: "..." }
│   │   │       ├── PacienteResponse.java
│   │   │       ├── EvolucionResponse.java
│   │   │       ├── UsuarioResponse.java     # sin campo password
│   │   │       └── ScoreRiesgoResponse.java
│   │   ├── enums/
│   │   │   ├── Rol.java                     # MEDICO, RECEPCIONISTA, ADMIN
│   │   │   ├── EstadoEvolucion.java         # BORRADOR, CONFIRMADO, OCULTO
│   │   │   └── NivelRiesgo.java             # BAJO, MEDIO, ALTO
│   │   ├── exception/
│   │   │   ├── GlobalExceptionHandler.java  # @RestControllerAdvice
│   │   │   ├── ResourceNotFoundException.java
│   │   │   ├── ConflictException.java
│   │   │   └── ForbiddenException.java
│   │   └── security/
│   │       ├── JwtUtil.java                 # generar/validar tokens
│   │       ├── JwtFilter.java               # OncePerRequestFilter
│   │       └── UserDetailsServiceImpl.java
│   └── resources/
│       └── application.properties           # con ${DB_USER}, ${DB_PASS}, ${JWT_SECRET}
└── test/
    └── java/com/saludtotal/
        ├── service/
        │   ├── PacienteServiceTest.java
        │   ├── EvolucionServiceTest.java
        │   ├── UsuarioServiceTest.java
        │   └── AuthServiceTest.java
        └── integration/
            ├── AuthIntegrationTest.java
            └── PacienteIntegrationTest.java
```

**Structure Decision**: Single Spring Boot project (Option 1). Layered architecture estándar controller→service→repository→entity. Sin módulos Maven/Gradle multi-proyecto; toda la lógica en un único JAR.

## Complexity Tracking

No hay violaciones a la Constitution. Sin entradas requeridas.
