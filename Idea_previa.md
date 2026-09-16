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

Row-Level Security como mecanismo, no como capa
Dentro de shared schema, la pregunta "¿quién impone el aislamiento?" es ortogonal a dónde viven los datos. Sin RLS, el aislamiento depende de que la app nunca olvide el WHERE tenant_id = ?. Con RLS, el motor de BBDD lo impone aunque la query de la capa de aplicación esté mal escrita. Mismo modelo de datos (shared schema), garantía distinta.

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

Seguridad: lo analizare en el archivo idea_seguridad.md






COSAS EN LIMPIO:

Tras analizar como se podria estructurar la bbdd según las diferentes exigencias de los clientes, llego a la conclusión en nuestro caso que lo mas conveniente seria Shared schema (pool) + Row-Level Security, sin particionado ni sharding. ¿Porque? El sharding nos cubre de problemas que en principio
no necesitamos cubrir debido a las necesidades de nuestro caso, si aceptaramos una escalabilidad de clientes dentro de este multitenant estudiariamos la opción mas viable para una migración, si no se diseña algo personal u otro multitenant, de la misma manera para el particionado físico, ya que esta resuuelve problemas para el borrado rapido cuando las tablas son demasiado grandes, por lo que el delete masivo es un cuello de botella real, ademas que el mantenimiento constante no lo hace práctico debidoal consumo de tiempo para nuestra situación, no es óptimo, particionar por tenant_id es una optimización de escala y de operaciones (borrado rápido, rendimiento en tablas grandes) que no aporta seguridad, y con exigencia media-baja el problema que resuelve todavía no existe — así que solo queda el coste de mantenerla, sin el beneficio. Para el caso de extender el eje a cómputo, red y cuenta cloud es una opción muy interesante pero lo descarto por la sencilla razón de nuestro target (la complejidad de explicar y auditar va a cada capa y para casos reales a los que apuntamos no tiene sentido dicha caer en dicha complejidad, ya que complicariamos mucho la venta del SaaS y perderiamos potenciales clientes), tampoco se benefician los problemas que resuelve ya que aisla cómputos para no saturar la cpu/memoria cosa que no nos va a pasar al igual que el aislamiento por red ya que es altamente burocratico (sector financiero, salud y defensa) como con cuenta cloud, que normamelmente lo piden clientes de auditorias.El despliegue serai muy grande porque dependeria de N desiciones independientes cada uno con su propio coste, además de que al requerir tanta configuración se puede caer en la incosistencia, en deefinitica contradice nuestro argumento de coste-beneficio.

RLS: