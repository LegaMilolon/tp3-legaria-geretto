---
description: "Task list — SaludTotal Backend (Historia Clínica Unificada)"
---

# Tasks: SaludTotal Backend

**Input**: Design documents from `specs/saludtotal/`

**Stack**: Java 21 · Spring Boot · Hibernate/JPA · MySQL 8 · Spring Security + JWT · Lombok · Gradle (Groovy DSL)

**Organization**: Tareas agrupadas por user story para implementación y testeo independiente.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias bloqueantes)
- **[Story]**: User story a la que pertenece la tarea
- Paths base: `src/main/java/com/saludtotal/`

---

## Phase 1: Setup — Estructura del Proyecto

**Objetivo**: Scaffolding inicial del proyecto Spring Boot con Gradle.

- [ ] T001 Inicializar proyecto Spring Boot con Gradle (Groovy DSL) en `build.gradle` — dependencias: web, data-jpa, security, validation, lombok, springdoc-openapi, jjwt, mysql-connector
- [ ] T002 Crear paquete base `com.saludtotal` y clase principal `src/main/java/com/saludtotal/SaludTotalApplication.java` con `@SpringBootApplication` y `@EnableScheduling`
- [ ] T003 Configurar `src/main/resources/application.properties` con variables de entorno `${DB_HOST}`, `${DB_PORT}`, `${DB_NAME}`, `${DB_USER}`, `${DB_PASS}`, `${JWT_SECRET}`, `ddl-auto=create`, collation `utf8mb4_unicode_ci`
- [ ] T004 [P] Crear estructura de paquetes vacíos: `config/`, `controller/`, `service/`, `repository/`, `entity/`, `dto/request/`, `dto/response/`, `enums/`, `exception/`, `security/`

---

## Phase 2: Fundacional — Infraestructura Transversal

**Objetivo**: Componentes bloqueantes que todos los módulos necesitan antes de implementar cualquier user story.

- [ ] T005 Crear enum `src/main/java/com/saludtotal/enums/Rol.java` con valores `MEDICO`, `RECEPCIONISTA`, `ADMIN`
- [ ] T006 [P] Crear enum `src/main/java/com/saludtotal/enums/EstadoEvolucion.java` con valores `BORRADOR`, `CONFIRMADO`, `OCULTO`
- [ ] T007 [P] Crear enum `src/main/java/com/saludtotal/enums/NivelRiesgo.java` con valores `BAJO` (0-39), `MEDIO` (40-69), `ALTO` (70-100) y método estático `fromScore(int score)`
- [ ] T008 Crear `src/main/java/com/saludtotal/exception/ResourceNotFoundException.java` extendiendo `RuntimeException`
- [ ] T009 [P] Crear `src/main/java/com/saludtotal/exception/ConflictException.java` extendiendo `RuntimeException`
- [ ] T010 [P] Crear `src/main/java/com/saludtotal/exception/ForbiddenException.java` extendiendo `RuntimeException`
- [ ] T011 Crear `src/main/java/com/saludtotal/exception/GlobalExceptionHandler.java` con `@RestControllerAdvice` manejando `ResourceNotFoundException` (404), `ConflictException` (409), `ForbiddenException` (403), `MethodArgumentNotValidException` (400 con detalle de campos), `Exception` genérica (500)
- [ ] T012 Crear `src/main/java/com/saludtotal/security/JwtUtil.java` con métodos `generateToken(Usuario)`, `extractEmail(String)`, `extractClaim(String, Function)`, `isTokenValid(String, UserDetails)` usando `io.jsonwebtoken`; claims: `dni`, `email`, `rol`
- [ ] T013 Crear `src/main/java/com/saludtotal/security/UserDetailsServiceImpl.java` implementando `UserDetailsService`; método `loadUserByUsername(email)` busca en `UsuarioRepository`
- [ ] T014 Crear `src/main/java/com/saludtotal/security/JwtFilter.java` extendiendo `OncePerRequestFilter`; extrae token del header `Authorization: Bearer`, valida con `JwtUtil`, verifica `usuario.isActivo()` contra DB en cada request; retorna 401 si cuenta desactivada
- [ ] T015 Crear `src/main/java/com/saludtotal/config/SecurityConfig.java` con `@Configuration` y `@EnableMethodSecurity`; rutas públicas: `POST /api/auth/login`, `/swagger-ui/**`, `/v3/api-docs/**`; resto requiere auth; política `STATELESS`; agrega `JwtFilter` antes de `UsernamePasswordAuthenticationFilter`
- [ ] T016 Crear `src/main/java/com/saludtotal/config/OpenApiConfig.java` configurando Swagger UI con `SecurityScheme` Bearer JWT y `@SecurityRequirement` global

---

## Phase 3: Autenticación y Gestión de Usuarios (US4-P1)

**Objetivo**: Login funcional + alta, listado y desactivación de cuentas. Es el prerequisito de todos los demás módulos porque todos los endpoints requieren JWT.

**Independent Test**: `POST /api/auth/login` con credenciales válidas retorna JWT; con cuenta desactivada retorna 401. `POST /api/usuarios` crea usuario con BCrypt. `PATCH /api/usuarios/{id}/estado` desactiva y token previo es rechazado.

- [ ] T017 [US4] Crear entidad `src/main/java/com/saludtotal/entity/Usuario.java` con campos `id`, `nombre`, `apellido`, `dni` (unique), `email` (unique), `password`, `rol` (Enum), `activo`, `fechaAlta`, `ultimoAcceso`; implementa `UserDetails`; `@JsonIgnore` en `password`; anotaciones Lombok `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`
- [ ] T018 [US4] Crear `src/main/java/com/saludtotal/repository/UsuarioRepository.java` extendiendo `JpaRepository<Usuario, Long>`; métodos: `findByEmail(String)`, `countByRolAndActivoTrue(Rol)`, `existsByEmail(String)`, `existsByDni(String)`
- [ ] T019 [US4] Crear `src/main/java/com/saludtotal/dto/request/LoginRequest.java` con `@NotBlank String email`, `@NotBlank String password`
- [ ] T020 [P] [US4] Crear `src/main/java/com/saludtotal/dto/request/UsuarioRequest.java` con `@NotBlank nombre`, `@NotBlank apellido`, `@NotBlank dni`, `@NotBlank @Email email`, `@NotBlank password`, `@NotNull Rol rol`
- [ ] T021 [P] [US4] Crear `src/main/java/com/saludtotal/dto/response/LoginResponse.java` con `token`, `tipo` ("Bearer"), `rol`, `nombre`, `apellido`
- [ ] T022 [P] [US4] Crear `src/main/java/com/saludtotal/dto/response/UsuarioResponse.java` con todos los campos excepto `password`
- [ ] T023 [US4] Implementar `src/main/java/com/saludtotal/service/AuthService.java` con método `login(LoginRequest)`: verifica credenciales con `BCryptPasswordEncoder`, lanza `UnauthorizedException` si inválidas o cuenta inactiva, genera JWT via `JwtUtil`, actualiza `ultimoAcceso`
- [ ] T024 [US4] Implementar `src/main/java/com/saludtotal/service/UsuarioService.java` con métodos: `crear(UsuarioRequest)` (hashea password, valida unicidad email/dni), `listar(Rol, Boolean, Pageable)`, `obtenerPorId(Long)`, `actualizar(Long, UsuarioRequest)`, `cambiarEstado(Long, boolean, Long idAdmin)` (valida auto-desactivado y único admin), `resetPassword(Long, String)` (valida cuenta activa), `eliminarFisico(Long)` (valida sin actividad en auditoría)
- [ ] T025 [US4] Crear `src/main/java/com/saludtotal/controller/AuthController.java` con `POST /api/auth/login` usando `@Valid`; sin `@PreAuthorize` (ruta pública)
- [ ] T026 [US4] Crear `src/main/java/com/saludtotal/controller/UsuarioController.java` con endpoints: `GET /api/usuarios`, `GET /api/usuarios/{id}`, `POST /api/usuarios`, `PUT /api/usuarios/{id}`, `PATCH /api/usuarios/{id}/estado`, `POST /api/usuarios/{id}/reset-password`, `DELETE /api/usuarios/{id}`; todos con `@PreAuthorize("hasRole('ADMIN'")`
- [ ] T027 [US4] Crear `src/main/java/com/saludtotal/config/DataInitializer.java` implementando `CommandLineRunner`; crea 3 usuarios semilla (ADMIN, MEDICO, RECEPCIONISTA) con contraseñas hasheadas con BCrypt solo si la tabla está vacía
- [ ] T028 [US4] Escribir test unitario `src/test/java/com/saludtotal/service/AuthServiceTest.java` cubriendo: login exitoso, credenciales inválidas, cuenta desactivada
- [ ] T029 [P] [US4] Escribir test unitario `src/test/java/com/saludtotal/service/UsuarioServiceTest.java` cubriendo: crear usuario, email duplicado, DNI duplicado, auto-desactivado (403), único admin (409), reset en cuenta inactiva (409)

---

## Phase 4: Gestión de Pacientes P1 (US3-1, US3-2)

**Objetivo**: Alta y búsqueda de pacientes. Prerequisito de historia clínica y biométrico.

**Independent Test**: `POST /api/pacientes` con DNI válido y edad ≥ 18 → HTTP 201. Búsqueda `GET /api/pacientes?q=gonz` retorna pacientes con apellido "González" case-insensitive.

- [ ] T030 [US3] Crear entidad `src/main/java/com/saludtotal/entity/Paciente.java` con campos `id`, `nombre`, `apellido`, `dni` (unique), `fechaNacimiento`, `obraSocial`, `telefono`, `activo`, `monitoreoActivo`, `fechaAlta`; anotación `@SQLRestriction("activo = true")` de Hibernate 6
- [ ] T031 [US3] Crear validador custom `src/main/java/com/saludtotal/validation/MayorDeEdad.java` (anotación) + `MayorDeEdadValidator.java` implementando `ConstraintValidator`; verifica `Period.between(fecha, LocalDate.now()).getYears() >= 18`
- [ ] T032 [US3] Crear `src/main/java/com/saludtotal/dto/request/PacienteRequest.java` con `@NotBlank nombre`, `@NotBlank apellido`, `@NotBlank dni`, `@NotNull @Past @MayorDeEdad fechaNacimiento`, `obraSocial`, `telefono`
- [ ] T033 [P] [US3] Crear `src/main/java/com/saludtotal/dto/response/PacienteResponse.java` con todos los campos excepto `activo` y `monitoreoActivo` (internos)
- [ ] T034 [US3] Crear `src/main/java/com/saludtotal/repository/PacienteRepository.java` extendiendo `JpaRepository<Paciente, Long>`; métodos: `findByDni(String)`, `existsByDni(String)`, query JPQL `findByNombreContainingIgnoreCaseOrApellidoContainingIgnoreCaseOrDniContaining(String, String, String, Pageable)`, `findByMonitoreoActivo(boolean)`
- [ ] T035 [US3] Implementar `src/main/java/com/saludtotal/service/PacienteService.java` con métodos: `crear(PacienteRequest)` (valida DNI único), `buscar(String q, Pageable)`, `obtenerPorId(Long)`, `actualizar(Long, PacienteRequest)`, `darDeBaja(Long, Long idAdmin)` — fase P1 sin lógica de baja aún
- [ ] T036 [US3] Crear `src/main/java/com/saludtotal/controller/PacienteController.java` con `GET /api/pacientes`, `GET /api/pacientes/{id}`, `POST /api/pacientes`; roles: todos los roles autenticados
- [ ] T037 [US3] Escribir test unitario `src/test/java/com/saludtotal/service/PacienteServiceTest.java` cubriendo: crear paciente válido, DNI duplicado (409), menor de edad (400), búsqueda case-insensitive

---

## Phase 5: Historia Clínica — Registro de Evolución (US1-1, US1-4)

**Objetivo**: Médico crea y confirma evoluciones. Bloqueo pesimista. Firma implícita del médico.

**Independent Test**: `POST /api/evoluciones` crea evolución BORRADOR con médico autenticado. `PUT /api/evoluciones/{id}/confirmar` cambia a CONFIRMADO e impide edición posterior. Segundo médico que intenta bloquear una evolución ya bloqueada recibe 409.

- [ ] T038 [US1] Crear entidad `src/main/java/com/saludtotal/entity/EvolucionClinica.java` con campos: `id`, `paciente` (ManyToOne), `medico` (ManyToOne a Usuario), `timestamp`, `contenido` (TEXT), `estado` (EstadoEvolucion), `bloqueadoPor` (ManyToOne nullable), `timestampBloqueo`, `evolucionOriginal` (ManyToOne nullable, self-reference)
- [ ] T039 [US1] Crear `src/main/java/com/saludtotal/repository/EvolucionRepository.java` con métodos: `findByPacienteIdAndEstadoNotOrderByTimestampDesc(Long, EstadoEvolucion, Pageable)`, `findByBloqueadoPorNotNullAndTimestampBloqueoLessThan(LocalDateTime)` (para job de limpieza), `existsByPacienteIdAndBloqueadoPorNotNull(Long)` (para validar baja de paciente)
- [ ] T040 [P] [US1] Crear `src/main/java/com/saludtotal/dto/request/EvolucionRequest.java` con `@NotNull Long pacienteId`, `@NotBlank String contenido`
- [ ] T041 [P] [US1] Crear `src/main/java/com/saludtotal/dto/response/EvolucionResponse.java` con todos los campos incluyendo datos del médico firmante
- [ ] T042 [US1] Implementar `src/main/java/com/saludtotal/service/EvolucionService.java` con métodos: `crear(EvolucionRequest, Long medicoId)` — verifica paciente activo, crea en BORRADOR; `confirmar(Long id, Long medicoId)` — verifica ownership, cambia a CONFIRMADO; `bloquear(Long id, Long medicoId)` — UPDATE atómico via `@Modifying`; `liberar(Long id, Long medicoId)`; `agregarCorreccion(Long idOriginal, String contenido, Long medicoId)`; `ocultar(Long id, String motivo)` (solo admin)
- [ ] T043 [US1] Crear job `src/main/java/com/saludtotal/service/EvolucionLockCleanupJob.java` con `@Scheduled(fixedRate = 300000)` (cada 5 min) que libera bloqueos con `timestampBloqueo < NOW() - 30 min`
- [ ] T044 [US1] Crear `src/main/java/com/saludtotal/controller/EvolucionController.java` con endpoints: `GET /api/evoluciones/paciente/{pacienteId}`, `POST /api/evoluciones`, `PUT /api/evoluciones/{id}/confirmar`, `PUT /api/evoluciones/{id}/bloquear`, `PUT /api/evoluciones/{id}/liberar`, `POST /api/evoluciones/{id}/correccion`, `PATCH /api/evoluciones/{id}/ocultar`; roles: MEDICO y ADMIN
- [ ] T045 [US1] Completar `PacienteService.darDeBaja(Long, Long)` con verificación de bloqueo activo vía `EvolucionRepository.existsByPacienteIdAndBloqueadoPorNotNull()`; lanza `ConflictException` con mensaje descriptivo
- [ ] T046 [US1] Añadir `PUT /api/pacientes/{id}` y `DELETE /api/pacientes/{id}` y `GET /api/pacientes/inactivos` a `PacienteController.java`; soft-delete solo `hasRole('ADMIN')`
- [ ] T047 [US1] Escribir test unitario `src/test/java/com/saludtotal/service/EvolucionServiceTest.java` cubriendo: crear evolución, confirmar, bloqueo exitoso, bloqueo rechazado (409), corrección de evolución confirmada

---

## Phase 6: Auditoría y Seguridad (US2-1, US2-2)

**Objetivo**: Log de auditoría inmutable, detección de acceso anómalo, RBAC verificado.

**Independent Test**: Después de consultar historial de un paciente, existe un `EventoAuditoria` con el ID de médico y paciente correctos. Consultar 51 historiales distintos en < 10 min dispara alerta en el log.

- [ ] T048 [US2] Crear entidad `src/main/java/com/saludtotal/entity/EventoAuditoria.java` con campos `id`, `tipoEvento`, `usuario` (ManyToOne), `paciente` (ManyToOne nullable), `timestamp`, `dispositivo`, `resultado`
- [ ] T049 [P] [US2] Crear `src/main/java/com/saludtotal/repository/EventoAuditoriaRepository.java` con métodos: `countByUsuarioIdAndTimestampGreaterThan(Long, LocalDateTime)` (detección de anomalía), `existsByUsuarioId(Long)` (validar eliminación física de usuario)
- [ ] T050 [US2] Crear `src/main/java/com/saludtotal/service/AuditoriaService.java` con método `registrar(String tipoEvento, Long usuarioId, Long pacienteId, String resultado)` y `verificarAnomalia(Long usuarioId)` — si accesos > 50 en < 10 min, loguear alerta con `log.warn()` (simulación en prototipo)
- [ ] T051 [US2] Inyectar `AuditoriaService` en `EvolucionService` y `PacienteService`; llamar `registrar()` en cada operación de consulta o modificación de datos clínicos
- [ ] T052 [US2] Actualizar `UsuarioService.eliminarFisico()` para verificar `EventoAuditoriaRepository.existsByUsuarioId(id)` antes de permitir eliminación

---

## Phase 7: P2 — Edición de Pacientes y Usuarios (US3-3, US4-3, US4-5)

**Objetivo**: Completar operaciones de edición y administración de usuarios.

**Independent Test**: `PUT /api/pacientes/{id}` actualiza obra social; `PUT /api/usuarios/{id}` cambia el rol y el token viejo es rechazado en el siguiente request.

- [ ] T053 [P] [US3] Completar `PacienteService.actualizar(Long, PacienteRequest)` con validación de DNI único al editar; retorna `PacienteResponse` actualizado
- [ ] T054 [P] [US4] Completar `UsuarioService.actualizar(Long, UsuarioRequest)` con validación de email/DNI únicos al editar; al cambiar rol el JwtFilter invalida la sesión en el próximo request automáticamente (verifica activo y consistencia de claims)

---

## Phase 8: P2 — Historial Clínico Completo y Desactivación (US1-2, US4-4)

**Objetivo**: Historial unificado paginado desde cualquier sede + desactivación de cuentas con invalidación inmediata.

**Independent Test**: `GET /api/evoluciones/paciente/{id}?page=0&size=20` retorna evoluciones de todos los médicos ordenadas DESC. Desactivar cuenta → request siguiente con token activo devuelve 401.

- [ ] T055 [US1] Agregar a `EvolucionRepository` query JPQL para historial paginado ordenado por `timestamp DESC` excluyendo estado OCULTO
- [ ] T056 [P] [US4] Verificar que `JwtFilter` consulta `usuarioRepository.findByEmail()` en cada request para validar `activo`; este comportamiento ya debería estar en T014 — marcar como completado si es así, o ajustar

---

## Phase 9: P3 — Adjuntos y Soft-Delete (US1-3, US3-4)

**Objetivo**: Adjuntar archivos a evoluciones + baja de pacientes con validaciones completas.

**Independent Test**: `POST /api/evoluciones/{id}/adjuntos` recibe un archivo, lo almacena y retorna hash SHA-256. `DELETE /api/pacientes/{id}` con evolución en edición activa retorna 409.

- [ ] T057 [US1] Crear entidad `src/main/java/com/saludtotal/entity/AdjuntoClinico.java` con campos `id`, `evolucion` (ManyToOne), `tipo`, `hashIntegridad`, `urlAlmacenamiento`
- [ ] T058 [P] [US1] Crear `src/main/java/com/saludtotal/repository/AdjuntoRepository.java` con `findByEvolucionId(Long)`
- [ ] T059 [US1] Implementar lógica de adjuntos en `EvolucionService`: `agregarAdjunto(Long evolucionId, MultipartFile file)` — calcula SHA-256, guarda en disco local (`/uploads/adjuntos/`), strip de metadatos EXIF para imágenes usando `metadata-extractor` o guardado como blob sin EXIF
- [ ] T060 [US1] Agregar endpoint `POST /api/evoluciones/{id}/adjuntos` en `EvolucionController.java` aceptando `multipart/form-data`

---

## Phase 10: Módulo Biométrico (US5-1, US5-2, US5-3, US5-4)

**Objetivo**: Recolección de métricas de tipeo, score simulado diario, alertas y panel médico.

**Independent Test**: `POST /api/biometrico/sesion/inicio` crea sesión. Batch `ScoreBatchService` a las 00:00 genera un `ScoreRiesgo` por paciente con monitoreo activo. Si score ≥ 70, `AlertaBiometrica` es creada. `GET /api/biometrico/historial/{pacienteId}` retorna scores de los últimos 30 días.

- [ ] T061 [US5] Crear entidad `src/main/java/com/saludtotal/entity/SesionBiometrica.java` con `id`, `paciente` (ManyToOne), `timestampInicio`, `timestampFin`, `cantidadEventos`, `dispositivo`
- [ ] T062 [P] [US5] Crear entidad `src/main/java/com/saludtotal/entity/EventoTipeo.java` con `id`, `sesion` (ManyToOne), `timestamp`, `tipoEvento`, `duracionPulsacionMs`, `intervaloDesdeAnteriorMs`; **sin campo de carácter presionado**
- [ ] T063 [P] [US5] Crear entidad `src/main/java/com/saludtotal/entity/ScoreRiesgo.java` con `id`, `paciente` (ManyToOne), `timestampEvaluacion`, `score`, `nivel` (NivelRiesgo), `alertaGenerada`
- [ ] T064 [P] [US5] Crear entidad `src/main/java/com/saludtotal/entity/AlertaBiometrica.java` con `id`, `paciente` (ManyToOne), `score` (ManyToOne a ScoreRiesgo), `timestamp`, `canalNotificacion`, `estado`, `feedbackMedico`
- [ ] T065 [US5] Crear repositories: `SesionBiometricaRepository`, `EventoTipeoRepository`, `ScoreRiesgoRepository` con `findByPacienteIdAndTimestampEvaluacionBetween()`, `AlertaBiometricaRepository` con `findByPacienteId(Long, Pageable)`
- [ ] T066 [US5] Crear `src/main/java/com/saludtotal/dto/request/EventoTipeoRequest.java` (timestamp, tipoEvento, duracion, intervalo) y `src/main/java/com/saludtotal/dto/request/LoteEventosRequest.java` con `List<EventoTipeoRequest> eventos`
- [ ] T067 [US5] Implementar `src/main/java/com/saludtotal/service/BiometricoService.java` con métodos: `iniciarSesion(Long pacienteId, String dispositivo)`, `registrarEventos(Long sesionId, LoteEventosRequest)`, `finalizarSesion(Long sesionId)`, `actualizarConfiguracion(Long pacienteId, boolean activo)`, `obtenerHistorial(Long pacienteId)`, `registrarFeedback(Long alertaId, String feedback)`
- [ ] T068 [US5] Implementar `src/main/java/com/saludtotal/service/ScoreBatchService.java` con `@Scheduled(cron = "0 0 0 * * *")`; por cada paciente con `monitoreoActivo = true`: genera score aleatorio (0-100), determina nivel via `NivelRiesgo.fromScore()`, persiste `ScoreRiesgo`; si score ≥ 70 crea `AlertaBiometrica` y loguea `log.info("[PUSH] Alerta biométrica paciente ID={}", p.getId())`
- [ ] T069 [US5] Crear `src/main/java/com/saludtotal/controller/BiometricoController.java` con endpoints: `POST /api/biometrico/sesion/inicio`, `POST /api/biometrico/sesion/{id}/eventos`, `POST /api/biometrico/sesion/{id}/fin`, `PATCH /api/biometrico/configuracion`, `GET /api/biometrico/historial/{pacienteId}`, `POST /api/biometrico/alerta/{id}/feedback`

---

## Phase 11: Polish y Preocupaciones Transversales

**Objetivo**: Documentación Swagger, tests de integración de auth, refinamientos finales.

- [ ] T070 Agregar anotaciones `@Operation`, `@ApiResponse` y `@Tag` a todos los controllers para documentación Swagger completa
- [ ] T071 [P] Crear `src/test/java/com/saludtotal/integration/AuthIntegrationTest.java` con `@SpringBootTest`; prueba flujo completo: login → JWT → request protegido → desactivar cuenta → token rechazado
- [ ] T072 [P] Crear `src/test/java/com/saludtotal/integration/PacienteIntegrationTest.java`; prueba: crear paciente → buscar por DNI → editar → soft-delete → verificar que no aparece en listado
- [ ] T073 Revisar y ajustar `application.properties` para agregar perfil de test (`spring.profiles.active=test`) con `ddl-auto=create-drop` para H2 in-memory en tests de integración, evitando dependencia de MySQL real en CI
- [ ] T074 [P] Verificar que el campo `password` no aparece en ninguna respuesta JSON; revisar todos los `UsuarioResponse.java` y serialización de `Usuario.java`
- [ ] T075 Agregar `.gitignore` con exclusión de `.env`, `build/`, `.gradle/`, `*.jar`, archivos de IDE

---

## Dependencies: Orden de Implementación

```
Phase 1 (Setup)
    → Phase 2 (Fundacional: enums, exceptions, JWT, Security)
        → Phase 3 (Auth + Usuarios) ← todos los demás módulos dependen de este
            → Phase 4 (Pacientes P1)   → Phase 5 (Historia Clínica)
            → Phase 4 (Pacientes P1)   → Phase 10 (Biométrico)
            → Phase 6 (Auditoría)      ← depende de Phase 4 y Phase 5
            → Phase 7 (P2 ediciones)   ← depende de Phase 4 y Phase 3
            → Phase 9 (P3 adjuntos)    ← depende de Phase 5
```

**Stories independientes entre sí** (pueden implementarse en paralelo una vez completado Phase 3):
- Historia Clínica (Phases 5, 8, 9) ↔ Biométrico (Phase 10)

---

## Parallel Execution Examples

**Dentro de Phase 2** (todos son archivos distintos):
> T005, T006, T007 pueden hacerse simultáneamente (enums)
> T008, T009, T010 pueden hacerse simultáneamente (exceptions)

**Dentro de Phase 3**:
> T020, T021, T022 pueden hacerse simultáneamente (DTOs de usuario)

**Dentro de Phase 10**:
> T061, T062, T063, T064 pueden hacerse simultáneamente (entidades biométricas)

---

## Implementation Strategy (MVP First)

| Scope | Phases | Entrega |
|---|---|---|
| **MVP mínimo** (auth + pacientes) | 1, 2, 3, 4 | Sistema con login, CRUD de usuarios y pacientes |
| **MVP clínico** (+ historia clínica) | + 5, 6 | Registro de evoluciones con auditoría |
| **MVP completo** | + 7, 8, 9 | Todas las features P1 y P2 |
| **Full system** | + 10, 11 | Módulo biométrico + documentación Swagger |

**Total de tareas**: 75  
**Tareas paralelas [P]**: 26  
**Distribución por módulo**: Setup/Fundacional: 16 · Auth/Usuarios: 13 · Pacientes: 8 · Historia Clínica: 10 · Auditoría: 5 · Biométrico: 9 · Polish: 6  
**Scope MVP sugerido**: Phases 1-4 (T001–T037) — 37 tareas, sistema funcional con auth y CRUD de pacientes
