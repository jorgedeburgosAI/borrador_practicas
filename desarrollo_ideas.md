Arquitectura Multitenant


1. Qué es

Patrón arquitectónico donde una única instancia de aplicación (código, y a menudo infraestructura) sirve a múltiples clientes ("tenants") de forma aislada lógica o físicamente, cada uno con sus propios datos, configuración y, en algunos casos, personalización. Existen tres modelos principales:

Shared database, shared schema: todos los tenants en las mismas tablas, diferenciados por una columna tenant_id.
Shared database, separate schema: mismo motor de BBDD, un schema por tenant.
Separate database per tenant: aislamiento total a nivel de instancia de BBDD.


2. Ventajas

Coste operativo bajo (un único despliegue, un único mantenimiento de código).
Escalado horizontal más simple si el modelo es "shared schema".
Actualizaciones y parches de seguridad se aplican una sola vez para todos los tenants.
Onboarding de nuevos clientes rápido (no hay que desplegar infraestructura nueva).


3. Inconvenientes

El aislamiento lógico (shared schema) es más frágil que el físico: un fallo en la lógica de filtrado expone datos de otro tenant.
Ruido de vecinos ("noisy neighbor"): un tenant con carga alta puede degradar el rendimiento de otros si no hay throttling/quotas.
Personalización profunda por tenant (esquemas de datos distintos, integraciones específicas) es más difícil de sostener que con instancias separadas.
Un incidente de seguridad o bug de datos afecta potencialmente a todos los tenants a la vez.


4. Puntos críticos

Filtrado de tenant_id olvidado en una query: es el fallo más común y más grave. Si un desarrollador olvida el WHERE tenant_id = ? en una sola consulta, hay fuga de datos entre clientes.
Migraciones de esquema: en modelo schema-per-tenant, aplicar una migración a 200 tenants sin fallos parciales es una operación delicada.
Claves de cifrado y secretos por tenant: si se comparten claves entre tenants, un compromiso afecta a todos.
Borrado de datos (derecho al olvido, RGPD): en shared schema, borrar los datos de un tenant sin tocar los de otro requiere disciplina absoluta en las queries de borrado.


5. Costes en tokens


No aplica directamente a la arquitectura multitenant en sí (no consume tokens de LLM). Lo relevante aquí es otro tipo de coste: necesitas trackear consumo de tokens LLM por tenant para facturación/rate-limiting, lo cual añade una tabla de métricas (tenant_id, tokens_used, timestamp) y lógica de agregación — esto sí es un requisito de diseño derivado de ser multitenant + IA.

6. Buenas prácticas

Row-Level Security (RLS) a nivel de motor de BBDD (PostgreSQL lo soporta nativamente) en vez de confiar solo en el filtrado en código de aplicación — así el aislamiento no depende de que el desarrollador no se olvide de nada.
Middleware que inyecte automáticamente el tenant_id en cada request/query, nunca dejarlo a discreción manual del desarrollador en cada endpoint.
Tests automatizados específicos de "cross-tenant leakage" (intentar acceder a datos de tenant B autenticado como tenant A).
Logging y auditoría separados por tenant para trazabilidad en caso de incidente.
Rate limiting y quotas por tenant desde el diseño inicial, no como parche posterior.

7. Cuellos de botella

En shared schema con tablas grandes, los índices sobre tenant_id son obligatorios; sin ellos, cada query escanea datos de todos los tenants.
Conexiones a BBDD: con modelo "database per tenant", el pool de conexiones puede agotarse rápido si hay muchos tenants activos simultáneamente.
Migraciones síncronas sobre muchos schemas/BBDD pueden bloquear el despliegue completo si una falla a mitad.

8. Errores comunes

Confiar el aislamiento únicamente a la capa de aplicación sin RLS ni defensa en profundidad.
No testear explícitamente el aislamiento (asumir que "funciona" porque no ha fallado en desarrollo).
Mezclar lógica de negocio con lógica de resolución de tenant (el tenant_id debería resolverse en un único punto centralizado, no repetido por todo el código).
Guardar el tenant_id en el JWT sin validarlo contra la sesión/BBDD en cada request sensible (permite manipulación del token para suplantar tenant).

9. Legalidad/GRC

RGPD: cada tenant es, normalmente, el "responsable del tratamiento" (data controller) de sus propios datos de usuario final, y tu SaaS es el "encargado del tratamiento" (data processor). Esto exige un DPA (Data Processing Agreement) por tenant y capacidad técnica real de exportar/borrar datos de un tenant sin tocar a otros.
Si algún tenant opera en sector regulado (salud, legal, financiero), puede exigir aislamiento físico (BBDD separada) por contrato o normativa sectorial, no solo lógico.
Localización de datos: si tienes tenants en distintas jurisdicciones (UE vs fuera de UE), la arquitectura debe permitir fijar dónde residen físicamente los datos de cada tenant.

10. Alternativas de stack/arquitectura

Shared schema + RLS (PostgreSQL) — la opción que sueles ver recomendada por defecto: buen equilibrio coste/aislamiento si se implementa bien. Trade-off: el aislamiento depende de que el RLS esté correctamente configurado en cada tabla, sin excepciones.
Schema-per-tenant (PostgreSQL) — mejor aislamiento que shared schema sin llegar al coste de BBDD separadas. Trade-off: gestión de migraciones se vuelve más compleja a medida que crecen los tenants (cientos de schemas que mantener sincronizados).
Database-per-tenant — máximo aislamiento y el más fácil de justificar ante auditorías de seguridad/RGPD. Trade-off: coste operativo y de infraestructura crece linealmente con el número de tenants; no escala bien para muchos tenants pequeños.
Multitenancy a nivel de Kubernetes namespace/cluster (un enfoque más "infra-heavy") — aislamiento a nivel de red y cómputo, no solo de datos. Interesante si los tenants tienen requisitos de compliance muy dispares entre sí. Trade-off: complejidad operativa alta, normalmente solo justificable en SaaS B2B enterprise, no para un proyecto de prácticas.
BBDD especializadas multitenant nativas (ej. Citus para PostgreSQL, que particiona por tenant_id de forma distribuida) — competitiva si se prevé escalar mucho. Trade-off: añade una pieza de infraestructura más que aprender y mantener.

Modelo Silo / Pool / Bridge (terminología AWS SaaS)

Es un framework más amplio que engloba tus tres opciones y añade la dimensión de qué recursos aíslas, no solo la BBDD:

Silo: cada tenant tiene su propia pila de recursos dedicada (BBDD, y a veces también cómputo, red, hasta cuenta cloud entera). Tu "separate database" es un caso particular de silo a nivel de datos.
Pool: todos los tenants comparten los mismos recursos (tu "shared schema" es el pool más extremo).
Bridge: mezcla — por ejemplo, cómputo compartido pero almacenamiento aislado, o al revés. Es el terreno intermedio real donde vive la mayoría de sistemas SaaS maduros.

Sharding / pool con particionado horizontal

Distinto de "una BBDD por tenant": aquí tienes N instancias de BBDD, y cada tenant se asigna a una de ellas (por hash, rango, o tabla de enrutamiento). Cada shard aloja a muchos tenants, no uno solo. Te da parte del aislamiento de rendimiento del modelo "separate DB" sin el coste operativo de gestionar miles de instancias.

Estrategia híbrida por tier de cliente

Muy común en la práctica: tenants pequeños/free van en shared schema (pool), tenants enterprise o con requisitos de compliance van en BBDD dedicada (silo). No es una arquitectura fija sino una política de asignación que combina las anteriores según el contrato o el volumen del tenant.

Aislamiento a nivel de fila dentro de shared schema

Row-Level Security (por ejemplo en Postgres) no es una arquitectura nueva, pero cambia las garantías del modelo "shared schema": en vez de confiar solo en que la aplicación filtre por tenant_id, la propia BBDD impone el aislamiento a nivel de motor, aunque una query mal escrita se salte el filtro en la capa de app.

Particionado nativo por tenant_id

Dentro de shared schema, usar particionado de tabla (PARTITION BY LIST/HASH (tenant_id)) para que cada tenant viva en una partición física distinta aunque lógicamente sea la misma tabla. Mejora rendimiento y permite operaciones como "borra todo un tenant" de forma eficiente.

Aislamiento más allá de la base de datos

La multitenencia no acaba en la capa de datos — se puede aplicar el mismo espectro (pool/bridge/silo) a:

Cómputo: instancia de app compartida vs. dedicada por tenant.
Red: VPC compartida vs. VPC por tenant.
Cuenta cloud: el caso extremo, una cuenta AWS/GCP entera por tenant (account-per-tenant), típico en sectores muy regulados.

la pregunta que suele mandar no es "¿cuál es la más segura?" sino trade-offs concretos: coste operativo de mantener N esquemas/instancias, facilidad de hacer noisy neighbor (un tenant satura recursos de otro), complejidad de migraciones, y requisitos de compliance (algunos clientes enterprise exigen aislamiento físico por contrato, no basta con lógico).


Ideas de desarrollo personales:

He detectado 3 problemas fundamentales:

-La seguridad entre tenants

-Escalabilidad y diseño de tablas de cada tenant

-Borrado o modificacion de tablas que puedan afectar a otros tenants

IMPORTANTE TESTEAR 

Se deberia de analizar si el cliente a entrar en dicha arquitectura esta al mismo nivel vertical que los demás ya que puede suponer una degradacion de rendimientos en los demas tenant. Posible solucion crear una solucion especifica para el cliente o si se prevee clientes con mismas necesidades crear otro multitenant(estudiar viabilidad para rdto optimo sin latencais), de dicha manera nos quitamos los incovenientes de escalabilidad vertical. Para ello deberiamos visualizar el nivel previsto de exigencias y escoger un stack diferente según las necesidades.



Opciones en multitenant

Una vez analizados los cientes deberiamos establecer la arquitectura de la bbdd a usar: 
Pool = shared schema
Bridge = shared DB, separate schema
Silo = separate database per tenant

Sharding (pool particionado): Es N instancias, y cada tenant se asigna a una vía hash/rango/tabla de enrutamiento. No renta debido a:

Cuando el número de tenants crece tanto que una sola instancia empieza a tener problemas de rendimiento reales (no anticipados, medidos).
Cuando aparece un segmento de clientes con exigencia alta mezclado con el resto, y ahí el sharding (o mejor, una estrategia híbrida por tier) sí resuelve algo concreto.
Cuando hay requisitos de compliance que obligan a cierto aislamiento, algo que no mencionas que sea el caso.

RLS; Row-Level Security (RLS) en shared schema

Ventajas:

Aislamiento a nivel de motor de BBDD, no solo de aplicación: aunque un desarrollador olvide el WHERE tenant_id = ? en una query, la BBDD igualmente filtra.
Reduce drásticamente la superficie de bugs de fuga de datos entre tenants, que es uno de los fallos más graves y comunes en shared schema.
No requiere cambiar el modelo de datos ni el despliegue — se añade sobre lo que ya tienes.

Desventajas:

Overhead de rendimiento: cada query pasa por una política de seguridad adicional; en tablas grandes con políticas complejas puede notarse.
Complejidad de configuración y mantenimiento: cada tabla nueva necesita su política, y es fácil olvidarse de aplicarla en una tabla nueva (aunque menos grave que olvidarlo en la app, sigue siendo un punto de fallo).
Debugging más difícil: cuando una query "no devuelve lo esperado", a veces es la política RLS actuando de forma no obvia, y hay que tenerlo en cuenta al depurar.
No es soportado igual en todos los motores (Postgres lo tiene muy maduro; MySQL no tiene RLS nativo, por ejemplo — habría que emular con vistas).
 Particionado físico por tenant_id + RLS — ventajas y desventajas combinadas

Ventajas:

Son complementarios, no redundantes: la partición te da rendimiento (partition pruning) y borrado rápido; RLS te da seguridad (el motor impone el filtro aunque la app falle). Cubres dos problemas distintos con un solo diseño.
Offboarding de un tenant (GDPR, baja de cliente) pasa de DELETE lento a DROP PARTITION casi instantáneo, con la garantía extra de que RLS sigue protegiendo mientras tanto.
Si tienes ambos, un bug de aplicación que olvide el WHERE tenant_id no filtra datos ajenos (por RLS), aunque pierdas el beneficio de pruning en esa query concreta.

Desventajas:

Doble mantenimiento: cada tabla nueva necesita partición y política RLS definidas; te puedes olvidar de una de las dos, y es fácil que la partición avance sin que la política RLS se actualice igual.
El pruning solo se activa si la query incluye el filtro de tenant_id explícitamente — si no, escaneas todas las particiones igual, y ahí sigue pesando el overhead de evaluar la política RLS en cada una.
Con muchos tenants, partición por tenant individual se vuelve inmanejable (miles de particiones = overhead de metadatos); tendrías que agrupar por hash/rango, perdiendo parte del beneficio de "borrado instantáneo por tenant".
Migraciones de esquema más pesadas: un cambio de columna se aplica a cada partición, y hay que verificar que las políticas RLS lo sigan cubriendo.

¿Rentable con clientes de exigencia media-baja?

Aquí conviene separar los dos mecanismos, porque no valen lo mismo:

RLS solo: sí compensa, incluso con exigencia baja. No es una feature de rendimiento ligada al volumen — es una red de seguridad barata de implementar sobre shared schema, y el riesgo que mitiga (fuga de datos entre tenants por bug de app) no depende de cuánto factura o exige el cliente. Es coste bajo, beneficio constante.
Particionado físico solo: no compensa. Sus ventajas (pruning, borrado rápido) solo se notan cuando el volumen de datos por tabla es alto o el offboarding de tenants es frecuente. Con exigencia media-baja, una tabla sin particionar aguanta de sobra, y te ahorras la complejidad de mantener particiones sincronizadas con el esquema.

Punto crítico real: si usas un pool de conexiones (PgBouncer, por ejemplo), SET a nivel de sesión puede "filtrarse" entre requests si el pool reutiliza conexiones sin resetear la variable. Esto es un error de producción documentado en varios post-mortems de SaaS reales — hay que resetear explícitamente la variable al devolver la conexión al pool, o usar SET LOCAL dentro de una transacción para que el scope sea por transacción, no por sesión.

Resolución de tenant: dónde y cómo

El patrón correcto es resolver el tenant una sola vez, en el borde de la aplicación (middleware de autenticación), nunca dentro de la lógica de negocio:

El JWT lleva el tenant_id firmado por el servidor (nunca generado o modificable por el cliente).
El middleware valida el JWT, extrae tenant_id, y lo inyecta en el contexto de la request.
La capa de acceso a datos (ORM/query builder) lee ese contexto automáticamente — el desarrollador de negocio nunca escribe tenant_id a mano en una query.

Con ORMs como Prisma o SQLAlchemy esto se implementa con hooks/interceptors globales, no repitiendo el filtro en cada repositorio.


Row-Level Security como mecanismo, no como capa
Dentro de shared schema, la pregunta "¿quién impone el aislamiento?" es ortogonal a dónde viven los datos. Sin RLS, el aislamiento depende de que la app nunca olvide el WHERE tenant_id = ?. Con RLS, el motor de BBDD lo impone aunque la query de la capa de aplicación esté mal escrita. Mismo modelo de datos (shared schema), garantía distinta.

TEMA DE SEGURIDAD: 

4.1 Contexto del tenant en las peticiones (JWT firmado)
Qué es

El tenant_id viaja dentro del payload del JWT, firmado criptográficamente por el servidor de autenticación, en vez de pasarse como parámetro de URL (?tenant=acme) o header manipulable por el cliente.

Ventajas
El cliente no puede alterar su propio tenant_id porque la firma del JWT lo invalidaría.
Un único punto de verdad para la identidad + contexto de tenant, en cada request.
Compatible con arquitecturas stateless (no requiere consultar sesión en servidor en cada request si el JWT es autocontenido).
Inconvenientes
Si el JWT tiene TTL largo y el usuario cambia de tenant (multi-tenant membership, ej. un consultor que trabaja para varios clientes), el token queda desactualizado hasta que expira o se refresca.
Revocación complicada: si un tenant es suspendido/baneado, los JWT ya emitidos siguen siendo válidos hasta que expiran, salvo que implementes una blacklist.
Puntos críticos
Nunca confiar en un tenant_id que venga en el body o en la URL sin contrastarlo contra el JWT. Un endpoint que acepta tenant_id como parámetro de request body y no lo valida contra el del token es una vulnerabilidad de escalación horizontal (IDOR).
El JWT debe firmarse con un secreto/clave que nunca esté expuesta en el cliente ni en el código fuente (usar variables de entorno o un servicio de secrets como Vault/AWS Secrets Manager).
Costes en tokens

No aplica (esto es JWT de autenticación, no tokens de LLM — cuidado con no confundir terminología en tu memoria de TFG, un evaluador técnico notará si mezclas ambos conceptos).

Buenas prácticas
Usar RS256 (firma asimétrica) en vez de HS256 (simétrica) si vas a validar el JWT en múltiples servicios/microservicios — así solo el servicio de auth tiene la clave privada, y el resto solo necesita la pública para verificar.
TTL corto (15-30 min) para el access token + refresh token de vida más larga, para poder revocar/actualizar el tenant_id con frecuencia razonable.
Incluir también el user_id y roles dentro del mismo JWT para no tener que hacer una consulta adicional a BBDD en cada request solo para autorización.
Cuellos de botella
Verificación de firma RS256 es más costosa computacionalmente que HS256; en sistemas de muy alto volumen de requests esto se nota, aunque para un SaaS de tamaño medio no es relevante.
Errores comunes
Meter datos sensibles del tenant (no solo el ID) dentro del JWT — el payload de un JWT es solo Base64, no está cifrado, cualquiera puede leerlo.
Olvidar invalidar tokens tras cambio de plan/suspensión del tenant.
Legalidad/GRC
Si el JWT contiene datos personales identificables además del tenant_id, esos datos "viajan" en cada request y quedan potencialmente en logs de proxies/CDN — revisar qué se loguea downstream.
4.2 Autorización y seguridad (aislamiento estricto entre tenants)
Qué es

El conjunto de políticas (a nivel de aplicación y/o de BBDD) que garantizan que ninguna petición, bajo ningún flujo, puede devolver o modificar datos de un tenant distinto al autenticado.

Ventajas

Es la garantía central que hace viable un SaaS multitenant frente a clientes que preguntarán explícitamente por esto en el proceso de compra (especialmente en B2B).

Inconvenientes

Implementarlo correctamente implica defensa en capas (defense in depth), lo cual añade complejidad y superficie de testing respecto a una app single-tenant.

Puntos críticos
Doble capa obligatoria: autorización a nivel de aplicación (middleware) + autorización a nivel de BBDD (RLS). Confiar solo en la primera es el error que más incidentes reales ha causado en SaaS multitenant — un solo endpoint nuevo que se salte el middleware (por ejemplo, un endpoint de "admin" o "soporte interno" añadido con prisas) expone datos de todos los tenants.
Los agentes de IA de tu proyecto son un vector de riesgo adicional aquí: si un agente tiene function-calling con acceso a una función que consulta BBDD, esa función debe heredar el mismo contexto de tenant que el request original, no ejecutarse con permisos elevados "porque es el LLM el que lo pide".
Costes en tokens

Relevante indirectamente: si implementas guardrails de IA que validan que el agente no intente acceder a datos fuera de su tenant, eso añade una llamada extra (o un paso de validación) que puede sumar tokens si se hace vía LLM en vez de vía código determinista. Recomendación: esta validación debe ser código determinista (if tenant_id != contexto → rechazar), nunca delegada al LLM, porque un LLM puede ser manipulado vía prompt injection para saltarse una instrucción en texto.

Buenas prácticas
Autorización basada en políticas explícitas (ABAC/RBAC) evaluadas en un punto central, no dispersas en cada controlador.
Tests de penetración internos específicos: intentar, autenticado como tenant A, acceder a recursos de tenant B por ID directo (ej. /api/conversaciones/{id} con un ID que pertenece a otro tenant) — este es el test de aislamiento más básico y el que más suele fallar en auditorías reales.
Principio de mínimo privilegio también para el propio backend: el usuario de BBDD que usa la aplicación no debería poder hacer BYPASS RLS (en Postgres, el rol BYPASSRLS debe reservarse solo para tareas administrativas offline).
Cuellos de botella

Verificaciones de autorización añaden latencia por request si se hacen contra servicios externos (ej. políticas evaluadas vía un servicio tipo OPA/Open Policy Agent remoto) — normalmente asumible, pero hay que medirlo bajo carga.

Errores comunes
Autorización "por omisión permitida" (si no se especifica lo contrario, se permite) en vez de "por omisión denegada" (deny by default).
Probar el aislamiento solo manualmente en desarrollo y no como suite automatizada de regresión.
Legalidad/GRC

Este punto es el núcleo de cualquier auditoría de seguridad (ISO 27001, SOC 2) que un cliente B2B pueda exigir antes de firmar contrato. Documentar el modelo de autorización y sus tests es, en la práctica, parte de la evidencia de compliance.

4.3 Métricas, logs y rate limiting segmentados por tenant
Qué es

Toda observabilidad (logs de aplicación, métricas de uso, límites de tasa de peticiones) debe estar etiquetada y agregada por tenant_id, no solo a nivel global de sistema.

Ventajas
Permite diagnosticar problemas específicos de un cliente sin "ruido" del resto.
Habilita planes de precios basados en uso real (rate limiting por tenant = básico para modelo freemium/tiers).
Aísla el impacto de un tenant abusivo (accidental o malicioso) sin afectar al resto — mitiga el problema de "noisy neighbor" mencionado antes.
Inconvenientes

Añade dimensionalidad a toda tu infraestructura de observabilidad: más cardinalidad en métricas (un tenant_id por cada tenant activo) puede encarecer herramientas de monitoring que cobran por cardinalidad (ej. Datadog, Prometheus con muchas etiquetas).

Puntos críticos
Logs con datos personales: si logueas payloads completos de requests que incluyen datos de usuario final, y esos logs no están segmentados con control de acceso por tenant, cualquier ingeniero con acceso a logs puede ver datos de todos los clientes — esto es un problema de GRC, no solo técnico.
Rate limiting mal implementado a nivel global (en vez de por tenant) permite que un tenant "ruidoso" agote la cuota de toda la plataforma, afectando a clientes que pagan más.
Costes en tokens

Aquí es donde el tracking de tokens por tenant (ya diseñado en el punto anterior de la conversación) se conecta directamente con el rate limiting: la lógica de "tenant X ha superado 100k tokens este mes en plan Basic → bloquear o degradar a modelo más barato" debe evaluarse en tiempo real, antes de hacer la llamada al LLM (para no gastar la llamada y luego rechazar).

Buenas prácticas
Usar tenant_id como label/tag obligatorio en toda métrica desde el diseño del sistema de logging (structured logging con JSON, no logs de texto plano).
Rate limiting con algoritmo token bucket o sliding window por tenant, implementado en una capa rápida (Redis) para no penalizar la latencia del request.
Alertas automáticas por tenant que se acerque a su límite, no solo bloqueo silencioso al llegar al 100%.
Cuellos de botella

Consultar límites de rate limiting en Redis en cada request añade una llamada de red extra; en sistemas de alto volumen esto se optimiza con lógica de "leaky bucket" local con sincronización periódica, en vez de consulta síncrona en cada petición.

Errores comunes
Rate limiting aplicado solo en el API Gateway global, sin diferenciar planes de tenant (todos comparten el mismo límite).
Logs sin rotación/retención definida por tenant — algunos clientes (según contrato/RGPD) pueden exigir borrado de sus logs tras cierto periodo, y si están mezclados con los de otros tenants, borrar selectivamente es complejo.
Legalidad/GRC
RGPD: los logs que contienen datos personales están sujetos a las mismas obligaciones de minimización y retención limitada que cualquier otro dato personal. Segmentar por tenant facilita cumplir "derecho al olvido" de forma quirúrgica.
Rate limiting por tenant también es relevante para SLA contractuales (si prometes cierto throughput a un cliente Enterprise, debes poder demostrarlo con métricas segmentadas).
4.4 Independencia de la estructura arquitectónica (monolito vs microservicios vs hexagonal)
Qué es

El multitenant es una decisión ortogonal al estilo arquitectónico general de la aplicación: se puede implementar tanto en un monolito como en microservicios o en una arquitectura limpia/hexagonal, porque el aislamiento de tenant es una preocupación transversal (cross-cutting concern), no una decisión de descomposición de servicios.

Ventajas
Te da libertad para elegir la arquitectura según otros criterios (complejidad del equipo, tamaño del proyecto) sin que la multitenencia fuerce la decisión.
Para un proyecto de prácticas/TFG, esto es una ventaja práctica directa: puedes justificar un monolito modular (más simple de defender y de terminar a tiempo) sin que eso reste rigor a la solución multitenant.
Inconvenientes
En microservicios, el contexto de tenant debe propagarse entre servicios (vía JWT, headers, o contexto distribuido tipo OpenTelemetry baggage), lo cual añade complejidad de "plumbing" que en un monolito es trivial (una sola inyección de contexto).
En arquitectura hexagonal, hay que decidir en qué capa vive la resolución del tenant (normalmente en el adaptador de entrada, nunca en el dominio) para no contaminar la lógica de negocio con detalles de infraestructura.
Puntos críticos

Si migras de monolito a microservicios más adelante, el mecanismo de propagación de tenant_id entre servicios es uno de los puntos que más fricción genera si no se diseñó pensando en ello desde el principio (aunque el punto de partida sea un monolito).

Costes en tokens

No aplica directamente — es una decisión estructural, no de consumo de IA.

Buenas prácticas
Para un proyecto de tu envergadura (prácticas/TFG), monolito modular con arquitectura hexagonal/limpia interna es la opción más defendible: separas claramente dominio, aplicación e infraestructura, y el tenant se resuelve en el adaptador de entrada (ej. middleware HTTP), sin tocar el núcleo de negocio.
Si en algún momento documentas "cómo escalaría esto a microservicios", demuestra que entiendes la propagación de contexto — es un punto que suma mucho en una defensa de TFG o entrevista técnica, sin que tengas que implementarlo realmente.
Cuellos de botella

No aplica de forma directa a esta decisión en sí (los cuellos de botella dependen del modelo de datos elegido, ya cubierto en 4.1-4.3).

Errores comunes

Sobre-diseñar microservicios para un proyecto de este alcance "porque suena más profesional" — en la práctica, un evaluador técnico valorará más un monolito bien estructurado (con separación de capas clara) que microservicios mal delimitados por falta de experiencia previa gestionando sistemas distribuidos.

Legalidad/GRC

No aplica directamente a la elección arquitectónica en sí, salvo que microservicios distribuidos en distintas regiones/nubes puedan complicar la localización de datos exigida por RGPD si no se controla explícitamente dónde corre cada servicio.

Alternativas de stack/arquitectura (aplicable a los 4 puntos en conjunto)
Necesidad	Opción estándar	Alternativa competitiva	Trade-off
JWT + gestión de identidad	Auth propio (ej. con jsonwebtoken en Node o PyJWT)	Auth0 / Clerk / Supabase Auth (gestionado)	Gestionado = menos control pero cero mantenimiento de seguridad de auth; propio = más control pero tú asumes el riesgo de bugs de seguridad en algo tan crítico
Autorización por políticas	Lógica ad-hoc en middleware	Open Policy Agent (OPA) o Cerbos	Herramientas dedicadas centralizan políticas y son auditables, pero añaden una pieza de infraestructura más a aprender/mantener — para tu alcance, probablemente sobredimensionado
Rate limiting	Implementación propia con Redis	API Gateway con rate limiting nativo (Kong, AWS API Gateway)	El gateway resuelve esto "gratis" a nivel de infraestructura, pero añade una capa más y coste si es cloud gestionado
Logs/observabilidad multitenant	Stack propio (Winston/Pino + Grafana Loki)	Datadog / New Relic (gestionado, con tagging nativo)	Gestionado es más rápido de implementar bien pero tiene coste recurrente que escala con volumen — para un TFG, el stack propio con Loki es más defendible económicamente y técnicamente



Solución a los puntos críticos: logs con PII y rate limiting global

Voy a resolver cada uno con implementación concreta, conectándolo con lo que ya tienes diseñado (JWT con tenant_id, middleware de resolución de tenant, tabla tenant_usage).

Problema 1: Logs con datos personales sin segmentación
La solución no es "no loguear" — es loguear con estructura y control de acceso desde el diseño

Paso 1 — Redacción/masking antes de que el log toque disco

Nunca loguees el payload crudo de la request. Implementa una capa de sanitización en el propio logger:

javascript
function sanitizeForLog(payload) {
  const camposSensibles = ['email', 'nombre', 'telefono', 'texto_transcrito', 'audio_url'];
  const limpio = { ...payload };
  for (const campo of camposSensibles) {
    if (limpio[campo]) limpio[campo] = '[REDACTED]';
  }
  return limpio;
}

En tu caso concreto (speech-to-text), el texto transcrito de la voz del usuario es dato personal por definición — nunca debe aparecer en logs de debug, ni siquiera en desarrollo, porque es fácil que ese hábito se filtre a producción.

Paso 2 — tenant_id como campo obligatorio, logging estructurado (JSON), nunca texto plano

javascript
logger.info({
  tenant_id: contexto.tenant_id,
  user_id: contexto.user_id,
  event: 'llm_request',
  tokens_used: 342,
  // nunca: texto_completo, audio_data, email_usuario
});

Paso 3 — Aislamiento real a nivel de almacenamiento de logs, no solo a nivel de campo

Aquí está la solución arquitectónica que resuelve el problema de raíz: usa una herramienta de logs con multitenancy nativo, no un único índice compartido filtrado por campo.

Grafana Loki tiene multitenancy nativo real (header X-Scope-OrgID): cada tenant obtiene su propio stream de logs, físicamente separado a nivel de almacenamiento e índice. Esto significa que el control de acceso no depende de que nadie olvide un WHERE tenant_id = ? al consultar logs — es estructural, igual que el RLS que ya definimos para la BBDD.

# Al enviar logs desde tu app
X-Scope-OrgID: <tenant_id>

Con esto, un ingeniero de soporte que solo tiene credenciales para el tenant A no puede consultar logs del tenant B, ni por error ni por curiosidad — el propio sistema de logs lo impide, igual que RLS impide leer filas de otro tenant en Postgres.

Paso 4 — Retención y borrado selectivo (resuelve el error común que señalaste)

Con streams separados por tenant en Loki, borrar los datos de un tenant específico (derecho al olvido RGPD) es borrar su stream completo — no una operación de filtrado sobre datos mezclados. Configura retención por tenant vía políticas (algunos tenants Enterprise pueden contractualmente exigir 90 días, otros 30):

yaml
# loki config, retención por tenant vía overrides
overrides:
  tenant_a:
    retention_period: 2160h  # 90 días
  tenant_b:
    retention_period: 720h   # 30 días
Control de acceso adicional (defensa en capas, igual que en autorización)
RBAC en Grafana: los API keys/tokens de consulta se emiten scoped a un OrgID concreto, así que aunque alguien tenga acceso a Grafana, solo ve lo que su rol permite.
Auditoría de quién consulta logs de qué tenant — un log de "quién accedió a los logs" (meta-logging), relevante para demostrar compliance ante auditoría.
Problema 2: Rate limiting global en vez de por tenant
La solución: dos niveles simultáneos, no uno u otro

Necesitas ambos, no reemplazar el global por el de tenant:

Límite global de sistema → protege tu infraestructura de caerse (protección técnica).
Límite por tenant → protege el reparto justo entre clientes (protección de negocio/SLA).
Implementación con Redis (token bucket por tenant)
javascript
async function checkRateLimit(tenant_id, plan) {
  const key = `ratelimit:${tenant_id}`;
  const limite = LIMITES_POR_PLAN[plan]; // ej. 100 req/min en Basic
  
  const actual = await redis.incr(key);
  if (actual === 1) {
    await redis.expire(key, 60); // ventana de 60s
  }
  
  if (actual > limite) {
    throw new RateLimitExceeded(tenant_id, limite);
  }
}

Para que sea atómico bajo concurrencia (evitar condición de carrera donde dos requests simultáneas pasan el check antes de que se actualice el contador), usa un script Lua ejecutado atómicamente en Redis en vez de INCR+expire por separado:

lua
-- rate_limit.lua
local actual = redis.call('INCR', KEYS[1])
if actual == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if actual > tonumber(ARGV[1]) then
  return 0
end
return 1
Conexión directa con el coste en tokens de LLM (lo que ya diseñamos en tenant_usage)

Aquí resuelves el punto crítico que señalaste sobre "verificar antes de gastar la llamada": el check debe ser de dos fases, no solo al final:

javascript
async function llamarLLM(tenant_id, plan, promptEstimado) {
  // FASE 1: pre-check antes de gastar dinero en la llamada
  const usoActual = await getUsoMensual(tenant_id);
  const limiteTokens = LIMITE_TOKENS_POR_PLAN[plan];
  
  if (usoActual + promptEstimado > limiteTokens) {
    // degradar a modelo más barato o bloquear, ANTES de llamar
    return degradarOBloquear(tenant_id, plan);
  }
  
  const respuesta = await llamarAPILLM(promptEstimado);
  
  // FASE 2: registrar consumo real (puede diferir de la estimación)
  await registrarUso(tenant_id, respuesta.tokens_input, respuesta.tokens_output);
  
  return respuesta;
}

Punto crítico que debes resolver aquí: la estimación de tokens de entrada la puedes calcular antes (con un tokenizer local, ej. tiktoken), pero los tokens de salida no se conocen hasta que la respuesta llega. Por eso el pre-check debe ser conservador (usar el máximo de max_tokens configurado como estimación de salida), no optimista.

Cuellos de botella — solución al problema de latencia por consulta síncrona a Redis

Si cada request hace una llamada de red a Redis solo para el rate-limit check, añades latencia innecesaria en el camino crítico. Solución: local token bucket con sincronización periódica:

javascript
// Cache local en memoria del proceso, sincronizado cada N segundos
class LocalBucketCache {
  constructor() {
    this.buckets = new Map(); // tenant_id -> tokens restantes localmente
    setInterval(() => this.sincronizarConRedis(), 5000);
  }
  
  consumirLocal(tenant_id) {
    // decisión rápida en memoria, sin red
    const bucket = this.buckets.get(tenant_id);
    return bucket > 0;
  }
}

Esto es un trade-off consciente: ganas latencia, pierdes precisión exacta (puede haber un pequeño margen de sobre-consumo entre sincronizaciones) — aceptable para rate limiting de uso general, no aceptable para el control de gasto de tokens LLM si el margen de error tiene impacto económico directo (ahí mejor mantener la consulta síncrona, porque el coste de una llamada extra a Redis es mínimo comparado con el coste de una llamada no controlada al LLM).

Middleware único que centraliza todo (conecta con el diseño de autorización del punto 4.2)

Igual que con tenant_id, resuelve esto en un único punto, no disperso:

javascript
app.use(async (req, res, next) => {
  const { tenant_id, plan } = req.contexto; // ya resuelto por el middleware de auth
  const permitido = await checkRateLimit(tenant_id, plan);
  if (!permitido) return res.status(429).json({ error: 'rate_limit_exceeded' });
  next();
});
Alertas (resuelve "no solo bloqueo silencioso")
javascript
async function verificarYAlertar(tenant_id, usoActual, limite) {
  const porcentaje = usoActual / limite;
  if (porcentaje >= 0.8 && porcentaje < 0.85) {
    await enviarAlerta(tenant_id, 'Has usado el 80% de tu cuota mensual');
  }
}
Resumen — cómo se conecta todo
Pieza ya diseñada	Cómo resuelve el punto crítico
JWT con tenant_id firmado (4.1)	Fuente de verdad para saber a qué tenant aplicar el límite/log
Middleware centralizado de resolución de tenant	Punto único donde inyectar tanto el check de rate limit como el tagging de logs
Tabla tenant_usage	Base de datos para el pre-check de tokens antes de llamar al LLM
RLS en Postgres (arquitectura multitenant)	Mismo principio aplicado ahora a logs (Loki multitenancy) — aislamiento estructural, no solo por disciplina de código