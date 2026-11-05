# Tarea 01 — Base HTTP y autenticación (`authn`)

**Estado:** pendiente de implementación.  
**Alcance:** primer incremento ejecutable de la API REST Go.  
**Referencia obligatoria:** `AGENTS.md` en la raíz del repositorio y, si existe, `docs/architecture.md`.

## 1. Objetivo

Construir la base mínima, **funcional y comprobable**, de nuestra API REST de fidelización SaaS en Go. Implementar el módulo técnico `authn` para verificar los **access tokens JWT emitidos por Supabase Auth** y exponer una ruta pública y una ruta autenticada.

El resultado debe poder ejecutarse localmente y probarse de forma automatizada **sin credenciales reales ni una base de datos**. Esta tarea no implementa el registro de usuarios ni el dominio de negocios.

## 2. Preparación obligatoria

Antes de modificar archivos:

1. Leer `AGENTS.md`, inspeccionar la estructura y el estado actual del repositorio, y respetar los archivos existentes.
2. Revisar `go.mod` y la organización ya creada. Reutilizar dependencias aprobadas si están presentes.
3. Si **no existe `go.mod`**, pedir primero la ruta definitiva del módulo Go (`module ...`); **no inventarla** ni usar un nombre temporal sin autorización.
4. Confirmar qué mecanismo de firma utiliza el proyecto de Supabase que se integrará: este diseño usa **JWKS y firmas asimétricas**. Si el proyecto aún emplea el secreto legacy HS256, informar el bloqueo y solicitar dirección; no introducir la clave secreta como atajo.
5. Presentar un resumen muy breve del plan y del alcance antes de comenzar. La implementación de esta tarea queda autorizada por la orden explícita de ejecutarla; solicitar confirmación adicional solo ante decisiones bloqueantes.

## 3. Tecnologías y ubicación

- Lenguaje: **Go**.
- Router: **chi**.
- Logs: **slog** (evitar registrar tokens y datos sensibles).
- Supabase Auth para emitir tokens desde los clientes. Nuestra API **solo verifica** tokens; no autentica contraseñas, no emite JWT y no administra sesiones.
- Biblioteca de verificación JWT/JWKS reconocida y mantenida. Elegir **una** opción apropiada para Go, comprobar su compatibilidad y justificarla brevemente; no escribir algoritmos criptográficos manualmente.
- Pruebas con `testing` y `net/http/httptest`. Usar servidores JWKS y claves **de prueba** generadas localmente, nunca llaves reales.
- No añadir PostgreSQL, `pgx`, `sqlc`, Supabase Storage ni otras integraciones que todavía no se usen.

Colocar únicamente los componentes necesarios, respetando la estructura aprobada:

```text
cmd/api/
internal/app/
internal/config/
internal/server/
internal/server/middleware/
internal/authn/
internal/platform/supabase/jwt/
```

Se pueden utilizar `internal/platform/httpx/`, `internal/platform/obs/` y pruebas junto a los paquetes si aportan valor inmediato. **No crear carpetas vacías para todos los módulos futuros** ni reorganizar otras rutas existentes.

## 4. Comportamiento funcional requerido

### 4.1 Inicialización del servidor

- Arrancar un servidor HTTP con configuración validada al inicio.
- Configurar el puerto/dirección HTTP a través del entorno, con un valor predeterminado seguro para desarrollo.
- Configurar mediante variables de entorno, como mínimo, **issuer esperado**, **audience esperada** y **URL fija del JWKS** de Supabase. Documentarlas en `.env.example` con valores ficticios y sin secretos.
- Validar que estos parámetros no estén vacíos cuando la aplicación arranca. Fallar con un error claro si la configuración es inválida.
- Usar timeouts razonables para el servidor y para las consultas HTTP al JWKS. Evitar llamadas de red en cada request cuando la clave ya esté disponible en caché.

### 4.2 Identidad (`authn`)

- Definir un tipo mínimo de identidad autenticada, por ejemplo `Identity`, con el identificador del usuario (`sub`) y el correo cuando esté presente.
- Mantener este contrato independiente del HTTP, del SDK de Supabase y de la implementación de JWT/JWKS.
- Exponer al middleware un contrato pequeño y fácil de sustituir en las pruebas.
- No incluir rol, `business_id`, membresías, sucursales ni permisos dentro de `Identity`: **estos datos no se autorizan mediante claims del JWT**.

### 4.3 Verificador de JWT de Supabase

- Leer el access token desde `Authorization: Bearer <token>`.
- Verificar criptográficamente la firma con las claves públicas del **JWKS configurado**.
- Validar explícitamente `iss`, `aud`, `exp` y los claims temporales aplicables (`nbf`, etc.); exigir un `sub` válido y no vacío.
- Mantener una lista explícita de **algoritmos de firma asimétricos permitidos**, coherente con la configuración real del proyecto. Rechazar `none`, HS256 u otros algoritmos inesperados.
- No confiar en `alg`, `kid`, `iss` o cualquier otro campo no verificado para determinar **arbitrariamente** dónde consultar claves o qué algoritmos aceptar.
- Implementar caché del JWKS con expiración y una estrategia segura para claves nuevas/rotación; limitar tiempos de espera y **fallar de forma cerrada** cuando no sea posible verificar la firma. Un fallo de red no debe hacer que se acepte un token sin verificar.
- No registrar el token JWT ni los contenidos íntegros de sus claims.
- No aceptar tokens de otro proyecto de Supabase, aunque su firma sea válida para otro emisor.

### 4.4 Middleware de autenticación

- Crear un middleware reutilizable para las rutas privadas.
- En ausencia de `Authorization`, ante un esquema que no sea Bearer, un token mal formado, vencido o inválido, responder `401 Unauthorized` con error JSON estable, sin exponer detalles criptográficos internos.
- Tras una verificación correcta, adjuntar la `Identity` al contexto de la petición mediante un acceso tipado y seguro.
- Evitar que los handlers dependan del verificador concreto de Supabase.

### 4.5 Endpoints mínimos

| Método y ruta | Acceso | Resultado |
| --- | --- | --- |
| `GET /healthz` | Público | `200 OK` con JSON sencillo indicando que el proceso HTTP responde. |
| `GET /v1/me` | JWT de Supabase válido | `200 OK` con `user_id` (`sub`) y `email` si está disponible; sin información de tenants ni permisos. |

`GET /v1/me` es **únicamente una prueba de identidad en esta fase**. La funcionalidad completa del cliente, sus tarjetas y su cuenta se incorporará después en los módulos correspondientes. Mantener el contrato pequeño para facilitar su evolución.

Usar códigos HTTP y errores JSON consistentes. No devolver rastros de pila ni mensajes detallados de verificación al usuario final.

## 5. Pruebas obligatorias

Crear pruebas automatizadas independientes de servicios externos de pago y de credenciales reales. Incluir al menos:

### HTTP / middleware

1. `GET /healthz` devuelve `200` **sin token**.
2. `GET /v1/me` sin `Authorization` devuelve `401`.
3. Un encabezado distinto a `Bearer` devuelve `401`.
4. Un token mal formado devuelve `401`.
5. Un token **válido y firmado por una clave de prueba presente en el JWKS de pruebas** devuelve `200`, y se recupera su identidad correcta.

### JWT y seguridad

6. Firma inválida o token alterado: rechazado.
7. Token expirado: rechazado.
8. `issuer` incorrecto: rechazado, aun con firma verificable.
9. `audience` incorrecta: rechazada, aun con firma verificable.
10. `sub` ausente o vacío: rechazado.
11. Algoritmo `none`, HS256 u otro no permitido: rechazado.
12. `kid` desconocido o clave no válida: rechazado.
13. JWKS inaccesible y sin clave de confianza utilizable: **no autorizar** el acceso.
14. Cuando sea practicable, prueba de refresco/rotación de claves del JWKS y de rechazo de una clave revocada después de que expire su período de caché aceptado.

### Configuración y estructura

15. Configuración incompleta o inválida: error de arranque claro, sin exponer secretos.
16. Las pruebas y el código de `authn` no requieren PostgreSQL, `.env` real ni un proyecto de Supabase activo.
17. No existen dependencias circulares y los paquetes del dominio no importan adaptadores de infraestructura.

Preferir pruebas de tabla (*table-driven tests*). Usar `httptest` tanto para HTTP como para un servidor JWKS local y evitar solicitudes reales a Supabase durante `go test`.

## 6. Fuera de alcance: prohibido implementar en esta tarea

- Registro, login, recuperación de contraseña, envío de OTP o plantillas de correo: corresponden al cliente y a la configuración de Supabase Auth.
- Creación de perfiles globales en `accounts` o acceso de clientes a sus tarjetas.
- Consulta de membresías, roles o permisos de `authz`; el JWT no concede acceso de staff por sí mismo.
- Empresas, sucursales, empleados, suscripciones, planes y entitlements.
- Tablas SQL, migraciones, acceso a PostgreSQL o políticas RLS.
- QR, sellos, canjes, recompensas, Storage, pagos o notificaciones.
- Panel Flutter, cliente web, panel de administración o despliegue de producción.
- Microservicios, colas, motores de reglas o arquitectura adicional innecesaria.

No introducir implementaciones vacías de los puntos anteriores para declarar la fase completa.

## 7. Validación y entrega

Antes de dar esta tarea por finalizada:

1. Ejecutar `gofmt` en los archivos Go modificados.
2. Ejecutar `go test ./...`.
3. Ejecutar `go test -race ./...` si el entorno dispone de soporte para ello.
4. Ejecutar `go vet ./...` si está disponible.
5. Verificar que el servidor puede arrancar con configuración de ejemplo adaptada a un entorno de pruebas, sin afirmar que se verificó Supabase real si no se hizo.
6. Entregar un resumen de:
   - Archivos creados o modificados.
   - Bibliotecas nuevas y motivo de su uso.
   - Endpoints implementados y sus respuestas.
   - Pruebas ejecutadas y resultado **real** de cada comando.
   - Riesgos o bloqueos pendientes para conectar Supabase real.

**Criterio de aceptación:** la API compila, ofrece los dos endpoints, acepta únicamente JWT válidos del emisor configurado, rechaza los escenarios negativos y supera la suite automatizada. Si alguna prueba no se pudo ejecutar, indicarlo con claridad; no darla por aprobada de forma ficticia.

**Detente al completar esta tarea. No continúes con `accounts` ni con otros módulos sin una nueva instrucción.**