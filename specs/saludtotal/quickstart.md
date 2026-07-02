# Quickstart: SaludTotal Backend

**Stack**: Java 21 · Spring Boot · MySQL 8 · Gradle (Groovy DSL)

## Prerrequisitos

- Java 21 instalado (`java -version`)
- MySQL 8 corriendo localmente
- Gradle 8+ (o usar el wrapper `./gradlew`)

## 1. Clonar y configurar variables de entorno

```bash
git clone <repo-url>
cd saludtotal
```

Crear un archivo `.env` (o setear las variables en el shell):

```bash
export DB_HOST=localhost
export DB_PORT=3306
export DB_NAME=saludtotal
export DB_USER=root
export DB_PASS=tu_password_local
export JWT_SECRET=un_secreto_largo_de_al_menos_32_caracteres
```

> **Nunca** commitear `.env` al repositorio. Agregarlo a `.gitignore`.

## 2. Crear la base de datos

```sql
CREATE DATABASE IF NOT EXISTS saludtotal
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

> Hibernate crea todas las tablas automáticamente con `ddl-auto=create` al iniciar la aplicación por primera vez.

## 3. Compilar y ejecutar

```bash
# Con Gradle wrapper
./gradlew bootRun

# O compilar primero y ejecutar el JAR
./gradlew build
java -jar build/libs/saludtotal-0.0.1-SNAPSHOT.jar
```

La aplicación levanta en `http://localhost:8080`.

## 4. Verificar que levantó correctamente

```bash
curl http://localhost:8080/actuator/health
# {"status":"UP"}
```

## 5. Acceder a Swagger UI

Abrir en el navegador:

```
http://localhost:8080/swagger-ui/index.html
```

Para probar endpoints protegidos:
1. Hacer `POST /api/auth/login` con las credenciales del admin semilla
2. Copiar el token JWT de la respuesta
3. Hacer clic en "Authorize" en Swagger UI
4. Pegar el token en el campo `Bearer <token>`

## 6. Credenciales semilla (DataInitializer)

Al iniciar, el `DataInitializer` crea automáticamente:

| Rol | Email | Contraseña | DNI |
|---|---|---|---|
| ADMIN | admin@saludtotal.local | admin123 | 99999999 |
| MEDICO | medico@saludtotal.local | medico123 | 88888888 |
| RECEPCIONISTA | recepcion@saludtotal.local | recep123 | 77777777 |

> Solo para desarrollo local. Cambiar en producción.

## 7. Ejecutar tests

```bash
./gradlew test
```

Los tests unitarios (capa Service) no requieren base de datos. Los tests de integración requieren MySQL corriendo con las mismas variables de entorno.

## 8. Estructura de `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '3.x.x'
    id 'io.spring.dependency-management' version '1.x.x'
    id 'java'
}

group = 'com.saludtotal'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '21'

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.x.x'
    implementation 'io.jsonwebtoken:jjwt-api:0.12.x'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.x'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.x'
    runtimeOnly 'com.mysql:mysql-connector-j'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

## 9. Problemas comunes

| Problema | Causa probable | Solución |
|---|---|---|
| `Access denied for user` | Variables de entorno no seteadas | Verificar `DB_USER` y `DB_PASS` |
| `Table doesn't exist` | Base de datos no creada | Ejecutar el `CREATE DATABASE` del paso 2 |
| `Invalid JWT signature` | `JWT_SECRET` vacío o distinto al usar | Verificar que la variable está seteada |
| Puerto 8080 en uso | Otro proceso en el puerto | `lsof -i :8080` y matar el proceso, o cambiar `server.port` |
