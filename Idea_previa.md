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

Seguridad: lo analizare en el archivo idea_seguridad.md


