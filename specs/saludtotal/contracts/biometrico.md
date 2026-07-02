# API Contract: Módulo Biométrico Predictivo

**Base URL**: `/api/biometrico`  
**Autenticación requerida**: ✅ Bearer JWT

---

## POST /api/biometrico/sesion/inicio

Inicia una sesión de captura biométrica para el paciente autenticado.

**Roles**: paciente (via token de app móvil)

**Request body**:
```json
{ "dispositivo": "Samsung Galaxy A54" }
```

**Response 201 Created**:
```json
{ "sesionId": 123 }
```

**Response 409 Conflict**: monitoreo desactivado para este paciente.

---

## POST /api/biometrico/sesion/{sesionId}/eventos

Envía un lote de eventos de tipeo de una sesión activa.

**Request body**:
```json
{
  "eventos": [
    {
      "timestamp": "2026-07-01T10:30:00.123",
      "tipoEvento": "PRESS",
      "duracionPulsacionMs": 85,
      "intervaloDesdeAnteriorMs": 210
    },
    {
      "timestamp": "2026-07-01T10:30:00.335",
      "tipoEvento": "RELEASE",
      "duracionPulsacionMs": 0,
      "intervaloDesdeAnteriorMs": 212
    }
  ]
}
```

**Nota**: NO se envía el carácter presionado, solo las métricas de timing.

**Response 200 OK**: `{ "eventosRegistrados": 2 }`

---

## POST /api/biometrico/sesion/{sesionId}/fin

Finaliza una sesión biométrica.

**Response 200 OK**: `{ "sesionId": 123, "duracionSegundos": 120, "totalEventos": 840 }`

---

## PATCH /api/biometrico/configuracion

Activa o desactiva el monitoreo biométrico para el paciente autenticado.

**Request body**:
```json
{ "activo": false }
```

**Response 200 OK**: `{ "monitoreoActivo": false }`

---

## GET /api/biometrico/historial/{pacienteId}

Retorna los scores de riesgo de los últimos 30 días para un paciente. Solo MEDICO y ADMIN.

**Response 200 OK**:
```json
{
  "pacienteId": 1,
  "scores": [
    {
      "id": 55,
      "fecha": "2026-07-01",
      "score": 72,
      "nivel": "ALTO",
      "alertaGenerada": true
    },
    {
      "id": 54,
      "fecha": "2026-06-30",
      "score": 45,
      "nivel": "MEDIO",
      "alertaGenerada": false
    }
  ]
}
```

---

## POST /api/biometrico/alerta/{alertaId}/feedback

El médico marca una alerta como falso positivo o confirmada. Solo MEDICO y ADMIN.

**Request body**:
```json
{ "feedback": "FALSO_POSITIVO" }
```

Valores válidos: `"FALSO_POSITIVO"` | `"CONFIRMADO"`

**Response 200 OK**: `{ "alertaId": 12, "feedbackMedico": "FALSO_POSITIVO" }`  
**Response 404 Not Found**: alerta no existe

---

## Nota sobre notificaciones push

En el prototipo, el sistema **simula** el envío de notificaciones push mediante un log en consola:

```
[NOTIFICACION PUSH] Paciente ID=1: "Tus patrones de tipeo recientes podrían indicar un riesgo de salud..."
[NOTIFICACION PUSH] Médico ID=3: "Alerta biométrica generada para paciente María González"
```

No se integra Firebase ni ningún servicio real de push en v1.
