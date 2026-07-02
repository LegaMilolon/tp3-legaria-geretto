# SaludTotal Constitution

## Core Principles

### I. Stack Canónico (NON-NEGOTIABLE)

El backend de SaludTotal se construye exclusivamente con el siguiente stack. No se introducen dependencias fuera de este conjunto sin revisión y aprobación explícita:

| Capa | Tecnología |
|---|---|
| Lenguaje | Java 21 |
| Framework | Spring Boot (última versión estable) |
| Persistencia | Hibernate / Spring Data JPA |
| Base de datos | MySQL 8 |
| Seguridad | Spring Security + JWT (stateless, sin sesiones HTTP) |
| Reducción de boilerplate | Lombok |
| Documentación API | springdoc-openapi (Swagger UI con Bearer JWT) |
| Testing | JUnit 5 + Mockito |
| Build | Gradle (Groovy DSL) |

### II. Base de Datos gestionada por Hibernate

La base de datos se crea y mantiene íntegramente a través de Hibernate — no se escriben scripts SQL manuales de DDL:

- `ddl-auto=create` para entornos de desarrollo/reset; `update` o `validate` para staging/producción.
- Los datos semilla se cargan mediante un `DataInitializer` (implementación de `CommandLineRunner` o `ApplicationRunner`) que se ejecuta al arrancar la aplicación.
- Ninguna migración de esquema se gestiona fuera de las entidades JPA y las propiedades de Hibernate.

### III. Seguridad: JWT Stateless + BCrypt (NON-NEGOTIABLE)

- **Autenticación**: exclusivamente mediante JWT. El token contiene los claims: `dni`, `email` y `rol`. No se usan sesiones HTTP ni cookies de sesión.
- **Contraseñas**: siempre hasheadas con BCrypt antes de persistir. Está **prohibido** almacenar contraseñas en texto plano en ninguna capa (base de datos, logs, respuestas de API).
- **Endpoints protegidos**: todos los endpoints requieren `Authorization: Bearer <token>` **excepto**:
  - `POST /api/auth/login`
  - Rutas de Swagger UI (`/swagger-ui/**`, `/v3/api-docs/**`)

### IV. Sin Hardcoding de Credenciales (NON-NEGOTIABLE)

Ninguna credencial, secret o configuración sensible se escribe directamente en el código fuente o en archivos de configuración versionados. Toda credencial se lee desde variables de entorno:

| Variable | Propósito |
|---|---|
| `${DB_USER}` | Usuario de la base de datos MySQL |
| `${DB_PASS}` | Contraseña de la base de datos MySQL |
| `${JWT_SECRET}` | Clave secreta para firma/verificación de JWT |

Cualquier valor sensible adicional que surja durante el desarrollo sigue el mismo patrón con variable de entorno descriptiva.

### V. Testing Obligatorio

- Toda lógica de negocio en la capa de servicio tiene cobertura con tests unitarios (JUnit 5 + Mockito).
- Los tests de integración cubren los flujos de autenticación y los endpoints críticos.
- No se hace merge de código que rompa tests existentes.
- Los tests no dependen de estado externo (base de datos real, red); se usan mocks para dependencias externas en tests unitarios.

### VI. Simplicidad y Consistencia

- Se sigue la estructura de capas estándar de Spring Boot: `controller` → `service` → `repository` → `entity`.
- Los DTOs se usan para entrada/salida de la API; las entidades JPA no se exponen directamente en los endpoints.
- Se prefiere la solución más simple que cumpla el requisito; no se sobre-ingenieriza.
- Los nombres de clases, métodos y variables siguen las convenciones de Java (camelCase para métodos/variables, PascalCase para clases).

## Restricciones Adicionales

- **Lombok**: se usa para reducir boilerplate (`@Getter`, `@Setter`, `@Builder`, `@NoArgsConstructor`, etc.). No se mezclan anotaciones Lombok con getters/setters manuales en la misma clase.
- **springdoc-openapi**: todos los endpoints públicos tienen descripción con `@Operation` y los modelos tienen `@Schema`. La Swagger UI debe funcionar con autenticación Bearer JWT configurada.
- **Gradle Groovy DSL**: el build script es `build.gradle` (no Kotlin DSL). Las dependencias se declaran en el bloque `dependencies {}` usando el formato estándar de Gradle.
- **Logs**: nunca se loguea información sensible (contraseñas, tokens JWT completos, datos clínicos de pacientes). Se usa SLF4J/Logback a través de las abstracciones de Spring.

## Gobernanza

Esta Constitution es el documento rector del proyecto SaludTotal. En caso de conflicto entre una decisión de implementación y un principio de esta Constitution, la Constitution prevalece.

- Toda desviación de los principios NON-NEGOTIABLE requiere discusión explícita y aprobación antes de implementar.
- Los principios pueden ser enmendados documentando la razón, el impacto y el plan de migración.
- Toda sesión de desarrollo asume esta Constitution como contexto activo.

**Version**: 1.0.0 | **Ratified**: 2026-07-01 | **Last Amended**: 2026-07-01
