# Feature Specification: Gestión de Pacientes

**Feature Branch**: `feat/003-gestion-pacientes`

**Created**: 2026-07-01

**Status**: Draft

**Input**: Spec de reemplazo — módulo central de ABM de pacientes para el sistema SaludTotal.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Alta de Paciente (Priority: P1)

Una recepcionista necesita registrar a un nuevo paciente en el sistema al momento de su primera consulta, ingresando sus datos personales básicos.

**Why this priority**: Sin pacientes registrados no existe historial clínico posible. Es el punto de entrada de todos los demás módulos.

**Independent Test**: Se puede probar haciendo POST a `/api/pacientes` con datos válidos y verificando que el paciente queda persistido en la base de datos con un ID autogenerado.

**Acceptance Scenarios**:

1. **Given** la recepcionista completa el formulario con nombre, apellido, DNI, fecha de nacimiento y obra social, **When** confirma el alta, **Then** el sistema crea el paciente, retorna HTTP 201 con el ID asignado y el paciente aparece en la lista general.
2. **Given** la recepcionista intenta registrar un DNI que ya existe en el sistema, **When** envía el formulario, **Then** el sistema retorna HTTP 409 Conflict con el mensaje "Ya existe un paciente con ese DNI".
3. **Given** la recepcionista omite el campo DNI (obligatorio), **When** intenta guardar, **Then** el sistema retorna HTTP 400 Bad Request con detalle del campo faltante.
4. **Given** la recepcionista ingresa una fecha de nacimiento que implica menos de 18 años de edad, **When** intenta guardar, **Then** el sistema retorna HTTP 400 con el mensaje "El paciente debe ser mayor de 18 años".

---

### User Story 2 - Búsqueda y Listado de Pacientes (Priority: P1)

Un médico o recepcionista necesita encontrar rápidamente a un paciente por nombre o DNI antes de una consulta.

**Why this priority**: La búsqueda es la puerta de entrada a toda consulta o registro de evolución. Si es lenta o imprecisa, el flujo clínico se interrumpe.

**Independent Test**: Se puede probar creando 10 pacientes de prueba y verificando que la búsqueda por nombre parcial y por DNI exacto retorna los resultados correctos en menos de 2 segundos.

**Acceptance Scenarios**:

1. **Given** existen pacientes registrados, **When** el usuario busca por nombre parcial (ej: "Gonz"), **Then** el sistema retorna todos los pacientes cuyo nombre o apellido contiene esa cadena, ignorando mayúsculas/minúsculas.
2. **Given** el usuario busca por DNI exacto, **When** el DNI existe, **Then** el sistema retorna exactamente ese paciente. Si no existe, retorna lista vacía (no error).
3. **Given** no se proveen filtros de búsqueda, **When** se lista el endpoint `/api/pacientes`, **Then** el sistema retorna la lista paginada (page/size) ordenada por apellido ascendente.

---

### User Story 3 - Edición de Datos del Paciente (Priority: P2)

La recepcionista necesita actualizar los datos de un paciente cuando cambia su obra social, dirección o teléfono de contacto.

**Why this priority**: Los datos de contacto y cobertura médica cambian con frecuencia. Un sistema sin edición obliga a dar de baja y re-registrar pacientes.

**Independent Test**: Se puede probar haciendo PUT a `/api/pacientes/{id}` con datos modificados y verificando que los cambios se reflejan en una consulta GET posterior.

**Acceptance Scenarios**:

1. **Given** un paciente existe, **When** la recepcionista actualiza su obra social y teléfono, **Then** el sistema guarda los cambios y retorna HTTP 200 con los datos actualizados.
2. **Given** la recepcionista intenta cambiar el DNI a uno ya usado por otro paciente, **When** envía el PUT, **Then** el sistema retorna HTTP 409 Conflict sin modificar ningún dato.
3. **Given** la recepcionista intenta editar un paciente con ID inexistente, **When** envía el PUT, **Then** el sistema retorna HTTP 404 Not Found.

---

### User Story 4 - Baja (Soft-Delete) de Paciente (Priority: P3)

Un administrador necesita desactivar el registro de un paciente duplicado o cargado por error, sin perder el historial de evoluciones asociado.

**Why this priority**: En un prototipo es necesario poder limpiar datos de prueba. El soft-delete preserva la integridad referencial con las evoluciones.

**Independent Test**: Se puede probar haciendo DELETE a `/api/pacientes/{id}` y verificando que el paciente no aparece en listados pero sí es recuperable desde un endpoint de solo admins.

**Acceptance Scenarios**:

1. **Given** un administrador da de baja a un paciente, **When** ejecuta la operación, **Then** el paciente queda marcado como inactivo y desaparece de búsquedas normales, pero sus evoluciones clínicas siguen existiendo en la base de datos.
2. **Given** un paciente está dado de baja, **When** se intenta crear una nueva evolución para ese paciente, **Then** el sistema retorna HTTP 409 indicando que el paciente está inactivo.
3. **Given** el paciente tiene una evolución con bloqueo activo (otro médico la tiene abierta en edición), **When** el administrador intenta dar de baja al paciente, **Then** el sistema retorna HTTP 409 con el mensaje "El paciente tiene una evolución en edición activa por Dr. [nombre]". La baja no se ejecuta.

---

### Edge Cases

- ~~¿Qué ocurre si se intenta dar de baja un paciente que tiene evoluciones con bloqueo activo (en edición)?~~ → **Resuelto**: el sistema retorna HTTP 409 con mensaje descriptivo; la baja no se ejecuta (ver FR-010).
- ~~¿Puede la búsqueda manejar nombres con tildes y caracteres especiales del español (ej: "Núñez" encontrado al buscar "nunez")?~~ → **Resuelto**: sí, mediante collation `utf8mb4_unicode_ci` de MySQL 8 (ver FR-008).
- ~~¿Qué pasa si se envían campos adicionales no esperados en el JSON de alta? (tolerancia a propiedades extra)~~ → **Resuelto**: se ignoran silenciosamente; Jackson configurado con `FAIL_ON_UNKNOWN_PROPERTIES = false`.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE exponer un endpoint `POST /api/pacientes` para crear pacientes con validación de campos obligatorios (DNI, nombre, apellido, fecha de nacimiento). La `fechaNacimiento` DEBE ser una fecha pasada y el paciente DEBE tener al menos 18 años; en caso contrario retorna HTTP 400 con el mensaje "El paciente debe ser mayor de 18 años".
- **FR-002**: El DNI DEBE ser único en el sistema; el intento de duplicar un DNI retorna HTTP 409.
- **FR-003**: El sistema DEBE exponer `GET /api/pacientes` con soporte de paginación (`page`, `size`) y búsqueda por query param (`q`) que filtre por nombre, apellido o DNI.
- **FR-004**: El sistema DEBE exponer `GET /api/pacientes/{id}` para obtener el detalle de un paciente por ID interno.
- **FR-005**: El sistema DEBE exponer `PUT /api/pacientes/{id}` para actualizar datos editables (obra social, teléfono, dirección). El DNI es editable solo si no entra en conflicto con otro paciente existente.
- **FR-006**: El sistema DEBE exponer `DELETE /api/pacientes/{id}` que realiza soft-delete (campo `activo = false`). Solo el rol Administrador puede ejecutar esta operación.
- **FR-007**: Todos los endpoints de pacientes requieren Bearer JWT válido. Matriz de permisos por rol:

  | Operación | MEDICO | RECEPCIONISTA | ADMIN |
  |---|---|---|---|
  | `POST /api/pacientes` | ✅ | ✅ | ✅ |
  | `GET /api/pacientes` | ✅ | ✅ | ✅ |
  | `GET /api/pacientes/{id}` | ✅ | ✅ | ✅ |
  | `PUT /api/pacientes/{id}` | ✅ | ✅ | ✅ |
  | `DELETE /api/pacientes/{id}` | ❌ | ❌ | ✅ |
  | `GET /api/pacientes/inactivos` | ❌ | ❌ | ✅ |
- **FR-008**: La búsqueda por nombre/apellido DEBE ser case-insensitive y tolerante a tildes. Implementación: collation `utf8mb4_unicode_ci` en MySQL 8 (default); la query JPA usa `LIKE LOWER(:q)` sobre columnas con ese collation. No se requiere función `unaccent` ni configuración extra.
- **FR-009**: Los pacientes inactivos (soft-deleted) NO DEBEN aparecer en los resultados de búsqueda ni en el listado general. Un endpoint `GET /api/pacientes/inactivos` (solo Admin) permite consultarlos.

- **FR-010**: Antes de ejecutar el soft-delete de un paciente, el sistema DEBE verificar si existe alguna evolución de ese paciente con bloqueo activo. Si existe, retorna HTTP 409 con el mensaje "El paciente tiene una evolución en edición activa por Dr. [nombre]" y no modifica ningún dato.

- **Paciente**: Atributos: `id` (Long, autogenerado), `nombre` (String, not null), `apellido` (String, not null), `dni` (String, unique, not null), `fechaNacimiento` (LocalDate, not null, **debe ser pasada y con edad ≥ 18 años**), `obraSocial` (String, nullable), `telefono` (String, nullable), `activo` (boolean, default true), `fechaAlta` (LocalDateTime, autogenerada).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El endpoint de búsqueda retorna resultados en menos de 2 segundos para una base de hasta 10.000 pacientes de prueba.
- **SC-002**: El 100% de los intentos de duplicar un DNI retorna HTTP 409 sin crear ningún registro.
- **SC-003**: Un soft-delete nunca elimina físicamente registros de evoluciones clínicas asociadas al paciente.
- **SC-004**: La búsqueda es case-insensitive: "gonzalez", "González" y "GONZALEZ" retornan los mismos resultados.

## Clarifications

### Session 2026-07-01

- Q: El rol MEDICO, ¿qué operaciones puede hacer sobre `/api/pacientes`? → A: Mismos permisos que RECEPCIONISTA: puede crear y editar pacientes, pero no eliminar (soft-delete exclusivo de ADMIN).
- Q: Admin intenta dar de baja a un paciente con evolución en edición (bloqueo activo). ¿Qué hace el sistema? → A: Bloquear la baja con HTTP 409 y mensaje "El paciente tiene una evolución en edición activa por Dr. [nombre]".
- Q: ¿Cómo se implementa la normalización de tildes en la búsqueda? → A: Collation `utf8mb4_unicode_ci` de MySQL 8 (default); insensible a mayúsculas y tildes automáticamente sin código extra.
- Q: Si el cliente envía campos desconocidos en el JSON, ¿el sistema los rechaza o los ignora? → A: Ignorar silenciosamente (Jackson default `FAIL_ON_UNKNOWN_PROPERTIES = false`).
- Q: ¿Qué validaciones aplica el sistema al campo `fechaNacimiento`? → A: Debe ser fecha pasada y el paciente debe tener al menos 18 años.

## Assumptions

- La recepcionista puede crear y editar pacientes; solo el Administrador puede dar de baja.
- El DNI se almacena como String para soportar formatos con puntos, guiones o sin ellos; la validación de formato es básica (solo que no esté vacío).
- La obra social es texto libre en v1; no se valida contra un padrón externo.
- No existe importación masiva de pacientes (CSV/Excel) en v1; el alta es individual.
- El endpoint de listado paginado usa por defecto `page=0`, `size=20`, ordenado por `apellido ASC`.
- La base de datos y todas las tablas de pacientes usan collation `utf8mb4_unicode_ci`; esto lo gestiona Hibernate al crear el esquema (`ddl-auto`) y no requiere scripts SQL manuales.
- Jackson está configurado con `FAIL_ON_UNKNOWN_PROPERTIES = false`; los campos desconocidos en el JSON de entrada se ignoran silenciosamente.
