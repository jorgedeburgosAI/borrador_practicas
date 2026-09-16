

# Ideas principales

## Análisis de nivel vertical de clientes

Se deberia de analizar si el cliente a entrar en dicha arquitectura esta al mismo nivel vertical que los demás ya que puede suponer una degradacion de rendimientos en los demas tenant. Posible solucion crear una solucion especifica para el cliente o si se prevee clientes con mismas necesidades crear otro multitenant(estudiar viabilidad para rdto optimo sin latencais), de dicha manera nos quitamos los incovenientes de escalabilidad vertical. Para ello deberiamos visualizar el nivel previsto de exigencias y escoger un stack diferente según las necesidades.

## Decisión de estructura de BBDD

Tras analizar como se podria estructurar la bbdd según las diferentes exigencias de los clientes, llego a la conclusión en nuestro caso que lo mas conveniente seria **Shared schema (pool) + Row-Level Security**, sin particionado ni sharding.

¿Porque? El sharding nos cubre de problemas que en principio no necesitamos cubrir debido a las necesidades de nuestro caso, si aceptaramos una escalabilidad de clientes dentro de este multitenant estudiariamos la opción mas viable para una migración, si no se diseña algo personal u otro multitenant, de la misma manera para el particionado físico, ya que esta resuuelve problemas para el borrado rapido cuando las tablas son demasiado grandes, por lo que el delete masivo es un cuello de botella real, ademas que el mantenimiento constante no lo hace práctico debidoal consumo de tiempo para nuestra situación, no es óptimo, particionar por `tenant_id` es una optimización de escala y de operaciones (borrado rápido, rendimiento en tablas grandes) que no aporta seguridad, y con exigencia media-baja el problema que resuelve todavía no existe — así que solo queda el coste de mantenerla, sin el beneficio.

Para el caso de extender el eje a cómputo, red y cuenta cloud es una opción muy interesante pero lo descarto por la sencilla razón de nuestro target (la complejidad de explicar y auditar va a cada capa y para casos reales a los que apuntamos no tiene sentido dicha caer en dicha complejidad, ya que complicariamos mucho la venta del SaaS y perderiamos potenciales clientes), tampoco se benefician los problemas que resuelve ya que aisla cómputos para no saturar la cpu/memoria cosa que no nos va a pasar al igual que el aislamiento por red ya que es altamente burocratico (sector financiero, salud y defensa) como con cuenta cloud, que normamelmente lo piden clientes de auditorias. El despliegue serai muy grande porque dependeria de N desiciones independientes cada uno con su propio coste, además de que al requerir tanta configuración se puede caer en la incosistencia, en deefinitica contradice nuestro argumento de coste-beneficio.

## Buenas prácticas de seguridad

En buenas practicas para la seguridad aconsejan seguir 4 pasos:

### 1. `tenant_id` con token de autenticación (JWT)

Esto supone ventajas en seguridad ya que el cliente no puede alterar su propio `tenant_id`, pero presenta varios incovenientes sensibles a considerar y subsanar.

Primero: JWT con TTL largo, si el ususario cambia de tenant (como sera nuestro caso) el token queda desactualizado hasta que expira o se refresca, esto supone un impacto a nivel producción, y se puede dar en escenarios como:

- 1.1 Usuario expulsado de un tenant mid-sesion
- 1.2 Tenant suspendido
- 1.3 Usuario con membership en varios tenants
- 1.4 Downgrade de plan/permisos

La solución para estos casos con agentes de IA interviniendo, consiste en:

- Recortar el TTL para el acces token,
- revalidación server-side en operaciones sensibles (no confiar unicamente en el JWT),
- Establecer una blacklist para que los tokens emitidos de un tenant suspendido dejen de ser válidos
- Refresh token rotation con validacion de membership
- Forzar reissue de token al cambiar de tenant activo (para los casos de multitentant membership, al hacer el cambio de contexto en la UI dispare la llamada al backend para estar conectado a su tenant correcto y evitar el error de desajuste en frontend)
- Contrastar el `tenant_id` contra el JWT, no confiar que venga en el body o en la URL, es una vulnerabilidad de esalacion horizontal si no se hace.
- JWT firmado con un secreto/clave que nunca este expuesta en el cliente ni en el código fuente. (Vault/AWS Secrets)
- Usar RS256 (firma asimétrica) en vez de HS256 (simétrica) si vas a validar el JWT en múltiples servicios/microservicios — así solo el servicio de auth tiene la clave privada, y el resto solo necesita la pública para verificar.
- Incluir también el `user_id` y roles dentro del mismo JWT para no tener que hacer una consulta adicional a BBDD en cada request solo para autorización.

### 2. Autorización y seguridad

Capa especialmente buena para procesos B2B. Presenta dos puntos críticos a resolver:

- 2.1 Doble capa obligatoria: autorizacion a nivel de aplicacion (middleware) y autorización a nivel de BBDD (nuestro RLS)
- 2.2 Los agentes de IA al tener function-calling con acceso a una función que consulta la BBDD debe heredar el mismo contexto de tenant que el request original, no ejecutarse con permisos elevados.

**Buenas prácticas**

- Autorización basada en políticas explícitas (ABAC/RBAC) evaluadas en un punto central, no dispersas en cada controlador.
- Tests de penetración internos específicos: intentar, autenticado como tenant A, acceder a recursos de tenant B por ID directo (ej. `/api/conversaciones/{id}` con un ID que pertenece a otro tenant) — este es el test de aislamiento más básico y el que más suele fallar en auditorías reales.
- Principio de mínimo privilegio también para el propio backend: el usuario de BBDD que usa la aplicación no debería poder hacer BYPASS RLS (en Postgres, el rol `BYPASSRLS` debe reservarse solo para tareas administrativas offline).
- Pueden ser exigidos en una auditoria de seguridad por cualquier cliente antes de firmar el contrato para cumplir con el (ISO 27001, SOC 2)

> **Aspecto a tener en cuenta en cuanto a uso de tokens**
>
> Si implementamos guardrails de IA que validan que el agente no intente acceder a datos fuera de su tenant, eso añade una llamada extra (o un paso de validación) que puede sumar tokens si se hace vía LLM en vez de vía código determinista. Recomendación: esta validación debe ser código determinista (`if tenant_id != contexto → rechazar`), nunca delegada al LLM, porque un LLM puede ser manipulado vía prompt injection para saltarse una instrucción en texto.

### 3. Métricas, Logs, y rate limiting

Toda esta observabilidad debe estar etiquetada y agregada por `tenant_id`, no solo a nivel global, esto nos permite diagnosticar problemas específicos de un cliente, establecer planes de precios basados en eluso real y aislar el impacto de un tenant abusivo son afectar a los demás.

Los problemas que presenta esta sección son logs con datos personales, rate limiting mal implementado. La solucion conlleva una larga explicación que redactaré en `desarrollo_ideas.md`. 

>**Costes en tokens**
>
>Aquí es donde el tracking de tokens por tenant (ya diseñado en el punto anterior de la conversación) se conecta directamente con el rate limiting: la lógica de "tenant X ha superado 100k tokens este mes en plan Basic → bloquear o degradar a modelo más barato" debe evaluarse en tiempo real, antes de hacer la llamada al LLM (para no gastar la llamada y luego rechazar).

### 4. Independencia de la estructura arquitectónica

Sus problemas se centran mas en la migración de uno a otro si no hay un diseño inicial consciente de que esto pueda pasar. No aplica a nuestro caso.
