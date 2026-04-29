# 🚀 AuthSystem — Guía de Arranque y Pruebas

Sistema de autenticación con JWT + MFA + RBAC + ABAC  
Stack: Spring Boot 3.2 · PostgreSQL · React · Docker

---

## 📋 Requisitos previos

| Herramienta | Versión mínima |
|-------------|---------------|
| Java JDK | 21 |
| Maven | 3.8+ (incluido en IntelliJ) |
| Docker Desktop | Cualquier versión reciente |
| Node.js | 18 LTS o superior |
| IntelliJ IDEA | Community o Ultimate |

---

## 1️⃣ Arrancar la base de datos (Docker)

Abre PowerShell en la raíz del proyecto:

```powershell
cd D:\aRQ_sw\mod04\seguridad\prod_fullstack_auth
docker start postgres-auth
```

Verificar que esté corriendo:

```powershell
docker ps
```

Debe aparecer `postgres-auth` con estado `Up`.

Si el contenedor no existe, créalo con:

```powershell
docker run --name postgres-auth `
  -e POSTGRES_USER=postgres `
  -e POSTGRES_PASSWORD=Pelimm2575 `
  -e POSTGRES_DB=auth_db `
  -p 5432:5432 `
  -d postgres:15
```

---

## 2️⃣ Verificar application.properties

Archivo: `backend/src/main/resources/application.properties`

Debe contener EXACTAMENTE esto:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/auth_db
spring.datasource.username=postgres
spring.datasource.password=Pelimm2575
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

jwt.secret=clave_super_secreta_tecsup_2024_auth_system_32chars
jwt.expiration=900000

server.port=8080
```

---

## 3️⃣ Arrancar el backend (IntelliJ)

1. Abre IntelliJ IDEA
2. Abre el proyecto desde: `backend/`
3. Espera que Maven descargue dependencias (barra inferior)
4. Haz clic derecho en `AuthApplication.java` → **Run**

### ✅ Señales de arranque exitoso

```
Found 2 JPA repository interfaces.
HikariPool-1 - Start completed.
✅ Usuario admin creado (password: password123)
✅ Usuario user creado (password: password123)
✅ Productos de prueba creados
Tomcat started on port 8080
Started AuthApplication in X seconds
```

### ❌ Si falla con "jwt.secret not found"

→ El `application.properties` no tiene las propiedades JWT. Revisa el paso 2.

### ❌ Si falla la conexión a la base de datos

→ Docker no está corriendo. Ejecuta el paso 1 primero.

---

## 4️⃣ Arrancar el frontend (PowerShell)

```powershell
cd D:\aRQ_sw\mod04\seguridad\prod_fullstack_auth\frontend
npm run dev
```

Abrir en el navegador: **http://localhost:5173**  
(o http://localhost:5174 si el 5173 está ocupado)

---

## 5️⃣ Flujo de pruebas completo

### 🔐 Prueba 1 — Login como USER (RBAC básico)

1. Ingresar con `user` / `password123`
2. Resultado esperado: entrar al Dashboard
3. Verificar: **NO aparece** el botón "+ Nuevo producto"
4. Verificar: **NO aparece** el botón "Eliminar"
5. Verificar: **NO aparece** el link "Panel Admin"

✅ Confirma que `ROLE_USER` tiene solo lectura.

---

### 🔐 Prueba 2 — Login como ADMIN (RBAC completo)

1. Cerrar sesión
2. Ingresar con `admin` / `password123`
3. Resultado esperado: entrar al Dashboard
4. Verificar: **SÍ aparece** el botón "+ Nuevo producto"
5. Verificar: **SÍ aparece** el botón "Panel Admin" en la navbar

✅ Confirma que `ROLE_ADMIN` tiene acceso total.

---

### 🔒 Prueba 3 — ABAC (restricción por atributo)

1. Estar logueado como `admin`
2. En la tabla de productos, localizar **"Servidor HP ProLiant"** (bloqueado 🔒)
3. Intentar eliminarlo → botón dice **"🚫 ABAC"**
4. Resultado esperado: mensaje de error **"ABAC denegado: el producto está bloqueado"**

✅ Confirma que incluso ADMIN no puede eliminar productos bloqueados.

---

### 🔒 Prueba 4 — ABAC (eliminación permitida)

1. Estar logueado como `admin`
2. En la tabla, localizar **"Laptop Dell XPS"** (libre ✅)
3. Hacer clic en **"Eliminar"** → confirmar
4. Resultado esperado: mensaje verde **"Producto eliminado correctamente"**

✅ Confirma que ADMIN puede eliminar productos no bloqueados.

---

### 👥 Prueba 5 — Panel de Administración (RBAC)

1. Estar logueado como `admin`
2. Hacer clic en **"Panel Admin"** en la navbar
3. Ver la lista de usuarios: `admin` y `user`
4. Cambiar el rol de `user` → debería cambiar entre `ROLE_USER` y `ROLE_ADMIN`

Para verificar que el endpoint está protegido, prueba en Postman:

```
GET http://localhost:8080/api/admin/usuarios
Sin Authorization header → responde 403 Forbidden
```

✅ Confirma que el panel admin solo es accesible con token de ADMIN.

---

## 6️⃣ Pruebas con Postman (opcional, nivel técnico)

### Login
```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "password123"
}
```
Respuesta: `{ "token": "eyJ...", "role": "ROLE_ADMIN", "status": "SUCCESS" }`

### Listar productos (con token)
```
GET http://localhost:8080/api/productos
Authorization: Bearer <token_del_login>
```

### Eliminar producto bloqueado (debe fallar con 403)
```
DELETE http://localhost:8080/api/productos/2
Authorization: Bearer <token_del_admin>
```
Respuesta esperada: `403 — ABAC denegado: el producto está bloqueado.`

### Acceder a admin sin token (debe fallar con 403)
```
GET http://localhost:8080/api/admin/usuarios
Sin header Authorization
```
Respuesta esperada: `403 Forbidden`

---

## 7️⃣ Reinicio limpio (si algo falla)

```powershell
# 1. Detener backend desde IntelliJ (botón Stop rojo)

# 2. Reiniciar contenedor Docker
docker restart postgres-auth

# 3. Esperar 5 segundos, luego reiniciar backend en IntelliJ

# 4. Reiniciar frontend si es necesario
cd frontend
npm run dev
```

---

## 📁 Estructura del proyecto

```
prod_fullstack_auth/
├── backend/
│   ├── src/main/java/pe/edu/tecsup/auth/
│   │   ├── AuthApplication.java
│   │   ├── config/
│   │   │   ├── SecurityConfig.java     ← JWT + CORS + @EnableMethodSecurity
│   │   │   └── DataInitializer.java    ← Crea usuarios y productos al arrancar
│   │   ├── controller/
│   │   │   ├── AuthController.java     ← /api/auth/login y /verify-mfa
│   │   │   ├── ProductoController.java ← CRUD con RBAC + ABAC
│   │   │   └── AdminController.java    ← /api/admin/** solo ADMIN
│   │   ├── security/
│   │   │   ├── JwtService.java         ← genera y valida tokens
│   │   │   ├── JwtFilter.java          ← intercepta cada request
│   │   │   └── CustomUserDetailsService.java
│   │   ├── service/
│   │   │   ├── AuthService.java
│   │   │   ├── MfaService.java
│   │   │   └── AccessPolicy.java       ← reglas ABAC
│   │   └── model/
│   │       ├── Usuario.java
│   │       └── Producto.java
│   └── src/main/resources/
│       └── application.properties
├── frontend/
│   └── src/
│       ├── api/          ← cliente HTTP con interceptor JWT
│       ├── context/      ← AuthContext (sesión global)
│       ├── pages/        ← Login, MFA, Dashboard, AdminPanel
│       └── components/   ← PrivateRoute, AdminRoute
└── README.md
```

---

## 🧠 Resumen de autorización

| Acción | USER | ADMIN | Regla |
|--------|------|-------|-------|
| Ver productos | ✅ | ✅ | Autenticado |
| Crear producto | ❌ | ✅ | RBAC |
| Editar producto libre | ❌ | ✅ | RBAC |
| Editar producto bloqueado | ❌ | ❌ | RBAC + ABAC |
| Eliminar producto libre | ❌ | ✅ | RBAC + ABAC (nivel > 5) |
| Eliminar producto bloqueado | ❌ | ❌ | ABAC |
| Panel Admin | ❌ | ✅ | RBAC |

