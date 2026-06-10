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
├── global.xml       → configs compartidas (HTTP listener, HTTP request, configuration-properties)
├── hello-api.xml    → flow de saludo
├── users-api.xml    → flows de usuarios (GET externo + POST simulado)
└── status-api.xml   → health check

src/main/resources/
└── config-dev.yaml  → propiedades de ambiente (puerto, host de API externa)
```

**Por qué está separado así (alta cohesión / bajo acoplamiento):**
Cada archivo tiene una sola responsabilidad (cohesión). Los flows se conectan a las configs de `global.xml` solo por **nombre** (`config-ref`), no por dependencia directa de archivo (acoplamiento bajo). Mule carga todos los XML de `src/main/mule/` como un solo dominio de configuración.

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
Simula la creación de un usuario (no persiste en BD — solo valida y devuelve eco).

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
  "id": 4821,
  "nombre": "Alex",
  "email": "alex@correo.com",
  "mensaje": "Usuario creado (simulado, no persistido)"
}
```

**Response 400** (si falta `nombre` o `email`):
```json
{
  "error": true,
  "mensaje": "Los campos 'nombre' y 'email' son obligatorios"
}
```

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
| **Alta cohesión / bajo acoplamiento** | Split en 4 archivos XML | Cada archivo = una responsabilidad; conexión por nombre, no por dependencia directa. |

## Roadmap — qué falta

- [ ] **Maven básico**: `mvn clean package`, ciclo de vida (`clean → compile → test → package`), entender `target/`.
- [ ] **API-led Connectivity** (teoría): capas Experience / Process / System API — analogía restaurante (mesero / cocina / despensa). `users-api.xml` ya es conceptualmente un mini "System API".
- [ ] **MUnit**: tests automáticos para cada flow.
- [ ] **Ambientes**: `config-prod.yaml` + cómo seleccionar ambiente al correr.
- [ ] **Persistencia real**: conectar a una base de datos (hoy el POST de usuarios es simulado).
- [ ] **Deploy**: llevar la app a CloudHub / Runtime Fabric.
