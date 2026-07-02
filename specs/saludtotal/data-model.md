# Data Model: SaludTotal Backend

**Date**: 2026-07-01 | **Stack**: Java 21 + Hibernate + MySQL 8

## Diagrama ER (texto)

```
Paciente ──< EvolucionClinica >── Usuario (médico)
    |               |
    |          AdjuntoClinico
    |
    ├──< SesionBiometrica ──< EventoTipeo
    |
    └──< ScoreRiesgo ──< AlertaBiometrica

Usuario ──< EventoAuditoria >── Paciente
```

---

## Entidades JPA

### Paciente

```java
@Entity
@Table(name = "pacientes")
@SQLRestriction("activo = true")
public class Paciente {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String nombre;

    @Column(nullable = false)
    private String apellido;

    @Column(nullable = false, unique = true)
    private String dni;

    @Column(nullable = false)
    private LocalDate fechaNacimiento;

    private String obraSocial;
    private String telefono;

    @Column(nullable = false)
    private boolean activo = true;

    @Column(nullable = false)
    private boolean monitoreoActivo = false;

    @Column(nullable = false, updatable = false)
    private LocalDateTime fechaAlta = LocalDateTime.now();
}
```

**Restricciones**:
- `dni` UNIQUE
- `fechaNacimiento` debe ser pasada y paciente ≥ 18 años (validado en DTO)
- `activo = false` → soft-delete (oculto por `@SQLRestriction`)
- `monitoreoActivo` controla si el batch biométrico procesa este paciente

---

### Usuario

```java
@Entity
@Table(name = "usuarios")
public class Usuario implements UserDetails {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String nombre;

    @Column(nullable = false)
    private String apellido;

    @Column(nullable = false, unique = true)
    private String dni;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;  // BCrypt hash

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Rol rol;

    @Column(nullable = false)
    private boolean activo = true;

    @Column(nullable = false, updatable = false)
    private LocalDateTime fechaAlta = LocalDateTime.now();

    private LocalDateTime ultimoAcceso;
}
```

**Restricciones**:
- `dni` UNIQUE, `email` UNIQUE
- `password` nunca se serializa en responses (anotado con `@JsonIgnore`)
- `activo = false` invalida el acceso en `JwtFilter`

**Enum Rol**:
```java
public enum Rol { MEDICO, RECEPCIONISTA, ADMIN }
```

---

### EvolucionClinica

```java
@Entity
@Table(name = "evoluciones_clinicas")
public class EvolucionClinica {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "paciente_id")
    private Paciente paciente;

    @ManyToOne(optional = false)
    @JoinColumn(name = "medico_id")
    private Usuario medico;

    @Column(nullable = false)
    private LocalDateTime timestamp = LocalDateTime.now();

    @Column(columnDefinition = "TEXT", nullable = false)
    private String contenido;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private EstadoEvolucion estado = EstadoEvolucion.BORRADOR;

    // Bloqueo pesimista
    @ManyToOne
    @JoinColumn(name = "bloqueado_por")
    private Usuario bloqueadoPor;

    private LocalDateTime timestampBloqueo;

    // Evolución de corrección vinculada
    @ManyToOne
    @JoinColumn(name = "evolucion_original_id")
    private EvolucionClinica evolucionOriginal;
}
```

**Enum EstadoEvolucion**:
```java
public enum EstadoEvolucion { BORRADOR, CONFIRMADO, OCULTO }
```

**Reglas de negocio** (en `EvolucionService`):
- Solo MEDICO y ADMIN pueden crear/confirmar evoluciones
- Una evolución CONFIRMADA es inmutable; solo se puede agregar una nueva como corrección (`evolucionOriginal != null`)
- OCULTO: solo ADMIN puede setear; nunca se borra físicamente
- Bloqueo: se adquiere al abrir para edición; se libera al guardar/cancelar o tras 30 min (job limpieza)

---

### AdjuntoClinico

```java
@Entity
@Table(name = "adjuntos_clinicos")
public class AdjuntoClinico {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "evolucion_id")
    private EvolucionClinica evolucion;

    @Column(nullable = false)
    private String tipo;           // "IMAGEN" | "PDF"

    @Column(nullable = false)
    private String hashIntegridad; // SHA-256 del archivo

    @Column(nullable = false)
    private String urlAlmacenamiento;
}
```

---

### EventoAuditoria

```java
@Entity
@Table(name = "eventos_auditoria")
public class EventoAuditoria {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String tipoEvento; // "CREAR_EVOLUCION", "CONSULTAR_HISTORIAL", etc.

    @ManyToOne
    @JoinColumn(name = "usuario_id")
    private Usuario usuario;

    @ManyToOne
    @JoinColumn(name = "paciente_id")
    private Paciente paciente;

    @Column(nullable = false)
    private LocalDateTime timestamp = LocalDateTime.now();

    private String dispositivo;
    private String resultado; // "EXITO" | "FALLO"
}
```

**Nota**: El log de auditoría se escribe directamente en el service mediante un método `auditoria.registrar(...)`. No tiene endpoint de escritura pública. Retención: sin purga automática en v1.

---

### SesionBiometrica

```java
@Entity
@Table(name = "sesiones_biometricas")
public class SesionBiometrica {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "paciente_id")
    private Paciente paciente;

    private LocalDateTime timestampInicio;
    private LocalDateTime timestampFin;
    private int cantidadEventos;
    private String dispositivo;
}
```

---

### EventoTipeo

```java
@Entity
@Table(name = "eventos_tipeo")
public class EventoTipeo {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "sesion_id")
    private SesionBiometrica sesion;

    @Column(nullable = false)
    private LocalDateTime timestamp;

    @Column(nullable = false)
    private String tipoEvento; // "PRESS" | "RELEASE"

    private int duracionPulsacionMs;
    private int intervaloDesdeAnteriorMs;
    // NO almacena el carácter presionado
}
```

---

### ScoreRiesgo

```java
@Entity
@Table(name = "scores_riesgo")
public class ScoreRiesgo {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "paciente_id")
    private Paciente paciente;

    @Column(nullable = false)
    private LocalDateTime timestampEvaluacion = LocalDateTime.now();

    @Column(nullable = false)
    private int score; // 0-100; umbral de alerta: >= 70

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private NivelRiesgo nivel; // BAJO(0-39), MEDIO(40-69), ALTO(70-100)

    @Column(nullable = false)
    private boolean alertaGenerada = false;
}
```

**Enum NivelRiesgo**:
```java
public enum NivelRiesgo {
    BAJO,    // score 0-39
    MEDIO,   // score 40-69
    ALTO     // score 70-100 → dispara alerta
}
```

---

### AlertaBiometrica

```java
@Entity
@Table(name = "alertas_biometricas")
public class AlertaBiometrica {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "paciente_id")
    private Paciente paciente;

    @ManyToOne(optional = false)
    @JoinColumn(name = "score_id")
    private ScoreRiesgo score;

    @Column(nullable = false)
    private LocalDateTime timestamp = LocalDateTime.now();

    private String canalNotificacion; // "PUSH" (simulado en prototipo)

    @Column(nullable = false)
    private String estado = "ENVIADA"; // ENVIADA, LEIDA, DESESTIMADA

    private String feedbackMedico; // null | "FALSO_POSITIVO" | "CONFIRMADO"
}
```

---

## Índices recomendados

| Tabla | Columna(s) | Motivo |
|---|---|---|
| `pacientes` | `dni` | Búsqueda directa y unicidad |
| `pacientes` | `apellido`, `nombre` | Búsqueda LIKE paginada |
| `evoluciones_clinicas` | `paciente_id`, `timestamp DESC` | Historial clínico paginado |
| `evoluciones_clinicas` | `bloqueado_por`, `timestamp_bloqueo` | Job de limpieza de bloqueos |
| `eventos_auditoria` | `paciente_id`, `timestamp` | Consulta de auditoría por paciente |
| `scores_riesgo` | `paciente_id`, `timestamp_evaluacion` | Panel biométrico del médico |
| `usuarios` | `email` | Login y unicidad |
| `usuarios` | `dni` | Unicidad |

Hibernate crea PRIMARY KEY e índices UNIQUE automáticamente desde las anotaciones. Los índices adicionales se agregan con `@Table(indexes = {...})` en las entidades si el rendimiento lo requiere.
