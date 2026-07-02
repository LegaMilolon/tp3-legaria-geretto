# API Contract: Evoluciones Clínicas

**Base URL**: `/api/evoluciones`  
**Autenticación requerida**: ✅ Bearer JWT  
**Roles**: MEDICO y ADMIN (RECEPCIONISTA no tiene acceso)

---

## GET /api/evoluciones/paciente/{pacienteId}

Retorna el historial clínico de un paciente ordenado por timestamp DESC (paginado).

**Query params**: `page` (default: 0), `size` (default: 20)

**Response 200 OK**:
```json
{
  "content": [
    {
      "id": 42,
      "pacienteId": 1,
      "medico": { "id": 3, "nombre": "Juan", "apellido": "García", "matricula": "MP12345" },
      "timestamp": "2026-07-01T10:30:00",
      "contenido": "Paciente refiere dolor torácico leve...",
      "estado": "CONFIRMADO",
      "bloqueadoPor": null,
      "evolucionOriginalId": null
    }
  ],
  "totalElements": 15,
  "totalPages": 1
}
```

**Nota**: Las evoluciones en estado `OCULTO` no aparecen en este endpoint.

---

## POST /api/evoluciones

Crea una nueva evolución en estado `BORRADOR`.

**Request body**:
```json
{
  "pacienteId": 1,
  "contenido": "Paciente refiere dolor torácico leve..."
}
```

**Response 201 Created**: objeto evolución con `id` y `estado: "BORRADOR"`  
**Response 404 Not Found**: paciente no existe  
**Response 409 Conflict**: paciente inactivo (dado de baja)

---

## PUT /api/evoluciones/{id}/confirmar

Confirma una evolución (cambia estado BORRADOR → CONFIRMADO). Inmutable después.

**Response 200 OK**: objeto evolución con `estado: "CONFIRMADO"`  
**Response 404**: evolución no existe  
**Response 409**: evolución ya confirmada o no pertenece al médico autenticado

---

## PUT /api/evoluciones/{id}/bloquear

Adquiere el bloqueo pesimista para edición.

**Response 200 OK**: `{ "bloqueado": true }`  
**Response 409 Conflict** (ya bloqueada por otro):
```json
{ "error": "En uso por Dr. Ana Martínez" }
```

---

## PUT /api/evoluciones/{id}/liberar

Libera el bloqueo (al guardar o cancelar edición).

**Response 200 OK**: `{ "bloqueado": false }`

---

## POST /api/evoluciones/{id}/correccion

Agrega una evolución de corrección vinculada a una evolución confirmada.

**Request body**:
```json
{
  "contenido": "Corrección: el valor de glucemia fue 180 mg/dl, no 380."
}
```

**Response 201 Created**: nueva evolución con `evolucionOriginalId: {id}` y `estado: "CONFIRMADO"`

---

## PATCH /api/evoluciones/{id}/ocultar

Solo ADMIN. Soft-delete de evolución (estado → OCULTO).

**Request body**:
```json
{ "motivo": "Evolución duplicada por error del sistema" }
```

**Response 200 OK**: `{ "estado": "OCULTO", "ocultadoPor": "admin@saludtotal.local", "motivo": "..." }`  
**Response 400 Bad Request**: motivo vacío o ausente
