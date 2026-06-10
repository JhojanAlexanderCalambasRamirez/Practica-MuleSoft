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
├── global.xml            → configs compartidas (HTTP listener, HTTP request, DB config, configuration-properties, secure properties)
├── error-handling.xml    → sub-flow compartido sf-error-response (payload {error, mensaje, detalle})
├── hello-api.xml         → flow de saludo
├── users-api.xml         → capa Process/Experience: endpoints HTTP de usuarios (GET externo, POST, GET db)
├── users-system-api.xml  → capa System API: sub-flows que hablan directo con la base de datos
└── status-api.xml        → health check

src/main/resources/
├── config-dev.yaml  → propiedades de ambiente (puerto, host de API externa, conexión DB; db.password cifrado con secure::; db.sslmode disable/require según ambiente)

src/test/munit/
├── status-api-test.xml        → tests de statusFlow (sin mocks)
├── hello-api-test.xml         → tests de hellomuleFlow (DataWeave, variables)
├── error-handling-test.xml    → tests de sf-error-response (payload de error)
├── users-system-api-test.xml  → tests de sf-db-get-users / sf-db-insert-user (mockea db:select / db:insert)
└── users-api-test.xml         → tests de usersFlow / usersCreateFlow / usersDbFlow (mockea http:request y flow-ref)
```

**Mismo principio en los tests:** un archivo de test por archivo de flow. Los tests de la capa Process (`users-api-test.xml`) mockean el `flow-ref` hacia la System API — no necesitan Postgres corriendo. Los tests de la System API (`users-system-api-test.xml`) mockean `db:select`/`db:insert` directamente.

**Por qué está separado así (alta cohesión / bajo acoplamiento):**
Cada archivo tiene una sola responsabilidad (cohesión). Los flows se conectan a las configs de `global.xml` solo por **nombre** (`config-ref`), no por dependencia directa de archivo (acoplamiento bajo). Mule carga todos los XML de `src/main/mule/` como un solo dominio de configuración.

**API-led Connectivity (práctico):**
- `users-system-api.xml` = **System API** ("la despensa") — único archivo que sabe que existe Postgres. Expone sub-flows `sf-db-get-users` y `sf-db-insert-user`.
- `users-api.xml` = **Process/Experience API** ("el mesero") — atiende HTTP, valida, y llama al System API vía `<flow-ref>`. No sabe nada de SQL.

Si mañana cambias Postgres por MongoDB, solo tocas `users-system-api.xml`. El resto del proyecto no se entera (bajo acoplamiento en acción).

**Error handling compartido (DRY):** `error-handling.xml` define el sub-flow `sf-error-response`, que arma el payload `{error, mensaje, detalle}` a partir de `vars.errorMensaje` + `error.description`. Los 4 `on-error-continue` de `users-api.xml` solo setean `vars.httpStatus` y `vars.errorMensaje`, y delegan el armado de la respuesta con `<flow-ref name="sf-error-response">` — el DataWeave de la respuesta de error vive en **un solo lugar**.

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
  password: "![rAYl487kyCDa4MUnqHtGkA==]"   # cifrado, ver "Secure Configuration Properties"
  sslmode: "disable"   # "require" en Neon/CloudHub, ver sección Deploy
```

`global.xml` define `<db:config name="Database_Config">` con `db:generic-connection` (driver `org.postgresql.Driver`, url `jdbc:postgresql://${db.host}:${db.port}/${db.database}?sslmode=${db.sslmode}`, password `${secure::db.password}`).

**Gotcha de classloading (Mule 4):** el DB Connector corre en su propio classloader, aislado del de la app. Aunque el driver `org.postgresql:postgresql` esté en `pom.xml`, el connector no lo ve salvo que se declare como `sharedLibrary` en la config del `mule-maven-plugin`. Sin esto: `Class 'org.postgresql.Driver' not found in classloader for artifact 'container'`.

**Gotcha de `db:insert` + Postgres:** `payload.generatedKeys` viene vacío (Postgres JDBC no devuelve la fila completa sin especificar columnas). Solución usada en `sf-db-insert-user`: insertar y luego `SELECT ... WHERE email = :email` para traer la fila recién creada.

## Ambientes (Configuration Properties)

`config-dev.yaml` tiene los valores **default de DEV** (commiteado a git, sin secretos en texto plano gracias a Secure Configuration Properties — ver abajo).

Para correr en otro ambiente **no hace falta otro archivo ni lógica de selección**: cualquier `-D<key>=<valor>` pasado a Maven/JVM **sobreescribe** la propiedad del mismo nombre que viene de `<configuration-properties file="config-dev.yaml">`. Es el mismo mecanismo que usa **CloudHub Runtime Manager** en la pestaña "Properties" de un deployment — variables que pisan los valores del `.yaml` empaquetado en el `.jar`.

Verificado:
```bash
mvn test -o -Ddb.database=hellomule_db_prod -Dhttp.port=8082
```

→ 11/11 BUILD SUCCESS, sin tocar `config-dev.yaml`. En CloudHub esto se traduce a: subir el mismo `.jar` y, en Runtime Manager → Properties, agregar `db.database=hellomule_db_prod`, `http.port=8082`, etc.

> **Nota:** Mule no soporta `${prop:default}` (valor por defecto inline) en este contexto — ni en el atributo `file` de `<configuration-properties>`, ni dentro del propio YAML. Por eso `config-dev.yaml` siempre tiene un valor literal de DEV, y el override de otros ambientes es 100% vía `-D` / Runtime Manager Properties.

## Secure Configuration Properties

`db.password` está cifrado en `config-dev.yaml` (no texto plano), usando el módulo `mule-secure-configuration-property-module`.

**`global.xml`:**

```xml
<configuration-properties file="config-dev.yaml" />
<secure-properties:config name="Secure_Properties_Config" file="config-dev.yaml" key="hellomule2026">
    <secure-properties:encrypt algorithm="Blowfish" mode="CBC" />
</secure-properties:config>
...
<db:generic-connection ... password="${secure::db.password}" />
```

**`config-dev.yaml`:**
```yaml
db:
  password: "![rAYl487kyCDa4MUnqHtGkA==]"
```

Valores cifrados van envueltos en `![...]` y se leen con el prefijo `secure::` (no `${db.password}` — `<secure-properties:config>` solo resuelve claves `secure::*`, devuelve vacío para cualquier otra). Por eso `<configuration-properties>` (props planas) y `<secure-properties:config>` (solo `secure::`) **coexisten apuntando al mismo archivo**.

**Generar / verificar el valor cifrado** con la [Secure Properties Tool](https://docs.mulesoft.com/mule-runtime/4.12/_attachments/secure-properties-tool-j17.jar) oficial de MuleSoft:

```bash
java -cp secure-properties-tool-j17.jar com.mulesoft.tools.SecurePropertiesTool string encrypt Blowfish CBC hellomule2026 "hellomule123"
java -cp secure-properties-tool-j17.jar com.mulesoft.tools.SecurePropertiesTool string decrypt Blowfish CBC hellomule2026 "rAYl487kyCDa4MUnqHtGkA=="
```

> **Sobre la `key="hellomule2026"`:** es literal, para portfolio/DEV. En un proyecto real conviene referenciarla como `key="${mule.key}"` y pasarla por `-Dmule.key=...` (o Secure Property en Runtime Manager) — así la clave de cifrado nunca queda en el repo.

## Deploy (CloudHub 2.0)

App corriendo en **CloudHub 2.0** (Anypoint Platform), target `Cloudhub-US-East-2` (US East / Ohio, Public Space).

**URL pública:** <https://hellomule-pyuq0i.5sc6y6-2.usa-e2.cloudhub.io/hello>

**Cómo se desplegó:**

1. `mvn clean package -DskipTests` → genera `target/hellomule-1.0.0-SNAPSHOT-mule-application.jar`.
2. Runtime Manager → Applications → Deploy Application → subir el `.jar` manualmente (no `mvn deploy`, ver gotcha).
3. Runtime: Edge channel, Mule `4.11.5`, Java `17`, 1 replica / 0.1 vCore (defaults).
4. Pestaña **Properties** del deploy — mismo mecanismo que "Ambientes" (arriba), pero seteado en Runtime Manager en vez de `-D`:

| Key | Value |
| --- | --- |
| `db.host` | host de la DB en la nube (`*.neon.tech`) |
| `db.port` | `5432` |
| `db.database` | `neondb` |
| `db.user` | `neondb_owner` |
| `db.sslmode` | `require` |
| `secure::db.password` | password en **texto plano** (ver gotcha) |

**Base de datos:** Postgres local no es accesible desde internet → se creó una DB serverless gratis en [Neon](https://neon.tech) (región `us-east-2`, misma región que CloudHub), con la misma tabla `users` (mismo schema que local).

**`db.sslmode` (nuevo):** Neon exige SSL; Postgres local no. Se agregó `db.sslmode` a `config-dev.yaml` (`"disable"` para local) y a la URL JDBC en `global.xml`: `?sslmode=${db.sslmode}`. En CloudHub se sobreescribe a `require` vía Properties.

**Gotcha — `secure::db.password` como Property de CloudHub no desencripta:** localmente, `${secure::db.password}` se resuelve desencriptando `db.password` de `config-dev.yaml` (Blowfish/CBC). Pero un Property de Runtime Manager con key `secure::db.password` se inyecta como system property y **pisa el placeholder con el valor literal, sin pasar por el desencriptado**. Si se pone ahí el valor cifrado (`![...]`), Postgres recibe esa string literal como password → `password authentication failed`. Conclusión: en Runtime Manager esa property va en **texto plano** (no `![...]`), marcada como "Secure" (ícono candado) para que CloudHub la enmascare en la UI — es una capa de seguridad de plataforma, distinta a la de `mule-secure-configuration-property-module` (que protege el secreto dentro del `.jar`).

**Gotcha — `mvn deploy` no es viable para CH2 sin Exchange:** el `mule-maven-plugin` (`cloudhub2Deployment`) requiere que el `.jar` esté publicado como asset de Exchange (`groupId` = UUID de la org + `distributionManagement` + credenciales en `settings.xml`). Cambiar el `groupId` del proyecto (`com.mycompany`) por un UUID era demasiado invasivo para este repo → se optó por **deploy manual vía Runtime Manager UI**.

## Testing con MUnit

MUnit es el "JUnit de Mule": cada `<munit:test>` tiene 3 partes —

- **`behavior`** (opcional): mocks. `<munit-tools:mock-when processor="db:select">` reemplaza un processor real por una respuesta fija (`<munit-tools:then-return>`), sin tocar Postgres, APIs externas, etc.
- **`execution`**: ejecuta el flow o sub-flow bajo test con `<flow-ref>`.
- **`validation`**: `<munit-tools:assert-that expression="#[...]" is="#[MunitTools::equalTo(...)]">` — compara el resultado con lo esperado.

**5 archivos, 11 tests, todos en verde:**

```bash
mvn clean test
```

```text
>> status-api-test.xml        Tests: 1, Errors: 0, Failures: 0
>> hello-api-test.xml         Tests: 1, Errors: 0, Failures: 0
>> error-handling-test.xml    Tests: 1, Errors: 0, Failures: 0
>> users-system-api-test.xml  Tests: 2, Errors: 0, Failures: 0
>> users-api-test.xml         Tests: 6, Errors: 0, Failures: 0
```

**Mocking de processors (`db:select`, `db:insert`, `http:request`, `flow-ref`):**

```xml
<munit-tools:mock-when processor="db:select">
    <munit-tools:then-return>
        <munit-tools:payload value='#[[{ id: 1, nombre: "Alex", email: "alex@gmail.com" }]]' mediaType="application/java" />
    </munit-tools:then-return>
</munit-tools:mock-when>
```

`processor="mule:flow-ref"` + `<munit-tools:with-attributes><munit-tools:with-attribute attributeName="name" whereValue="sf-db-insert-user" /></munit-tools:with-attributes>` permite mockear **solo** el `flow-ref` hacia la System API — el test de `usersCreateFlow` no necesita Postgres real.

**Inyección de errores** (para probar `error-handler` sin que el fallo ocurra de verdad):

```xml
<munit-tools:then-return>
    <munit-tools:error typeId="HTTP:CONNECTIVITY" />
</munit-tools:then-return>
```

Esto simula que `http:request` falla → dispara el `on-error-continue type="ANY"` → el test verifica `vars.httpStatus == 502`.

**Limitación encontrada:** mockear `attributes.queryParams` de un `http:listener` vía `<munit:set-event>` no funciona de forma confiable en MUnit 3.7.1 (el valor llega `null` al flow). Por eso `hello-api-test.xml` solo cubre el caso sin query param — los casos con query param se prueban manualmente (Postman/curl), documentado como conocimiento real de los límites de la herramienta.

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
| **MUnit** | `src/test/munit/*.xml` | "JUnit de Mule". `munit:test` = `behavior` (mocks) + `execution` (`flow-ref`) + `validation` (`assert-that`). Corre con `mvn test`. |
| **Mocking (mock-when / then-return)** | `users-system-api-test.xml`, `users-api-test.xml` | Reemplaza un processor real (`db:select`, `http:request`, `flow-ref`) por una respuesta fija — aísla la capa bajo test de la BD/API real. |
| **Inyección de errores en tests** | `users-api-test.xml` | `<munit-tools:error typeId="HTTP:CONNECTIVITY">` simula un fallo para probar el `error-handler` (502/409) sin que ocurra de verdad. |
| **Test isolation (alta cohesión en tests)** | `src/test/munit/` (5 archivos) | Mismo principio que el código: un archivo de test por archivo de flow. |
| **Error handling compartido (sub-flow)** | `error-handling.xml` (`sf-error-response`), `flow-ref` en `users-api.xml` | DRY: un solo lugar arma `{error, mensaje, detalle}`; cada flow solo setea `vars.httpStatus` + `vars.errorMensaje`. |
| **Ambientes (override de Configuration Properties)** | `config-dev.yaml` + `-Dkey=valor` | Mismo `.jar`, distinto valor por ambiente — system properties pisan el `.yaml`; análogo a "Properties" en CloudHub Runtime Manager. |
| **Secure Configuration Properties** | `global.xml` (`secure-properties:config`), `config-dev.yaml` (`db.password`) | Cifrar secretos (`![...]`, `secure::`) en vez de texto plano — `mule-secure-configuration-property-module`, Blowfish/CBC. |
| **Deploy a CloudHub 2.0** | Runtime Manager (deploy manual del `.jar`) | Mismo `.jar` + Properties (igual que "Ambientes") corre en la nube; DB cloud (Neon) requiere SSL → `db.sslmode=require`. |

## Roadmap — qué falta

- [x] **Maven básico**: `mvn clean package`, ciclo de vida (`clean → compile → test → package`), entender `target/`.
- [x] **API-led Connectivity**: capas Experience / Process / System API — analogía restaurante (mesero / cocina / despensa). `users-api.xml` (Process) + `users-system-api.xml` (System).
- [x] **Persistencia real**: PostgreSQL vía DB Connector, CRUD básico (insert + select).
- [x] **MUnit**: tests automáticos para cada flow (11 tests, mocks de DB/HTTP/flow-ref, inyección de errores).
- [x] **Error handling compartido**: sub-flow `sf-error-response` en `error-handling.xml`, reusado por los 4 `on-error-continue` de `users-api.xml` (DRY).
- [x] **Ambientes**: `config-dev.yaml` con defaults + override por system properties (`-Dkey=valor`), análogo a CloudHub Runtime Manager Properties.
- [x] **Secure Configuration Properties**: `db.password` cifrado (`![...]`, Blowfish/CBC, `secure::`) con `mule-secure-configuration-property-module`.
- [x] **Deploy**: app corriendo en CloudHub 2.0 (`Cloudhub-US-East-2`), DB en Neon Postgres serverless, deploy manual vía Runtime Manager UI (`.jar` + Properties).
