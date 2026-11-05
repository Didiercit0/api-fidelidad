## Propósito

Este repositorio contiene el backend de una plataforma SaaS multiempresa para programas de fidelización: tarjetas digitales, códigos QR, sellos, recompensas, sucursales, empleados, clientes y suscripciones.

Actúa como colaborador de ingeniería. Implementa **solo el alcance solicitado**, preserva las decisiones aprobadas, señala bloqueos reales y entrega cambios pequeños, verificables y seguros. No interpretes este archivo como autorización para implementar toda la plataforma de una vez.

## Arquitectura y tecnologías aprobadas

- **Go** para una **API REST** independiente, organizada como **monolito modular**. No crear microservicios en el MVP.
- **PostgreSQL** alojado en Supabase; acceso a datos con **pgx + sqlc**, SQL explícito y **sin ORM**.
- **Supabase Auth** para autenticar cuentas; **Supabase Storage** para archivos. La API Go gobierna autorización y lógica de negocio.
- Router HTTP **chi**, registro estructurado con **slog** y herramientas estándar de pruebas de Go.
- Una única herramienta de migraciones, elegida y documentada antes de crear migraciones; no efectuar cambios manuales desde el panel de producción.
- Respetar los nombres, contratos, módulos y estructura de carpetas aprobados. No reestructurar el proyecto por iniciativa propia.
- Mantener dependencias acíclicas entre módulos y contratos explícitos para sus colaboraciones. Usar `internal/app` como *composition root*.
- El núcleo del dominio no debe depender de `net/http`, `pgx`, SDK de Supabase o implementaciones de almacenamiento. Mantener los adaptadores HTTP en `rest/` y persistencia en `store/`, según corresponda.
- Evitar infraestructura prematura: microservicios, colas, event bus, motores de reglas, ORM, patrones y dependencias no justificadas.

## Organización del dominio

Los módulos previstos bajo `internal/` son:

| Módulo | Responsabilidad principal |
| --- | --- |
| `accounts` | Perfil local de la cuenta global vinculada al `sub` de Supabase Auth. |
| `businesses` | Empresa/establecimiento principal (*tenant*), configuración y alta coordinada desde invitación. |
| `branches` | Sucursales y sus estados dentro de un establecimiento. |
| `members` | Membresías del personal, invitaciones de empleados y asignaciones a sucursales. |
| `platformstaff` | Operadores de nuestra empresa SaaS, independientes del personal de los negocios. |
| `subscriptions` | Catálogo versionado de planes, admisión por invitación, cupos piloto, suscripciones, activación, vencimientos y cambios de plan. |
| `customers` | Registro de cliente dentro de cada tenant, vínculo con cuenta global y consentimiento. |
| `designs` | Diseños y configuraciones visuales de tarjetas. |
| `programs` | Programas, recompensas y versiones inmutables de sus reglas. |
| `cards` | Tarjetas, ciclos, inscripción, sellos, canjes, reversiones, QR de tarjeta e historial contable. |
| `assets` | Metadatos y operaciones autorizadas de archivos mediante Storage. |
| `stats` | Estadísticas y consultas de lectura. |
| `audit` | Auditoría de acciones administrativas. |
| `exports` | Exportación de datos de un establecimiento, incluida la salida del servicio. |

Además, conservar los paquetes técnicos `authn`, `authz`, `entitlements`, `server`, `config`, `platform/*`, `app` y utilidades de pruebas. **QR, sellos y recompensas no son microservicios ni módulos independientes**: las operaciones de tarjeta y su libro mayor pertenecen a `cards`; las definiciones de recompensas y reglas pertenecen a `programs`.

## Reglas innegociables de seguridad

1. **Autenticación no es autorización.** Validar JWT de Supabase con `issuer`, `audience`, algoritmos permitidos y URL JWKS **configurados**. Verificar firma, expiración y demás claims aplicables. No construir un endpoint JWKS a partir de un `iss` no verificado ni confiar en el encabezado `alg` sin lista permitida.
2. El JWT identifica la cuenta. Los roles, la pertenencia a un tenant, el estado de la membresía y el alcance de sucursales se consultan en la base de datos. No usar `user_metadata` o claims proporcionados por el cliente como autoridad para permisos.
3. **Aislamiento multiempresa obligatorio.** Todos los recursos propios de un negocio se consultan y modifican dentro del tenant autorizado. No aceptar un `business_id` del cliente como autorización suficiente. Comprobar además que un cliente solo accede a **sus propias** tarjetas, incluso dentro de un mismo tenant.
4. PostgreSQL RLS es una **segunda defensa**, no reemplaza los filtros y permisos del servicio. Usar rol de base de datos de mínimos privilegios (como `app_api`), sin bypass de RLS. Las operaciones de tenant deben ejecutarse mediante una transacción con contexto de tenant establecido de forma local y segura.
5. Las transacciones deben ser explícitas; no filtrar tipos concretos de `pgx` al núcleo del dominio. Preservar límites transaccionales para altas coordinadas y movimientos de tarjetas.
6. Sellos y canjes requieren validación de empleado, sucursal, reglas, límites y capacidad. Cada movimiento debe ser **idempotente**, seguro ante concurrencia y registrado en un libro mayor **append-only**, con reversiones auditables en vez de ediciones destructivas.
7. Un empleado no puede otorgar sellos ni canjear recompensas en su propia tarjeta. No exponer datos de empleados en el historial visible para el cliente.
8. Validar en la API todos los permisos y límites comerciales; ocultar una opción en Flutter o en el cliente web nunca es suficiente.
9. No registrar JWT, OTP, contraseñas, claves, datos personales innecesarios ni URLs firmadas en logs. No incrustar secretos en código, documentación, pruebas o commits.
10. Controlar las rutas públicas de inscripción con datos mínimos, validación de código, protección frente a abuso y respuestas que no filtren información privada.

## Reglas de negocio aprobadas para el MVP

- **Piloto:** acceso exclusivo por invitación; cohorte inicial configurable de **20 establecimientos**. Esta cifra **no** es un límite fijo de arquitectura ni representa clientes finales. No hay registro público de establecimientos.
- **Duración piloto:** **30 días a partir de la activación**, no desde la emisión o aceptación de la invitación. La activación requiere un estado previo de configuración.
- Cada **tenant** administra sus sucursales, empleados, programas y clientes; la suscripción se asigna al tenant, no a cada sucursal.
- En el MVP, los programas de fidelización son válidos en **todas las sucursales del mismo tenant**. Una tarjeta acumula sellos entre ellas y cada operación conserva la sucursal de origen. Nunca entre tenants distintos.
- Los clientes cuentan con una **cuenta global** de Supabase Auth, mediante correo y **OTP**; pueden utilizar un cliente web sin instalar una app. Una cuenta puede tener tarjetas de distintos negocios sin revelar esas afiliaciones entre negocios.
- La inscripción a un programa se inicia por QR, exige autenticación del cliente y crea de forma idempotente su registro en el negocio y su tarjeta. Una tarjeta por cliente y programa.
- Tres modalidades de acumulación del MVP: **por compra registrada**, **por compra mínima** y **por visita válida**, con validación manual del empleado y parámetros tipados de intervalo/topes/monto.
- Las reglas que afectan sellos o canjes se versionan de forma inmutable; los ciclos de tarjeta existentes conservan la versión aplicable.
- `subscriptions` mantiene el ciclo de vida de la suscripción y la admisión al piloto; `entitlements` expone capacidades, modos de operación y límites consumibles por los demás módulos. Evitar comprobaciones dispersas `if plan == "pro"`.
- Contemplar los estados: pendiente de activación, prueba, activa, gracia, consulta, archivado y baja; una suspensión administrativa puede bloquear operaciones independientemente del estado.
- Un downgrade o vencimiento **no elimina** sucursales, tarjetas, sellos, recompensas ni historial de forma automática. La baja definitiva requiere un procedimiento explícito y auditado; la exportación de salida debe existir para todos los planes.
- **Sin pasarela de pagos ni cobros recurrentes en el MVP.** Los precios y prestaciones exactas de planes comerciales siguen sujetos a definición.

## Decisiones pendientes: no inventarlas

Cuando afecten una tarea, detenerse y solicitar una definición concreta antes de introducir una decisión irreversible:

- Detalles del proveedor SMTP, configuración y verificación real del OTP en staging.
- Política de conservación y anonimización al eliminar cuentas, y datos de cliente copiados a cada negocio.
- Fecha exacta de corte del piloto, liberación de cupos y mecanismo definitivo de activación.
- Requisitos definitivos de canje y si el canje se permite durante modo Consulta.
- Políticas para emisión, rotación y mitigación de abuso del QR de tarjetas y de inscripción.
- Herramienta única de migraciones y parámetros operativos finales cuando todavía no estén aprobados.

No tratar como decisiones cerradas los valores provisionales de períodos de gracia, consulta o retención. La necesidad de una fase posterior no autoriza a implementarla ahora.

## Flujo obligatorio al trabajar con Codex

1. Leer este archivo, inspeccionar el código existente y consultar `docs/architecture.md` y la tarea específica en `docs/tasks/` **si existen**. La actualización final del MVP prevalece sobre borradores arquitectónicos anteriores. Si falta documentación crítica, informar el vacío; no recrear decisiones arbitrariamente.
2. Identificar el objetivo, las dependencias y los criterios de aceptación de **una sola tarea**. Si hay ambigüedades de seguridad, dinero, permisos, cambios de datos o alcance, preguntar antes de implementarlas.
3. Respetar el repositorio y su estado actual. No sobrescribir cambios del usuario, no reorganizar módulos no solicitados y no ejecutar operaciones destructivas sin autorización.
4. Implementar el cambio mínimo completo, con pruebas correspondientes. No añadir funciones ficticias o *stubs* que aparenten cumplimiento sin comportamiento real.
5. Ejecutar `gofmt` sobre los archivos Go modificados, `go test ./...` y, cuando el entorno lo permita, `go test -race ./...`. Ejecutar también verificaciones de integración y aislamiento aplicables cuando exista una base de pruebas configurada. Reportar honestamente cualquier prueba que no se haya podido ejecutar.
6. Para endpoints, cubrir rutas exitosas y errores, credenciales inválidas, permisos insuficientes y límites de tenant pertinentes. En casos sensibles, añadir pruebas de regresión de concurrencia, idempotencia, RLS y aislamiento entre clientes/negocios.
7. Al finalizar, resumir: archivos creados o modificados, comportamiento entregado, pruebas ejecutadas con resultado real, supuestos pendientes y siguiente paso propuesto. **No avanzar al siguiente módulo sin una nueva instrucción.**

## Primer hito de implementación

Comenzar por la **base ejecutable de la API y el módulo `authn`** cuando se solicite su tarea específica. El primer incremento debe probar la verificación de tokens de Supabase, un endpoint de salud público y una ruta protegida de identidad, mediante pruebas automatizadas. **No implementar todavía** `accounts`, `businesses`, migraciones de negocio o el resto de módulos solo por estar enumerados aquí.

Antes de considerar lista la Fase 1 completa, validar los tres riesgos técnicos ya identificados: verificación JWT con el proyecto real y rotación de claves; rol `app_api` + pooler + RLS; y especificación de ledger de tarjetas (idempotencia, concurrencia, modalidades y reversión). Los que no correspondan al primer incremento de `authn` deben registrarse como pendientes, no implementarse fuera de alcance.
