# hellomule

Proyecto de práctica MuleSoft (Mule 4.11) — desarrollado en VS Code con la extensión Mule DX, corriendo sobre runtime local (AnypointCodeBuilder).

Repo: https://github.com/JhojanAlexanderCalambasRamirez/Practica-MuleSoft

## Cómo correr

1. Abrir la carpeta en VS Code (extensiones Mule DX ya instaladas).
2. F5 → "Run Mule Application" (o "Debug Mule Application").
3. App queda escuchando en `http://localhost:8081`.
4. Probar endpoints con Postman o `curl`.

> Importante: editar el código y guardar **no** redeploya solo. Hay que volver a correr (F5) para que los cambios apliquen.

## Estructura del proyecto

```
src/main/mule/
├── global.xml            → configs compartidas (HTTP listener, HTTP request, DB config, configuration-properties)
├── hello-api.xml         → flow de saludo
├── users-api.xml         → capa Process/Experience: endpoints HTTP de usuarios (GET externo, POST, GET db)
├── users-system-api.xml  → capa System API: sub-flows que hablan directo con la base de datos
└── status-api.xml        → health check

src/main/resources/
├── config-dev.yaml  → propiedades de ambiente (puerto, host de API externa, conexión DB)
```

**Por qué está separado así (alta cohesión / bajo acoplamiento):**
Cada archivo tiene una sola responsabilidad (cohesión). Los flows se conectan a las configs de `global.xml` solo por **nombre** (`config-ref`), no por dependencia directa de archivo (acoplamiento bajo). Mule carga todos los XML de `src/main/mule/` como un solo dominio de configuración.

**API-led Connectivity (práctico):**
- `users-system-api.xml` = **System API** ("la despensa") — único archivo que sabe que existe Postgres. Expone sub-flows `sf-db-get-users` y `sf-db-insert-user`.
- `users-api.xml` = **Process/Experience API** ("el mesero") — atiende HTTP, valida, y llama al System API vía `<flow-ref>`. No sabe nada de SQL.

Si mañana cambias Postgres por MongoDB, solo tocas `users-system-api.xml`. El resto del proyecto no se entera (bajo acoplamiento en acción).

## Endpoints

Base path: `/hello` (definido en `HTTP_Listener_config`)

### `GET /hello`
Saludo simple con hora del día y query param opcional.

**Request:**
```
GET http://localhost:8081/hello?nombre=Alex
```

**Response 200:**
```json
{
  "saludo": "Buenas tardes",
  "message": "Hello Alex"
}
```

### `GET /hello/users`
Trae usuarios de una API externa pública (`jsonplaceholder.typicode.com/users`) y los transforma.

**Request:**
```
GET http://localhost:8081/hello/users
```

**Response 200:**
```json
[
  { "id": 1, "nombre": "Leanne Graham", "email": "Sincere@april.biz", "ciudad": "Gwenborough" },
  ...
]
```

**Response 502** (si la API externa falla — timeout, DNS, etc.):
```json
{
  "error": true,
  "mensaje": "No se pudo obtener la lista de usuarios",
  "detalle": "HTTP GET on resource '...' failed: Timeout exceeded."
}
```

### `POST /hello/users`
Crea un usuario y lo **persiste en PostgreSQL** (vía System API → `sf-db-insert-user`).

**Request:**
```
POST http://localhost:8081/hello/users
Content-Type: application/json

{
  "nombre": "Alex",
  "email": "alex@correo.com"
}
```

**Response 201:**
```json
{
  "id": 3,
  "nombre": "Alex",
  "email": "alex@correo.com",
  "creado": "2026-06-10T10:25:29.082084",
  "mensaje": "Usuario creado correctamente"
}
```

**Response 400** (si falta `nombre` o `email`):
```json
{
  "error": true,
  "mensaje": "Los campos 'nombre' y 'email' son obligatorios"
}
```

**Response 409** (si el `email` ya existe — constraint UNIQUE en la tabla):
```json
{
  "error": true,
  "mensaje": "El email ya está registrado",
  "detalle": "ERROR: duplicate key value violates unique constraint \"users_email_key\"..."
}
```

### `GET /hello/users/db`
Lista los usuarios persistidos en PostgreSQL (vía System API → `sf-db-get-users`).

**Request:**
```
GET http://localhost:8081/hello/users/db
```

**Response 200:**
```json
[
  { "id": 1, "nombre": "Alex", "email": "alex@gmail.com", "creado": "2026-06-10T10:17:45.767928" },
  { "id": 3, "nombre": "Camilo", "email": "camilo@gmail.com", "creado": "2026-06-10T10:25:29.082084" }
]
```

> Nota: si ves saltos en los `id` (1, 3, ...) es normal — Postgres incrementa la secuencia `SERIAL` **antes** de validar el constraint UNIQUE. Un insert duplicado fallido igual "consume" un número.

### `GET /hello/status`
Health check — para monitoreo (load balancers, Kubernetes, CloudHub).

**Request:**
```
GET http://localhost:8081/hello/status
```

**Response 200:**
```json
{
  "status": "UP",
  "app": "hellomule",
  "timestamp": "2026-06-09T19:30:00-05:00"
}
```

## Persistencia con PostgreSQL

App corre 100% local: Postgres vía Homebrew (`brew services start postgresql@18`), puerto 5432.

**Setup (una sola vez):**
```sql
-- como superuser (psql -U $(whoami) -d postgres)
CREATE ROLE hellomule_app WITH LOGIN PASSWORD 'hellomule123';
CREATE DATABASE hellomule_db OWNER hellomule_app;

-- conectado a hellomule_db
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  email VARCHAR(150) NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

**Acceso por terminal:**
```bash
psql -U hellomule_app -d hellomule_db -h localhost
```

**Conexión desde Mule** (`config-dev.yaml`):
```yaml
db:
  host: "localhost"
  port: "5432"
  database: "hellomule_db"
  user: "hellomule_app"
  password: "hellomule123"
```

`global.xml` define `<db:config name="Database_Config">` con `db:generic-connection` (driver `org.postgresql.Driver`, url `jdbc:postgresql://${db.host}:${db.port}/${db.database}`).

**Gotcha de classloading (Mule 4):** el DB Connector corre en su propio classloader, aislado del de la app. Aunque el driver `org.postgresql:postgresql` esté en `pom.xml`, el connector no lo ve salvo que se declare como `sharedLibrary` en la config del `mule-maven-plugin`. Sin esto: `Class 'org.postgresql.Driver' not found in classloader for artifact 'container'`.

**Gotcha de `db:insert` + Postgres:** `payload.generatedKeys` viene vacío (Postgres JDBC no devuelve la fila completa sin especificar columnas). Solución usada en `sf-db-insert-user`: insertar y luego `SELECT ... WHERE email = :email` para traer la fila recién creada.

## Conceptos cubiertos

| Concepto | Dónde se usa | Idea clave |
|---|---|---|
| **Mule Flow** | Todos los `.xml` | Pipeline: Source (Listener) → Processors → Response. |
| **Payload** | `set-payload` en cada flow | El "paquete" — cuerpo del mensaje que viaja por el flow. |
| **Attributes** | `attributes.queryParams` | La "etiqueta" del paquete — metadata del request (headers, query params). |
| **Variables** | `vars.saludo`, `vars.httpStatus` | "Post-its" del flow — viven durante la ejecución, no van en la respuesta salvo que se incluyan a propósito. |
| **DataWeave** | `set-payload` con `%dw 2.0` | Lenguaje de transformación. Usa: `map`, `if/else`, `default`, `++`, `now()`, `write()`, `isEmpty()`, `randomInt()`. |
| **Error Handling** | `users-api.xml` (Try + error-handler) | `Try` = red de seguridad. `on-error-continue` atrapa el error y devuelve respuesta controlada en vez de tumbar la app. |
| **Status codes dinámicos** | `http:response` / `http:error-response` con `vars.httpStatus` | El código HTTP de salida depende de lo que pasó adentro del flow (200/201/400/502/500). |
| **Custom errors** | `raise-error type="APP:VALIDATION"` | Errores propios de la app (no solo errores de conectores), atrapables por tipo. |
| **Configuration Properties** | `global.xml` + `config-dev.yaml` | Externalizar config (puertos, URLs) — mismo código, distinto archivo según ambiente. |
| **HTTP** | Listener + Request | Métodos (GET/POST), `allowedMethods`, conectar hacia afuera (`http:request`) y recibir desde afuera (`http:listener`). |
| **JSON** | `mimeType="application/json"` en todos los payloads | Formato de intercambio — objetos, arrays, anidación. |
| **REST** | Endpoints `/hello`, `/hello/users`, `/hello/status` | Recursos + verbos (GET = leer, POST = crear), status codes con significado. |
| **Git** | Todo el repo | `init`, `add`, `commit`, `push`, `.gitignore`, branch `main`. |
| **Alta cohesión / bajo acoplamiento** | Split en 5 archivos XML | Cada archivo = una responsabilidad; conexión por nombre, no por dependencia directa. |
| **DB Connector** | `users-system-api.xml` | `db:config`, `db:select`, `db:insert` — conectar Mule a una base de datos relacional. |
| **Sub-flow + flow-ref** | `users-api.xml` → `users-system-api.xml` | Reutilizar lógica entre flows; base técnica del API-led layering (Process llama a System). |
| **API-led Connectivity** | `users-api.xml` (Process) + `users-system-api.xml` (System) | Capas por responsabilidad: System = acceso a datos, Process/Experience = expone HTTP. |
| **Maven (avanzado)** | `pom.xml` (`sharedLibraries`) | Classloading aislado por conector — un driver JDBC necesita declararse explícitamente para ser visible. |

## Roadmap — qué falta

- [x] **Maven básico**: `mvn clean package`, ciclo de vida (`clean → compile → test → package`), entender `target/`.
- [x] **API-led Connectivity**: capas Experience / Process / System API — analogía restaurante (mesero / cocina / despensa). `users-api.xml` (Process) + `users-system-api.xml` (System).
- [x] **Persistencia real**: PostgreSQL vía DB Connector, CRUD básico (insert + select).
- [ ] **MUnit**: tests automáticos para cada flow.
- [ ] **Ambientes**: `config-prod.yaml` + cómo seleccionar ambiente al correr.
- [ ] **Secure Configuration Properties**: cifrar `db.password` con `secure::` en vez de texto plano.
- [ ] **Deploy**: llevar la app a CloudHub / Runtime Fabric.
