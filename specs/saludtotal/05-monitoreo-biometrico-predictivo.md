# Feature Specification: Monitoreo Biométrico Predictivo de Riesgo Cardíaco

**Feature Branch**: `feat/005-monitoreo-biometrico-predictivo`

**Created**: 2026-07-01

**Status**: Draft

**Input**: User description: "Quiero que la app le avise al paciente si tiene riesgo de infarto midiendo cómo tipea en el teclado (análisis biométrico predictivo)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Recolección de Datos Biométricos de Escritura (Priority: P1)

La aplicación del paciente recopila métricas de tipeo (velocidad, presión, patrones de error, intervalos entre teclas) de manera continua y pasiva durante el uso normal de la app, sin interrumpir la experiencia del usuario.

**Why this priority**: Sin datos biométricos de calidad, el modelo predictivo no puede generar alertas. Esta es la base de toda la funcionalidad del módulo.

**Independent Test**: Se puede probar verificando que el sistema almacena métricas de tipeo (timestamps de teclas, duración de pulsación, intervalos) durante una sesión normal de uso de la app por parte de un usuario.

**Acceptance Scenarios**:

1. **Given** el paciente está usando la app en su dispositivo personal, **When** escribe cualquier texto en cualquier campo de la aplicación, **Then** el sistema registra en segundo plano: timestamp de cada pulsación, duración de pulsación, intervalo entre teclas, errores de tipeo y correcciones.
2. **Given** el sistema recopila datos biométricos, **When** el paciente consulta su configuración de privacidad, **Then** puede ver exactamente qué datos biométricos se recopilan, con opción explícita de desactivar la recopilación.
3. **Given** el paciente desactivó la recopilación biométrica, **When** usa la app, **Then** el sistema no registra ninguna métrica de tipeo y el módulo predictivo permanece inactivo para ese usuario.

---

### User Story 2 - Alerta de Riesgo Cardíaco al Paciente (Priority: P1)

Cuando el modelo predictivo detecta un patrón de tipeo que estadísticamente se asocia a riesgo cardíaco elevado, el sistema debe notificar al paciente de forma clara y accionable.

**Why this priority**: Es el output principal del módulo y la razón por la que el cliente lo requiere. Sin la alerta, toda la recopilación de datos no tiene utilidad para el paciente.

**Independent Test**: Se puede probar inyectando datos biométricos de prueba que simulen el perfil de riesgo alto y verificando que el sistema genera la alerta correspondiente dentro del tiempo esperado.

**Acceptance Scenarios**:

1. **Given** el modelo predictivo calcula que el paciente tiene un score de riesgo cardíaco **≥ 70** basado en sus últimas 72 horas de métricas de tipeo, **When** el score supera ese umbral, **Then** el paciente recibe una notificación push con el mensaje: "Tus patrones de tipeo recientes podrían indicar un riesgo de salud. Te recomendamos consultar con tu médico."
2. **Given** el paciente recibe la alerta, **When** la abre, **Then** puede ver una explicación en lenguaje simple (sin tecnicismos) de qué fue detectado y un botón para contactar a su médico directamente.
3. **Given** el paciente recibe una alerta de riesgo, **When** esta se genera, **Then** también se notifica automáticamente al médico tratante del paciente para que tome acción si lo considera pertinente.

---

### User Story 3 - Panel de Historial Biométrico para el Médico (Priority: P2)

El médico tratante debe poder consultar el historial de métricas biométricas de tipeo y el historial de alertas generadas para sus pacientes, para incorporarlos como dato adicional en la consulta clínica.

**Why this priority**: El valor clínico del módulo aumenta cuando el médico puede interpretar los datos en contexto. Sin esto, las alertas son aisladas y el médico no puede hacer un seguimiento longitudinal.

**Independent Test**: Se puede probar generando datos biométricos de prueba para un paciente y verificando que el médico los puede ver en el panel de historial junto con sus correspondientes evaluaciones de riesgo.

**Acceptance Scenarios**:

1. **Given** el médico está viendo la historia clínica de un paciente que tiene activado el monitoreo biométrico, **When** accede a la sección "Biométrico", **Then** puede ver una línea de tiempo de los últimos 30 días con el score de riesgo diario calculado por el modelo.
2. **Given** el médico consulta el historial biométrico, **When** coloca el cursor sobre un spike (pico de riesgo) en la línea de tiempo, **Then** ve qué métricas específicas contribuyeron al aumento de riesgo ese día.
3. **Given** el médico considera que una alerta fue un falso positivo (por ejemplo, el paciente simplemente estaba cansado), **When** lo marca como tal en el sistema, **Then** el feedback se incorpora al modelo de aprendizaje para mejorar la precisión futura.

---

### User Story 4 - Activar / Desactivar Monitoreo Biométrico (Priority: P2)

El paciente puede encender o apagar el monitoreo biométrico desde la configuración de su app. Al desactivarlo, el sistema deja de recopilar métricas.

**Why this priority**: El usuario debe tener control sobre la funcionalidad. Es una buena práctica de UX que el monitoreo sea opt-in.

**Independent Test**: Se puede probar desactivando el monitoreo y verificando que no se generan nuevos `EventoTipeo` en la base de datos durante el uso posterior de la app.

**Acceptance Scenarios**:

1. **Given** el paciente tiene el monitoreo activo, **When** lo desactiva desde Configuración, **Then** el sistema deja de registrar eventos de tipeo inmediatamente y muestra el estado como "Monitoreo desactivado".
2. **Given** el monitoreo está desactivado, **When** el paciente lo reactiva, **Then** el sistema retoma la recopilación de métricas desde ese momento.

---

### Edge Cases

- ~~¿Qué pasa si el paciente tiene una discapacidad motora que afecta su tipeo de forma permanente? ¿Cómo se evitan falsas alarmas sistemáticas?~~ → **Resuelta**: fuera de alcance; limitación conocida del prototipo. El administrador puede desactivar el módulo biométrico para ese paciente de forma individual.
- ~~¿Cómo se manejan los datos biométricos de un menor de edad que usa la cuenta de un familiar adulto?~~ → **Resuelta**: fuera de alcance; se asume un dispositivo por cuenta.
- ~~¿Qué sucede si el mismo dispositivo es usado por varios miembros de la familia? ¿Se mezclan las métricas biométricas?~~ → **Resuelta**: fuera de alcance; limitación conocida del prototipo (ver Assumptions).
- ~~¿Cuál es la responsabilidad legal de la institución si el sistema no alerta a un paciente que sufre un infarto?~~ → **Descartado**: sistema educativo sin responsabilidad legal.
- ~~¿Cómo se valida científicamente la correlación entre patrones de tipeo y riesgo cardíaco antes del despliegue?~~ → **Descartado**: prototipo, no se requiere validación clínica.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE recopilar métricas de tipeo (intervalos entre teclas, duración de pulsación, patrones de error) de forma pasiva durante el uso normal de la app en el dispositivo del paciente.
- **FR-002**: El monitoreo biométrico es **opt-in**: el paciente puede activarlo o desactivarlo desde la configuración de la app en cualquier momento.
- **FR-003**: Al desactivar el monitoreo, el sistema detiene la recopilación inmediatamente. Los datos históricos previos se conservan para que el médico pueda seguir consultando el historial.
- **FR-004**: El sistema DEBE enviar una notificación push al paciente cuando el score de riesgo cardíaco calculado sea **≥ 70** (escala 0-100), con mensaje en lenguaje claro y no alarmista. El umbral de 70 es fijo en el código.
- **FR-005**: El sistema DEBE notificar simultáneamente al médico tratante cuando se genera una alerta de riesgo para uno de sus pacientes.
- **FR-006**: Los datos biométricos DEBEN almacenarse cifrados y separados de los datos clínicos del paciente; no deben incluir el contenido textual de lo que se escribió, solo las métricas de tipeo.
- **FR-007**: El médico DEBE tener acceso a un panel de historial biométrico por paciente con visualización temporal del score de riesgo.
- **FR-008**: El médico DEBE poder marcar alertas como falsos positivos para contribuir al aprendizaje del modelo.
- **FR-009**: El sistema DEBE recalcular el score de riesgo de cada paciente con monitoreo activo **una vez por día**, mediante un job programado con `@Scheduled` que se ejecuta a las 00:00. El score se **simula** generando un valor aleatorio (0-100) — no hay modelo de IA real en el prototipo. Si el score simulado es ≥ 70, dispara las notificaciones (FR-004, FR-005).

### Key Entities

- **SesiónBiométrica**: Período de uso activo del teclado. Atributos: ID, ID paciente, timestamp inicio/fin, cantidad de eventos de tipeo, dispositivo.
- **EventoTipeo**: Métrica individual de una pulsación. Atributos: ID, ID sesión, timestamp, tipo de evento (press/release), duración pulsación (ms), intervalo desde tecla anterior (ms). **Nota: NO almacena el carácter presionado.**
- **ScoreRiesgo**: Evaluación periódica del riesgo cardíaco. Atributos: ID, ID paciente, timestamp evaluación, score (0-100, **umbral de alerta: ≥ 70**), nivel (bajo: 0-39 / medio: 40-69 / alto: 70-100), métricas contribuyentes, alerta generada (boolean).
- **AlertaBiométrica**: Notificación de riesgo generada. Atributos: ID, ID paciente, ID score, timestamp, canal notificación, estado (enviada/leída/desestimada), feedback médico (null/falso positivo/confirmado).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El monitoreo se detiene en el 100% de los casos dentro de los 2 segundos de que el paciente lo desactiva desde configuración.
- **SC-002**: ~~El modelo predictivo tiene una tasa de precisión documentada antes de integrarlo.~~ **No aplica**: el score es simulado (valor aleatorio); no hay modelo real que evaluar.
- **SC-003**: Las alertas de riesgo llegan al paciente y al médico dentro de los **5 minutos posteriores** a la ejecución del batch diario (00:00); no son alertas en tiempo real.
- **SC-004**: Ningún dato biométrico contiene o permite inferir el contenido textual escrito por el paciente (solo métricas de timing, sin caracteres).

## Clarifications

### Session 2026-07-01

- Q: ¿A partir de qué valor del score (0-100) el sistema envía la alerta? → A: Score ≥ 70.
- Q: ¿Con qué frecuencia se recalcula el score de riesgo? → A: Una vez por día; batch programado a las 00:00 con `@Scheduled` de Spring.
- Q: ¿Cómo se integra el modelo predictivo externo? → A: El modelo no existe; el score se simula con un valor aleatorio (0-100) generado en el batch diario del prototipo.
- Q: Dispositivo compartido por varios miembros de la familia, ¿cómo se maneja? → A: Fuera de alcance; se asume un dispositivo por cuenta. La mezcla de métricas es una limitación conocida del prototipo.
- Q: Paciente con discapacidad motora permanente y falsas alarmas sistemáticas, ¿cómo se maneja? → A: Fuera de alcance; limitación conocida. El administrador puede desactivar el módulo biométrico a nivel individual para ese paciente.

## Assumptions

- El módulo biométrico aplica exclusivamente a la aplicación móvil del paciente (no a las tablets de los médicos).
- **El modelo predictivo es simulado**: el score de riesgo se genera con un valor aleatorio (0-100) en el batch diario. No se integra ningún modelo de IA real en el prototipo.
- Los pacientes utilizan sus propios dispositivos personales. Se asume **un dispositivo por cuenta**; el sistema no detecta ni maneja el caso de dispositivos compartidos entre distintos usuarios. La contaminación de métricas por uso compartido es una limitación conocida del prototipo.
- La funcionalidad biométrica predictiva es opcional y puede ser desactivada a nivel global (para todos los pacientes) o individual (para un paciente específico) por el administrador, sin afectar las demás funcionalidades del sistema.
- **Limitación conocida**: pacientes con condiciones motoras permanentes (Parkinson, temblor esencial, etc.) pueden generar falsas alarmas sistemáticas; no se implementa detección automática de estos casos. Solución: el administrador desactiva el módulo para esos pacientes.
