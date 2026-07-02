# API Contract: Autenticación

**Base URL**: `/api/auth`  
**Autenticación requerida**: ❌ (endpoints públicos)

---

## POST /api/auth/login

Autentica un usuario y retorna un JWT.

**Request body**:
```json
{
  "email": "medico@saludtotal.local",
  "password": "medico123"
}
```

**Response 200 OK**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tipo": "Bearer",
  "rol": "MEDICO",
  "nombre": "Juan",
  "apellido": "García"
}
```

**Response 401 Unauthorized** (credenciales inválidas):
```json
{ "error": "Credenciales inválidas" }
```

**Response 401 Unauthorized** (cuenta desactivada):
```json
{ "error": "Cuenta desactivada" }
```

**Claims del JWT**:
```json
{
  "sub": "medico@saludtotal.local",
  "dni": "88888888",
  "email": "medico@saludtotal.local",
  "rol": "MEDICO",
  "iat": 1751328000,
  "exp": 1751414400
}
```
