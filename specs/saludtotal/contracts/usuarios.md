# API Contract: Usuarios (Panel de Administración)

**Base URL**: `/api/usuarios`  
**Autenticación requerida**: ✅ Bearer JWT  
**Roles**: Solo ADMIN (todos los endpoints)

---

## GET /api/usuarios

Lista usuarios con paginación y filtros opcionales.

**Query params**:
- `rol` (opcional): `MEDICO` | `RECEPCIONISTA` | `ADMIN`
- `activo` (opcional): `true` | `false`
- `page` (default: 0), `size` (default: 20)

**Response 200 OK**:
```json
{
  "content": [
    {
      "id": 1,
      "nombre": "Admin",
      "apellido": "Sistema",
      "dni": "99999999",
      "email": "admin@saludtotal.local",
      "rol": "ADMIN",
      "activo": true,
      "fechaAlta": "2026-07-01T00:00:00",
      "ultimoAcceso": "2026-07-01T10:00:00"
    }
  ],
  "totalElements": 3,
  "totalPages": 1
}
```

**Nota**: El campo `password` nunca aparece en ninguna respuesta.

---

## GET /api/usuarios/{id}

**Response 200 OK**: objeto usuario (mismo schema)  
**Response 404 Not Found**: `{ "error": "Usuario no encontrado" }`

---

## POST /api/usuarios

**Request body**:
```json
{
  "nombre": "Carlos",
  "apellido": "López",
  "dni": "44556677",
  "email": "carlos@saludtotal.local",
  "password": "temporal123",
  "rol": "MEDICO"
}
```

**Validaciones**: todos los campos `@NotBlank`; `email` debe ser válido; `rol` debe ser un valor del enum.

**Response 201 Created**: objeto usuario con `id` (sin `password`)  
**Response 400 Bad Request**: campo inválido  
**Response 409 Conflict**: `{ "error": "El email ya está en uso" }` o `{ "error": "El DNI ya está en uso" }`

---

## PUT /api/usuarios/{id}

**Request body** (todos los campos editables):
```json
{
  "nombre": "Carlos Andrés",
  "apellido": "López",
  "dni": "44556677",
  "email": "carlosandres@saludtotal.local",
  "rol": "ADMIN"
}
```

**Response 200 OK**: objeto usuario actualizado  
**Nota**: Al cambiar el rol, las sesiones activas del usuario quedan invalidadas en el siguiente request (el `JwtFilter` verifica contra la DB).

---

## PATCH /api/usuarios/{id}/estado

Activa o desactiva una cuenta.

**Request body**:
```json
{ "activo": false }
```

**Response 200 OK**: `{ "id": 2, "activo": false }`  
**Response 403 Forbidden** (auto-desactivado):
```json
{ "error": "Un administrador no puede desactivar su propia cuenta" }
```
**Response 409 Conflict** (único admin):
```json
{ "error": "No se puede desactivar el único administrador activo del sistema" }
```

---

## POST /api/usuarios/{id}/reset-password

**Request body**:
```json
{ "nuevaPassword": "nueva_clave_temporal" }
```

**Response 200 OK**: `{ "mensaje": "Contraseña actualizada correctamente" }`  
**Response 409 Conflict** (cuenta desactivada):
```json
{ "error": "La cuenta está desactivada; reactive la cuenta antes de resetear la contraseña" }
```

---

## DELETE /api/usuarios/{id}

Eliminación física. Solo si el usuario no tiene actividad en el log de auditoría.

**Response 204 No Content**: eliminado exitosamente  
**Response 409 Conflict**:
```json
{ "error": "No se puede eliminar un usuario con actividad registrada; use desactivar en su lugar" }
```
