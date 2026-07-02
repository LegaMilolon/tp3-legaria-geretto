# Feature Specification: Panel de Administración y Gestión de Usuarios

**Feature Branch**: `feat/004-panel-administracion`

**Created**: 2026-07-01

**Status**: Draft

**Input**: Spec de reemplazo — módulo de gestión de usuarios del sistema, roles y acceso administrativo para SaludTotal.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Alta de Usuario del Sistema (Priority: P1)

El administrador del sistema necesita crear cuentas para los médicos, recepcionistas y otros administradores que van a operar el sistema.

**Why this priority**: Sin usuarios el sistema no puede operar. El administrador es el único que puede dar acceso; no existe auto-registro.

**Independent Test**: Se puede probar haciendo POST a `/api/usuarios` con datos válidos y verificando que el usuario queda creado con contraseña hasheada (BCrypt) y el rol asignado.

**Acceptance Scenarios**:

1. **Given** el administrador completa nombre, apellido, email, contraseña temporal y rol, **When** confirma el alta, **Then** el sistema crea el usuario con contraseña hasheada (BCrypt), retorna HTTP 201 con el ID asignado y el campo `password` nunca aparece en la respuesta.
2. **Given** el administrador intenta crear un usuario con un email ya registrado, **When** envía el POST, **Then** el sistema retorna HTTP 409 Conflict con el mensaje "El email ya está en uso".
3. **Given** el administrador no especifica un rol válido (MEDICO, RECEPCIONISTA, ADMIN), **When** envía el POST, **Then** el sistema retorna HTTP 400 Bad Request con detalle del campo inválido.

---

### User Story 2 - Listado y Búsqueda de Usuarios (Priority: P1)

El administrador necesita ver todos los usuarios del sistema, filtrarlos por rol o estado, y encontrar uno específico para editarlo o desactivarlo.

**Why this priority**: La gestión eficiente de usuarios requiere visibilidad completa. Sin listado y búsqueda, el admin trabaja a ciegas.

**Independent Test**: Se puede probar creando usuarios con distintos roles y verificando que el filtro por rol retorna solo los usuarios correctos.

**Acceptance Scenarios**:

1. **Given** existen usuarios registrados, **When** el admin consulta `GET /api/usuarios`, **Then** recibe la lista paginada con nombre, apellido, email, rol y estado (activo/inactivo), sin incluir el campo `password`.
2. **Given** el admin filtra por rol `MEDICO`, **When** consulta `/api/usuarios?rol=MEDICO`, **Then** el sistema retorna solo los usuarios con ese rol.
3. **Given** el admin filtra por estado `activo=false`, **When** consulta `/api/usuarios?activo=false`, **Then** el sistema retorna solo los usuarios desactivados.

---

### User Story 3 - Edición de Rol y Datos de Usuario (Priority: P2)

El administrador necesita cambiar el rol de un usuario (ej: promover a un recepcionista a admin) o actualizar su información personal.

**Why this priority**: Los roles cambian con el tiempo. Un sistema sin edición de roles obliga a crear cuentas nuevas y perder el historial de actividad.

**Independent Test**: Se puede probar haciendo PUT al usuario con un nuevo rol y verificando que el token JWT generado en el siguiente login incluye el nuevo rol en sus claims.

**Acceptance Scenarios**:

1. **Given** el admin edita el rol de un usuario de RECEPCIONISTA a ADMIN, **When** confirma el cambio, **Then** el sistema actualiza el rol y las sesiones activas de ese usuario expiran (requieren re-login para obtener un token con el nuevo rol).
2. **Given** el admin actualiza el nombre o email de un usuario, **When** envía el PUT con datos válidos, **Then** el sistema guarda los cambios y retorna HTTP 200 con los datos actualizados.
3. **Given** el admin intenta cambiar el email a uno ya usado por otro usuario, **When** envía el PUT, **Then** el sistema retorna HTTP 409 Conflict sin modificar ningún dato.

---

### User Story 4 - Activar / Desactivar Cuenta de Usuario (Priority: P1)

Cuando un médico o recepcionista deja de trabajar en la clínica, el administrador debe poder desactivar su cuenta inmediatamente para revocar su acceso.

**Why this priority**: La revocación de acceso es la operación de seguridad más crítica del módulo. Un ex-empleado con acceso activo es el mayor vector de riesgo de filtración.

**Independent Test**: Se puede probar desactivando una cuenta, intentando hacer login con esas credenciales y verificando que el sistema devuelve HTTP 401.

**Acceptance Scenarios**:

1. **Given** el admin desactiva la cuenta de un médico, **When** ese médico intenta hacer login, **Then** el sistema retorna HTTP 401 con el mensaje "Cuenta desactivada".
2. **Given** el admin desactiva la cuenta de un médico con sesión activa, **When** ese médico intenta hacer cualquier request con su token vigente, **Then** el sistema retorna HTTP 401 (el token queda invalidado al verificar el estado de la cuenta en cada request).
3. **Given** una cuenta está desactivada, **When** el admin la reactiva, **Then** el usuario puede volver a hacer login normalmente.
4. **Given** el admin autenticado intenta desactivar su propia cuenta, **When** ejecuta el PATCH con su propio ID, **Then** el sistema retorna HTTP 403 con el mensaje "Un administrador no puede desactivar su propia cuenta".
5. **Given** el admin autenticado intenta eliminar físicamente un usuario que tiene actividad registrada en el log de auditoría, **When** ejecuta el DELETE, **Then** el sistema retorna HTTP 409 con el mensaje "No se puede eliminar un usuario con actividad registrada; use desactivar en su lugar".

---

### User Story 5 - Reseteo de Contraseña por el Administrador (Priority: P2)

El administrador necesita resetear la contraseña de un usuario que la olvidó, asignándole una nueva contraseña temporal.

**Why this priority**: Sin este flujo, un usuario que olvida su contraseña queda bloqueado permanentemente (no hay auto-recuperación por email en v1).

**Independent Test**: Se puede probar haciendo POST al endpoint de reset, luego intentando login con la nueva contraseña temporal y verificando que funciona.

**Acceptance Scenarios**:

1. **Given** un médico olvidó su contraseña, **When** el admin ejecuta el reset asignando una nueva contraseña temporal, **Then** el sistema hashea la nueva contraseña con BCrypt, la persiste y retorna HTTP 200. La contraseña antigua queda inválida inmediatamente.
2. **Given** se reseteó la contraseña, **When** el médico hace login con la nueva contraseña temporal, **Then** el sistema concede acceso normalmente.
3. **Given** el admin intenta resetear la contraseña de una cuenta desactivada, **When** ejecuta el POST al endpoint de reset, **Then** el sistema retorna HTTP 409 con el mensaje "La cuenta está desactivada; reactive la cuenta antes de resetear la contraseña".

---

### Edge Cases

- ~~¿Qué ocurre si el único ADMIN del sistema intenta desactivar su propia cuenta?~~ → **Resuelto**: ningún admin puede desactivar su propia cuenta (HTTP 403); además, si fuera el único ADMIN activo, también aplica HTTP 409 (ver FR-009).
- ~~¿Se puede eliminar físicamente un usuario que nunca hizo ninguna acción en el sistema?~~ → **Resuelto**: sí, el ADMIN puede eliminar físicamente usuarios sin actividad en el log de auditoría; si tienen actividad, solo se permite desactivar (ver FR-012).
- ~~¿Qué pasa si se intenta resetear la contraseña de un usuario desactivado?~~ → **Resuelto**: el sistema retorna HTTP 409; el admin debe reactivar la cuenta primero (ver FR-008).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE exponer `POST /api/usuarios` (solo rol ADMIN) para crear usuarios con: nombre, apellido, **dni (único)**, email (único), contraseña y rol (MEDICO / RECEPCIONISTA / ADMIN).
- **FR-002**: Las contraseñas DEBEN hashearse con BCrypt antes de persistir; el campo `password` NUNCA debe aparecer en ninguna respuesta de la API. Validación de entrada: solo `@NotBlank` (cualquier cadena no vacía es aceptada); no se aplican reglas de complejidad.
- **FR-003**: El sistema DEBE exponer `GET /api/usuarios` (solo ADMIN) con paginación y filtros opcionales por `rol` y `activo`.
- **FR-004**: El sistema DEBE exponer `GET /api/usuarios/{id}` (solo ADMIN) para ver el detalle de un usuario.
- **FR-005**: El sistema DEBE exponer `PUT /api/usuarios/{id}` (solo ADMIN) para actualizar nombre, apellido, email y rol. El email debe seguir siendo único.
- **FR-006**: El sistema DEBE exponer `PATCH /api/usuarios/{id}/estado` (solo ADMIN) para activar o desactivar una cuenta (`{ "activo": false }`).
- **FR-007**: Al desactivar una cuenta, el sistema DEBE invalidar todos los tokens JWT activos de ese usuario verificando el estado en cada request (no solo al emitir el token).
- **FR-008**: El sistema DEBE exponer `POST /api/usuarios/{id}/reset-password` (solo ADMIN) para asignar una nueva contraseña hasheada. Si la cuenta está desactivada, retorna HTTP 409 con el mensaje "La cuenta está desactivada; reactive la cuenta antes de resetear la contraseña".
- **FR-009**: El sistema DEBE impedir dos casos al desactivar una cuenta:
  1. **Único ADMIN activo**: si el usuario a desactivar es el único ADMIN activo en el sistema, retorna HTTP 409 con "No se puede desactivar el único administrador activo del sistema".
  2. **Auto-desactivado**: si el usuario autenticado intenta desactivar su propia cuenta, retorna HTTP 403 con "Un administrador no puede desactivar su propia cuenta".
- **FR-010**: El email del usuario DEBE ser único en el sistema; duplicados retornan HTTP 409. El `dni` del usuario también DEBE ser único; intentar registrar un DNI ya existente retorna HTTP 409.
- **FR-011**: El JWT emitido en login DEBE incluir los claims `dni`, `email` y `rol` según lo establecido en la Constitution. Al cambiar el rol, las sesiones previas expiran (el nuevo rol se refleja solo en tokens emitidos tras el cambio).
- **FR-012**: El sistema DEBE exponer `DELETE /api/usuarios/{id}` (solo ADMIN) para eliminación física de un usuario. Esta operación solo puede ejecutarse si el usuario no tiene registros en el log de auditoría; en caso contrario retorna HTTP 409 con el mensaje "No se puede eliminar un usuario con actividad registrada; use desactivar en su lugar".

### Key Entities

- **Usuario**: Atributos: `id` (Long, autogenerado), `nombre` (String, not null), `apellido` (String, not null), `dni` (String, **unique, not null**), `email` (String, unique, not null), `password` (String hasheada, not null), `rol` (Enum: MEDICO/RECEPCIONISTA/ADMIN, not null), `activo` (boolean, default true), `fechaAlta` (LocalDateTime, autogenerada), `ultimoAcceso` (LocalDateTime, nullable).
- **Rol** (Enum): `MEDICO` — acceso a evoluciones y pacientes; `RECEPCIONISTA` — acceso a pacientes (lectura/escritura), sin acceso a evoluciones clínicas; `ADMIN` — acceso total incluyendo gestión de usuarios.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Una cuenta desactivada es rechazada en el 100% de los intentos de login y en el 100% de los requests con token previo.
- **SC-002**: El campo `password` nunca aparece en ninguna respuesta JSON de la API, incluyendo respuestas de error.
- **SC-003**: El sistema impide en el 100% de los casos dejar al sistema sin ningún ADMIN activo.
- **SC-004**: El cambio de rol se refleja correctamente en el siguiente JWT emitido tras re-login; el token anterior con el rol viejo es rechazado.

## Clarifications

### Session 2026-07-01

- Q: El campo `dni` en la entidad Usuario, ¿es obligatorio para todos los roles? → A: Sí, `dni` es obligatorio y único para todos los roles (MEDICO, RECEPCIONISTA, ADMIN).
- Q: ¿Se puede eliminar físicamente un usuario del sistema? → A: Sí, solo el ADMIN puede hacerlo y únicamente si el usuario no tiene actividad registrada (ningún log de auditoría asociado).
- Q: El admin intenta resetear la contraseña de una cuenta desactivada. ¿Se permite? → A: No; el sistema retorna HTTP 409 indicando que la cuenta está desactivada. El admin debe reactivarla primero.
- Q: ¿Qué requisitos mínimos debe cumplir una contraseña? → A: Sin restricciones; cualquier cadena no vacía es válida. Solo se valida `@NotBlank`.
- Q: ¿Puede un admin desactivar su propia cuenta? → A: No; un admin nunca puede desactivar su propia cuenta aunque haya otros admins activos. Debe pedirle a otro admin.

## Assumptions

- Solo el rol ADMIN puede gestionar usuarios; no existe auto-registro ni recuperación de contraseña por email en v1.
- El `dni` del usuario es obligatorio y único para todos los roles; es uno de los claims del JWT junto con `email` y `rol`.
- Los roles son fijos (MEDICO, RECEPCIONISTA, ADMIN); no hay roles personalizados en v1.
- La invalidación de tokens al desactivar una cuenta se implementa verificando el campo `activo` del usuario en el filtro JWT de Spring Security en cada request (stateless check contra la DB).
- No existe gestión de permisos granulares; los permisos son por rol completo, no por recurso individual.
