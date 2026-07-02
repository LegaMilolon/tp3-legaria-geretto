# Feature Specification: Seguridad y Confidencialidad de Datos Clínicos

**Feature Branch**: `feat/002-seguridad-confidencialidad`

**Created**: 2026-07-01

**Status**: Draft

**Input**: User description: "Lo más importante es la confidencialidad: si se filtra un dato de un paciente famoso, nos demandan y cerramos."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Control de Acceso por Rol (Priority: P1)

Un administrador necesita garantizar que solo el personal médico y administrativo autorizado pueda acceder a los datos clínicos de los pacientes, y que cada rol vea únicamente lo que necesita.

**Why this priority**: Es la base de toda la seguridad del sistema. Sin control de acceso robusto, ninguna otra medida de seguridad es efectiva. La filtración de datos de pacientes es el riesgo existencial número uno para el negocio.

**Independent Test**: Se puede probar creando tres cuentas con roles distintos (médico, recepcionista, administrador) e intentando acceder a funcionalidades restringidas para cada rol. Todas las restricciones deben aplicarse correctamente.

**Acceptance Scenarios**:

1. **Given** un usuario con rol "Recepcionista" está autenticado, **When** intenta acceder a la pantalla de evoluciones clínicas de cualquier paciente, **Then** el sistema deniega el acceso y muestra un mensaje de error sin revelar información del paciente.
2. **Given** un médico está autenticado, **When** intenta ver los datos financieros/facturación de la clínica, **Then** el sistema deniega el acceso.
3. **Given** un administrador desactiva la cuenta de un médico que dejó de trabajar en la clínica, **When** ese médico intenta iniciar sesión, **Then** el acceso es denegado inmediatamente, incluyendo sesiones activas existentes.

---

### User Story 2 - Auditoría Completa de Accesos (Priority: P1)

El director de la red necesita poder saber en cualquier momento quién accedió a la historia clínica de un paciente específico, cuándo lo hizo y desde qué dispositivo.

**Why this priority**: En caso de filtraciones (especialmente de pacientes públicos/famosos), la auditoría es la herramienta legal y técnica para determinar responsabilidades. También sirve como disuasivo.

**Independent Test**: Se puede probar accediendo a la historia de un paciente de prueba y luego consultando el log de auditoría para verificar que el acceso quedó registrado con todos los metadatos correctos.

**Acceptance Scenarios**:

1. **Given** un médico abrió el historial de un paciente, **When** el administrador consulta el log de auditoría para ese paciente, **Then** ve el registro con: ID del médico, nombre, timestamp exacto, sede y tipo de acción realizada.
2. **Given** existe un log de auditoría, **When** cualquier usuario (incluyendo administradores) intenta modificar o eliminar una entrada del log, **Then** el sistema rechaza la operación y registra el intento.
3. **Given** se detecta que una cuenta accedió a más de 50 historiales en menos de 10 minutos, **When** ocurre este patrón, **Then** el sistema envía una alerta automática al administrador de seguridad.

---

### User Story 3 - Cifrado de Datos en Reposo y en Tránsito (Priority: P1)

Los datos clínicos deben estar cifrados tanto cuando se almacenan en servidores como cuando viajan por la red, de modo que una intercepción o robo de dispositivo no exponga información sensible.

**Why this priority**: Mitigación de riesgo de filtraciones por acceso físico a servidores/dispositivos o interceptación de red.

**Independent Test**: Se puede probar inspeccionando el tráfico de red durante una operación de guardado y verificando que los datos son ilegibles sin la clave. También accediendo directamente a la base de datos para confirmar que los campos sensibles están cifrados.

**Acceptance Scenarios**:

1. **Given** una evolución clínica se transmite desde la tablet al servidor, **When** se captura el tráfico de red, **Then** el contenido es ilegible (cifrado TLS 1.3 o superior) y no contiene datos de pacientes en texto plano.
2. **Given** un administrador accede directamente a la base de datos sin pasar por la API, **When** consulta la tabla de evoluciones, **Then** los campos de contenido clínico están cifrados con AES-256 y son ilegibles sin la clave de descifrado.
3. **Given** una tablet es robada con sesión activa, **When** el administrador remarca el dispositivo como comprometido, **Then** la sesión en ese dispositivo es invalidada en menos de 60 segundos.

---

### User Story 4 - Autenticación de Médicos (Single-Factor) (Priority: P2)

Los médicos se autentican con usuario y contraseña. El sistema protege contra accesos no autorizados mediante bloqueo por intentos fallidos y expiración de sesión.

**Why this priority**: Autenticación robusta con single-factor es el baseline mínimo de seguridad; MFA descartado en v1 por decisión del cliente.

**Independent Test**: Se puede probar con credenciales correctas verificando acceso concedido, y con 3 intentos fallidos verificando el bloqueo de cuenta.

**Acceptance Scenarios**:

1. **Given** un médico ingresa usuario y contraseña correctos, **When** el sistema valida las credenciales, **Then** concede acceso y crea una sesión JWT con claims `dni`, `email` y `rol`.
2. **Given** un médico ingresa una contraseña incorrecta tres veces consecutivas, **When** ocurre el tercer intento fallido, **Then** la cuenta se bloquea temporalmente por 15 minutos y se notifica al administrador.
3. **Given** un médico ya autenticado no realiza ninguna acción por 30 minutos, **When** se cumple el timeout, **Then** la sesión expira y se requiere re-autenticación completa.

---

### Edge Cases

- ~~¿Qué ocurre si el segundo factor (SMS/app) no está disponible durante una emergencia médica?~~ → **Resuelta**: MFA descartado en v1; autenticación single-factor (ver FR-002).
- ~~¿Cómo se manejan los accesos durante auditorías legales o judiciales donde se requiere presentar registros?~~ → **Resuelta**: fuera de alcance; el sistema no contempla este escenario.
- ¿Qué sucede si el administrador que maneja las claves de cifrado se desvincula de la organización?
- ~~¿Cómo se garantiza la seguridad en dispositivos compartidos (una tablet usada por múltiples médicos en turnos)?~~ → **Resuelta**: cada médico tiene su propia tablet asignada; las tablets no se comparten (ver Assumptions).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE implementar control de acceso basado en roles (RBAC) con al menos los roles: Médico, Recepcionista, Administrador y Auditor.
- **FR-002**: El sistema DEBE autenticar usuarios mediante usuario y contraseña (single-factor). No se implementa MFA en v1. ⚠️ Tradeoff aceptado: el riesgo por robo de credenciales se mitiga con bloqueo tras 3 intentos fallidos (FR-008), expiración de sesión (FR-009) y revocación remota (FR-006).
- **FR-003**: El sistema DEBE cifrar todos los datos clínicos en tránsito usando TLS 1.3 o superior.
- **FR-004**: El sistema DEBE cifrar los datos clínicos sensibles en reposo usando AES-256 o equivalente.
- **FR-005**: El sistema DEBE mantener un log de auditoría inmutable para cada acceso a datos de pacientes (lectura, escritura, exportación).
- **FR-006**: El sistema DEBE permitir al administrador revocar sesiones activas de forma remota e inmediata.
- **FR-007**: El sistema DEBE detectar y alertar sobre patrones de acceso anómalos. El umbral es **fijo**: si una cuenta consulta más de **50 historiales distintos en menos de 10 minutos**, el sistema envía una alerta automática al administrador de seguridad. Este valor no es configurable.
- **FR-008**: El sistema DEBE bloquear una cuenta tras 3 intentos de autenticación fallidos consecutivos.
- **FR-009**: Las sesiones DEBEN expirar automáticamente tras un período de inactividad configurable (default: 30 minutos).
- **FR-010**: ~~El sistema DEBE cumplir con la Ley 25.326 y normativas del Ministerio de Salud.~~ **Descartado**: el sistema es un prototipo educativo que no entra en producción; no aplican requisitos de cumplimiento normativo (HIPAA, Ley 25.326, etc.).
- **FR-011**: El sistema DEBE notificar al administrador de seguridad dentro de las 72 horas si se detecta una posible brecha de datos.

### Key Entities

- **Usuario**: Persona con acceso al sistema. Atributos: ID, nombre, rol, estado (activo/inactivo), dispositivos registrados, último acceso.
- **Sesión**: Instancia de acceso autenticado. Atributos: ID, ID usuario, dispositivo, IP, timestamp inicio/fin, estado.
- **EventoAuditoría**: Registro inmutable de una acción. Atributos: ID, tipo de evento, ID usuario, ID paciente afectado, timestamp, sede, dispositivo, resultado (éxito/fallo).
- **AlertaSeguridad**: Notificación por comportamiento anómalo. Atributos: ID, tipo de alerta, timestamp, ID usuario involucrado, estado (pendiente/revisada).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los accesos a datos clínicos queda registrado en el log de auditoría sin excepciones.
- **SC-002**: Una sesión comprometida puede ser revocada remotamente en menos de 60 segundos.
- **SC-003**: Las alertas por acceso anómalo se generan en tiempo real (latencia inferior a 5 segundos desde el evento).
- **SC-004**: Ningún campo de contenido clínico es legible en texto plano en la base de datos o en tráfico de red interceptado.
- **SC-005**: El sistema supera un test de penetración externo sin vulnerabilidades críticas ni altas (según clasificación CVSS).

## Clarifications

### Session 2026-07-01

- Q: ¿Qué método de segundo factor debe usar el sistema para la autenticación de médicos? → A: Sin doble autenticación; se usa solo usuario y contraseña (single-factor). ⚠️ Tradeoff aceptado: aumenta el riesgo ante robo de credenciales; se compensa con bloqueo por intentos fallidos y revocación remota de sesiones.
- Q: ¿Qué ocurre si el segundo factor no está disponible durante una emergencia médica? → A: Resuelta automáticamente; MFA descartado en Q1.
- Q: ¿Se requiere cumplimiento HIPAA o Ley 25.326? → A: No. El sistema es un prototipo educativo que no va a salir al aire; no aplican requisitos legales de cumplimiento normativo.
- Q: Varios médicos comparten la misma tablet en distintos turnos. ¿Cómo debe manejarse la sesión al cambiar de médico? → A: Cada médico tiene su propia tablet asignada; las tablets no se comparten entre médicos.
- Q: El umbral de detección de anomalías, ¿debe ser configurable o fijo? → A: Fijo en el código: 50 historiales consultados en 10 minutos por la misma cuenta.
- Q: ¿Cómo debe actuar el sistema cuando una autoridad judicial solicita acceso a los registros de auditoría? → A: Fuera de alcance; el sistema no contempla accesos judiciales.

## Assumptions

- La gestión de claves de cifrado se realiza mediante un servicio dedicado de gestión de claves (KMS), no almacenando claves junto a los datos.
- Los dispositivos (tablets) están enrolados en un sistema de gestión de dispositivos móviles (MDM) que permite borrado remoto.
- **El sistema es un prototipo educativo que no entra en producción**; no aplican requisitos legales ni regulatorios (Ley 25.326, HIPAA, normativas del Ministerio de Salud). Las decisiones de seguridad se toman por buenas prácticas, no por obligaciones legales.
- Cada médico tiene su propia tablet asignada; las tablets no se comparten entre médicos ni entre turnos. El escenario de dispositivos compartidos está fuera de alcance.
- El sistema de alertas notifica al administrador de seguridad por email y notificación push; no incluye integración con SIEM externo en v1.
- Se asume que el equipo de infraestructura gestiona los certificados TLS y su renovación; el sistema consume certificados válidos.
- **Fuera de alcance**: accesos judiciales o legales a registros de auditoría. No se implementa ningún flujo ni rol especial para este escenario.
