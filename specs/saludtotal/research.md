# Research: SaludTotal Backend

**Date**: 2026-07-01 | **Phase**: 0 — Decisiones técnicas pre-implementación

## 1. Gestión del JWT y validación de sesiones activas

**Problema**: JWT es stateless por diseño, pero la spec exige invalidar tokens al desactivar una cuenta (spec 02, FR-007) y al cambiar el rol de un usuario (spec 04, FR-011).

**Decisión**: En cada request, el `JwtFilter` no solo valida la firma del token sino que también consulta la base de datos para verificar que el usuario sigue activo (`activo = true`). Esto rompe la pureza stateless pero es el enfoque más simple sin Redis ni blacklist.

```java
// JwtFilter.java — verificación de estado en cada request
String email = jwtUtil.extractEmail(token);
UserDetails user = userDetailsService.loadUserByUsername(email);
if (!((Usuario) user).isActivo()) {
    response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Cuenta desactivada");
    return;
}
```

**Trade-off aceptado**: 1 consulta adicional a DB por request. Aceptable para un prototipo universitario.

---

## 2. Bloqueo pesimista de evoluciones (spec 01, FR-011)

**Problema**: Implementar bloqueo pesimista sin `SELECT FOR UPDATE` explícito en JPA.

**Decisión**: Campo `bloqueadoPor` (FK nullable a `Usuario`) + `timestampBloqueo` en `EvolucionClinica`. Al abrir para edición, se hace un UPDATE atómico que setea el bloqueo solo si `bloqueadoPor IS NULL`. Liberación automática: job `@Scheduled` cada 5 minutos limpia bloqueos con `timestampBloqueo < NOW() - 30 min`.

```sql
-- Lógica de adquisición de bloqueo (via @Modifying en Repository):
UPDATE evoluciones SET bloqueado_por = :medicoId, timestamp_bloqueo = NOW()
WHERE id = :id AND bloqueado_por IS NULL
```

**Alternativa rechazada**: `@Lock(LockModeType.PESSIMISTIC_WRITE)` de JPA — requiere transacción abierta durante toda la edición, inviable en API REST.

---

## 3. Collation MySQL para búsqueda tolerante a tildes (spec 03, FR-008)

**Decisión**: Configurar el collation a nivel de conexión en `application.properties` para que Hibernate cree las tablas con `utf8mb4_unicode_ci`.

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME}?useUnicode=true&characterEncoding=UTF-8&connectionCollation=utf8mb4_unicode_ci
spring.jpa.properties.hibernate.connection.characterEncoding=utf8mb4
spring.jpa.properties.hibernate.connection.useUnicode=true
```

Con este collation, MySQL compara `González = gonzalez = GONZALEZ` automáticamente en queries LIKE. No se requiere `unaccent` ni lógica adicional en Java.

---

## 4. Soft-delete con Hibernate (specs 01, 03, 04)

**Decisión**: Usar `@SQLRestriction("activo = true")` de Hibernate 6 (disponible en Spring Boot 3.x) sobre las entidades `Paciente` y `Usuario`. Esto filtra automáticamente los registros inactivos en todas las queries sin modificar los repositorios.

```java
@Entity
@SQLRestriction("activo = true")
public class Paciente { ... }
```

Para consultar inactivos (admin), usar `@Query` nativa o una segunda entidad `PacienteAdmin` sin la restricción. Alternativamente, un `EntityManager` con `@Filter` de Hibernate.

**Alternativa más simple para el prototipo**: campo `activo` + condición `WHERE activo = true` manual en cada query del repositorio (más explícito, menos magia).

---

## 5. Batch diario de score biométrico (spec 05, FR-009)

**Decisión**: `@Scheduled(cron = "0 0 0 * * *")` en `ScoreBatchService`. El score se simula con `Random.nextInt(101)`. Si score ≥ 70, se crea una `AlertaBiometrica` y se simula el envío de notificación (log en consola en el prototipo; no hay servicio real de push).

```java
@Scheduled(cron = "0 0 0 * * *")
public void calcularScoresDiarios() {
    List<Paciente> pacientes = pacienteRepository.findByMonitoreoActivo(true);
    for (Paciente p : pacientes) {
        int score = new Random().nextInt(101);
        ScoreRiesgo scoreRiesgo = new ScoreRiesgo(p, score, resolverNivel(score));
        scoreRiesgoRepository.save(scoreRiesgo);
        if (score >= 70) {
            generarAlerta(p, scoreRiesgo);
        }
    }
}
```

Habilitar `@EnableScheduling` en la clase de configuración o en `SaludTotalApplication`.

---

## 6. Configuración de Spring Security y rutas públicas

**Decisión**: Configurar el `SecurityFilterChain` para:
- Permitir sin auth: `POST /api/auth/login`, `/swagger-ui/**`, `/v3/api-docs/**`
- Requerir Bearer JWT para todo lo demás
- Permisos por rol via `@PreAuthorize` en los controllers (ej: `@PreAuthorize("hasRole('ADMIN')")`)

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
  .authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/auth/login", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
      .anyRequest().authenticated()
  )
  .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
```

---

## 7. Validación de campos en DTOs

**Decisión**: Usar Jakarta Bean Validation (`@NotBlank`, `@NotNull`, `@Past`, etc.) en los DTOs de request. Habilitar con `@Valid` en los parámetros de los controllers. El `GlobalExceptionHandler` captura `MethodArgumentNotValidException` y retorna HTTP 400 con el detalle de los campos inválidos.

| Campo | Anotación |
|---|---|
| `nombre`, `apellido`, `dni` | `@NotBlank` |
| `fechaNacimiento` | `@NotNull` + validación custom `@Past` + edad ≥ 18 |
| `password` (Usuario) | `@NotBlank` (sin restricción de complejidad) |
| `email` | `@NotBlank` + `@Email` |
| `rol` | validado como Enum por Spring |

Para la validación de edad ≥ 18, crear un `@Constraint` personalizado `@MayorDeEdad` que calcula `Period.between(fechaNacimiento, LocalDate.now()).getYears() >= 18`.

---

## 8. Paginación

**Decisión**: Usar `Pageable` de Spring Data en todos los endpoints de listado (`GET /api/pacientes`, `GET /api/usuarios`). El cliente pasa `?page=0&size=20`. El response wrapper usa `Page<T>` serializado directamente (incluye `totalElements`, `totalPages`, `content`).

```java
@GetMapping
public Page<PacienteResponse> listar(
    @RequestParam(required = false) String q,
    Pageable pageable) { ... }
```

---

## 9. Variables de entorno y application.properties

```properties
# Base de datos
spring.datasource.url=jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:saludtotal}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASS}
spring.jpa.hibernate.ddl-auto=create

# JWT
jwt.secret=${JWT_SECRET}
jwt.expiration=86400000

# Scheduled tasks
spring.task.scheduling.enabled=true
```

Variables mínimas requeridas al ejecutar: `DB_USER`, `DB_PASS`, `JWT_SECRET`.
