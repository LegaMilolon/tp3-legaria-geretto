# Feature Specification: Historia Clínica Digital en Tablet

**Feature Branch**: `feat/001-historia-clinica-digital`

**Created**: 2026-07-01

**Status**: Draft

**Input**: User description: "El médico tiene que poder escribir la evolución del paciente en la tablet."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro de Evolución del Paciente (Priority: P1)

Un médico recibe a un paciente durante una consulta y necesita registrar la evolución clínica del día directamente desde su tablet, sin interrumpir el flujo de la consulta.

**Why this priority**: Es el núcleo del sistema. Sin esta funcionalidad, el resto de los módulos no tiene sentido. Es la razón principal por la que el cliente solicita el sistema.

**Independent Test**: Se puede probar completamente creando un paciente de prueba, iniciando sesión como médico y completando un registro de evolución. El resultado debe ser persistente y consultable inmediatamente.

**Acceptance Scenarios**:

1. **Given** el médico está autenticado y tiene un paciente seleccionado, **When** abre la pantalla de evolución y escribe el texto de la consulta, **Then** el sistema guarda el registro con fecha, hora y firma digital del médico.
2. **Given** el médico guardó una evolución, **When** otro médico autorizado consulta el historial del mismo paciente, **Then** puede ver todas las evoluciones anteriores ordenadas cronológicamente.
3. **Given** el médico inicia una evolución pero no la termina, **When** cierra accidentalmente la app, **Then** el borrador es recuperado automáticamente al reabrir.
4. **Given** el Médico A tiene abierta (en edición) la evolución de un paciente, **When** el Médico B intenta abrir la misma evolución, **Then** el sistema deniega el acceso de edición al Médico B y muestra el mensaje: "En uso por Dr. [nombre]". El Médico B puede ver la evolución en modo solo lectura.

---

### User Story 2 - Consulta del Historial Clínico Completo (Priority: P2)

Un médico necesita revisar el historial completo de un paciente antes de una consulta, incluyendo evoluciones anteriores registradas por otros médicos de la red de clínicas.

**Why this priority**: El valor del sistema "unificado" reside en que un médico de cualquier sede de la red pueda ver el historial completo, no solo sus propias notas.

**Independent Test**: Se puede probar creando evoluciones desde dos cuentas de médico distintas y verificando que ambas son visibles desde una tercera cuenta.

**Acceptance Scenarios**:

1. **Given** un paciente fue atendido en dos clínicas distintas de la red, **When** un médico busca al paciente por DNI o nombre, **Then** el sistema muestra el historial unificado de ambas sedes.
2. **Given** el médico está viendo el historial, **When** aplica filtros por fecha o especialidad, **Then** la lista se actualiza en tiempo real sin recargar la pantalla completa.
3. **Given** el historial tiene más de 100 evoluciones, **When** el médico carga la pantalla, **Then** el sistema pagina los resultados y carga el resto de forma incremental.

---

### User Story 3 - Adjuntar Archivos a la Evolución (Priority: P3)

El médico necesita adjuntar imágenes (radiografías, fotos de lesiones) o resultados de laboratorio al registro de evolución del paciente.

**Why this priority**: Complementa el registro textual con evidencia visual/diagnóstica. Es importante pero no bloquea el MVP.

**Independent Test**: Se puede probar subiendo una imagen desde la galería de la tablet y verificando que queda asociada a la evolución específica.

**Acceptance Scenarios**:

1. **Given** el médico está editando una evolución, **When** toca el botón "Adjuntar" y selecciona una imagen de la galería, **Then** la imagen queda vinculada a esa evolución y se muestra en miniatura dentro del registro.
2. **Given** se adjuntó una imagen con datos EXIF (geolocalización, metadatos del dispositivo), **When** el sistema la almacena, **Then** los metadatos sensibles son eliminados automáticamente antes de guardar.

---

### User Story 4 - Firma Digital del Médico (Priority: P2)

Cada evolución registrada debe quedar asociada de forma irrefutable al médico que la escribió, con validez legal.

**Why this priority**: Buena práctica de trazabilidad: permite saber quién escribió qué y cuándo, protegiend o al médico y a la institución ante cualquier disputa.

**Independent Test**: Se puede probar auditando los metadatos de una evolución guardada y confirmando que incluyen el ID, nombre y matrícula del médico firmante.

**Acceptance Scenarios**:

1. **Given** el médico termina de escribir una evolución, **When** confirma el guardado, **Then** el sistema registra automáticamente el nombre completo, matrícula y timestamp del médico sin requerir acción adicional.
2. **Given** una evolución fue guardada, **When** cualquier usuario intenta modificarla, **Then** el sistema bloquea la edición y solo permite agregar una nueva evolución de corrección vinculada a la original.
3. **Given** el administrador necesita ocultar una evolución confirmada, **When** ejecuta el soft-delete desde el panel de administración, **Then** la evolución cambia a estado **oculto**, desaparece del historial visible para médicos, y el sistema registra en el log: administrador, motivo, timestamp y ID de la evolución afectada.

---

### Edge Cases

- ~~¿Qué ocurre si dos médicos intentan registrar una evolución para el mismo paciente simultáneamente?~~ → **Resuelto**: bloqueo pesimista; el segundo médico ve "En uso por Dr. [nombre]" y accede en solo lectura (ver FR-011).
- ~~¿Cómo se maneja un paciente sin DNI o con datos de identidad incompletos?~~ → **Resuelto**: el DNI es obligatorio; el sistema rechaza el registro si no se proporciona (ver FR-008).
- ¿Qué sucede si la tablet pierde batería a mitad del registro de una evolución larga?
- ~~¿Se permite eliminar una evolución? ¿Bajo qué condiciones y con qué trazabilidad?~~ → **Resuelto**: solo el Administrador puede hacer soft-delete; el registro físico nunca se elimina; el log registra administrador, motivo y timestamp (ver FR-012).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir a médicos autenticados crear registros de evolución de pacientes desde una tablet.
- **FR-002**: El sistema DEBE almacenar cada evolución con fecha, hora, identificador del médico (nombre + matrícula) y contenido textual.
- **FR-003**: El sistema DEBE permitir la consulta del historial clínico unificado de un paciente desde cualquier sede de la red.
- **FR-004**: El sistema DEBE recuperar borradores de evoluciones no guardadas ante cierres inesperados de la aplicación.
- **FR-005**: El sistema DEBE marcar cada evolución como inmutable una vez confirmada; solo puede agregarse una evolución de corrección vinculada.
- **FR-006**: El sistema DEBE soportar adjuntar imágenes y documentos PDF a las evoluciones.
- **FR-007**: El sistema DEBE eliminar automáticamente metadatos sensibles (geolocalización, EXIF) de los archivos adjuntos antes de almacenarlos.
- **FR-008**: El sistema DEBE permitir búsqueda de pacientes por nombre, apellido o DNI. El DNI es campo obligatorio y único; el sistema DEBE rechazar el alta de un paciente sin DNI con un error de validación claro.
- **FR-009**: El sistema DEBE paginar el historial clínico para pacientes con gran volumen de evoluciones.
- **FR-010**: El sistema DEBE registrar un log de auditoría inmutable para toda acción de creación o consulta de evoluciones. Los logs se retienen mientras el sistema esté activo (no hay purga automática en v1).

- **FR-011**: El sistema DEBE implementar bloqueo pesimista sobre evoluciones en edición: cuando un médico abre una evolución para editar, el sistema DEBE bloquearla para otros usuarios. Si un segundo médico intenta editarla, DEBE recibir el mensaje "En uso por Dr. [nombre]" y solo podrá verla en modo lectura. El bloqueo se libera automáticamente al guardar, cancelar o tras 30 minutos de inactividad.
- **FR-012**: El sistema DEBE implementar soft-delete para evoluciones confirmadas. Solo el rol Administrador puede marcar una evolución como **oculta**. El registro físico NUNCA se elimina de la base de datos. El log de auditoría DEBE registrar: ID administrador, motivo (campo de texto obligatorio), timestamp y ID de evolución afectada.

- **Paciente**: Persona atendida. Atributos clave: ID interno, nombre, apellido, **DNI (obligatorio y único)**, fecha de nacimiento, obra social. El sistema DEBE rechazar el registro de un paciente si no se proporciona DNI.
- **Evolución Clínica**: Registro de una consulta. Atributos: ID, ID paciente, ID médico, timestamp, contenido de texto, adjuntos, estado (borrador/confirmado/**oculto**), bloqueo (ID médico que la tiene abierta + timestamp de bloqueo, nullable). Las evoluciones en estado **oculto** no aparecen en el historial visible para médicos pero se conservan físicamente en la base de datos.
- **Médico**: Usuario con rol clínico. Atributos: ID, nombre, apellido, matrícula, especialidad, sede asignada.
- **AdjuntoClínico**: Archivo vinculado a una evolución. Atributos: ID, tipo (imagen/PDF), hash de integridad, URL de almacenamiento seguro.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 95% de los médicos completa el registro de una evolución de texto en menos de 3 minutos desde que abre la pantalla.
- **SC-002**: La búsqueda de pacientes retorna resultados en menos de 2 segundos para bases de hasta 500.000 registros.
- **SC-003**: El 100% de las evoluciones confirmadas contienen firma digital válida (médico + timestamp).
- **SC-004**: La tasa de pérdida de borradores por cierre inesperado es inferior al 0,1%.
- **SC-005**: El historial unificado muestra evoluciones de todas las sedes en una única pantalla sin necesidad de cambiar de contexto.

## Clarifications

### Session 2026-07-01

- Q: ¿Por cuánto tiempo deben retenerse los logs de auditoría? → A: Sin requisito legal (prototipo educativo); los logs persisten mientras el sistema esté activo, sin purga automática.
- Q: ¿Cómo debe identificarse un paciente que no tiene DNI argentino? → A: El DNI es obligatorio; no se pueden registrar pacientes sin él
- Q: Dos médicos abren la evolución del mismo paciente al mismo tiempo. ¿Cómo debe resolverlo el sistema? → A: Bloqueo pesimista: el primero en abrir bloquea la evolución; el segundo ve un mensaje indicando qué médico la tiene abierta
- Q: ¿Se permite eliminar una evolución clínica ya confirmada? → A: Solo el administrador puede hacer soft-delete (marcar como oculta); queda trazabilidad completa en el log de auditoría
- Q: ¿El sistema debe permitir exportar o imprimir evoluciones? → A: Fuera de alcance en v1; no se implementa exportación ni impresión de evoluciones

## Assumptions

- Los médicos tienen una cuenta previamente creada por el administrador del sistema (no hay auto-registro).
- Cada tablet está asignada a una sede específica, aunque el médico puede atender pacientes de otras sedes.
- La firma digital en este prototipo es un registro de metadatos (nombre, matrícula, timestamp); no implica firma criptográfica con certificado X.509.
- Los adjuntos se limitan a imágenes (JPG, PNG) y documentos PDF con un tamaño máximo de 20 MB por archivo.
- El acceso al historial de pacientes está restringido a médicos y personal autorizado explícitamente; pacientes y familiares no acceden directamente desde esta aplicación en v1.
