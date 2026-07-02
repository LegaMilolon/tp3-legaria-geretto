# API Contract: Pacientes

**Base URL**: `/api/pacientes`  
**Autenticación requerida**: ✅ Bearer JWT  
**Roles**: Ver matriz de permisos

## Matriz de permisos

| Operación | MEDICO | RECEPCIONISTA | ADMIN |
|---|---|---|---|
| GET (listar/buscar) | ✅ | ✅ | ✅ |
| GET /{id} | ✅ | ✅ | ✅ |
| POST (crear) | ✅ | ✅ | ✅ |
| PUT /{id} (editar) | ✅ | ✅ | ✅ |
| DELETE /{id} (soft-delete) | ❌ | ❌ | ✅ |
| GET /inactivos | ❌ | ❌ | ✅ |

---

## GET /api/pacientes

Lista pacientes activos con paginación y búsqueda opcional.

**Query params**:
- `q` (opcional): búsqueda por nombre, apellido o DNI (case-insensitive, tolerante a tildes)
- `page` (default: 0), `size` (default: 20)

**Response 200 OK**:
```json
{
  "content": [
    {
      "id": 1,
      "nombre": "María",
      "apellido": "González",
      "dni": "30123456",
      "fechaNacimiento": "1985-03-15",
      "obraSocial": "OSDE",
      "telefono": "11-4567-8901",
      "fechaAlta": "2026-07-01T10:00:00"
    }
  ],
  "totalElements": 1,
  "totalPages": 1,
  "number": 0,
  "size": 20
}
```

---

## GET /api/pacientes/{id}

**Response 200 OK**: objeto paciente (mismo schema que arriba)  
**Response 404 Not Found**: `{ "error": "Paciente no encontrado" }`

---

## POST /api/pacientes

**Request body**:
```json
{
  "nombre": "María",
  "apellido": "González",
  "dni": "30123456",
  "fechaNacimiento": "1985-03-15",
  "obraSocial": "OSDE",
  "telefono": "11-4567-8901"
}
```

**Validaciones**:
- `nombre`, `apellido`, `dni`: `@NotBlank`
- `fechaNacimiento`: `@NotNull` + pasada + edad ≥ 18 años

**Response 201 Created**: objeto paciente con `id` asignado  
**Response 400 Bad Request**: `{ "errores": { "dni": "no debe estar vacío" } }`  
**Response 400 Bad Request**: `{ "error": "El paciente debe ser mayor de 18 años" }`  
**Response 409 Conflict**: `{ "error": "Ya existe un paciente con ese DNI" }`

---

## PUT /api/pacientes/{id}

**Request body** (campos editables):
```json
{
  "nombre": "María Eugenia",
  "apellido": "González",
  "dni": "30123456",
  "obraSocial": "Swiss Medical",
  "telefono": "11-9999-0000"
}
```

**Response 200 OK**: objeto paciente actualizado  
**Response 404 Not Found**: paciente no existe  
**Response 409 Conflict**: DNI ya en uso por otro paciente

---

## DELETE /api/pacientes/{id}

Solo ADMIN. Realiza soft-delete (`activo = false`).

**Response 204 No Content**: baja exitosa  
**Response 404 Not Found**: paciente no existe  
**Response 409 Conflict** (evolución con bloqueo activo):
```json
{ "error": "El paciente tiene una evolución en edición activa por Dr. Juan García" }
```

---

## GET /api/pacientes/inactivos

Solo ADMIN. Lista pacientes dados de baja.

**Response 200 OK**: lista paginada (mismo schema, incluye `activo: false`)
