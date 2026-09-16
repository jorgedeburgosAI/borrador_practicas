# Arquitectura Multitenant

## 1. Qué es

Patrón arquitectónico donde una única instancia de aplicación (código, y a menudo infraestructura) sirve a múltiples clientes (*tenants*) de forma aislada lógica o físicamente, cada uno con sus propios datos, configuración y, en algunos casos, personalización.

Existen tres modelos principales:

| Modelo | Descripción |
|---|---|
| **Shared database, shared schema** | Todos los tenants en las mismas tablas, diferenciados por una columna `tenant_id`. |
| **Shared database, separate schema** | Mismo motor de BBDD, un *schema* por tenant. |
| **Separate database per tenant** | Aislamiento total a nivel de instancia de BBDD. |

### Terminología equivalente (framework AWS): Silo / Pool / Bridge

Es un framework más amplio que engloba las tres opciones anteriores y añade la dimensión de *qué recursos aíslas*, no solo la BBDD:

- **Silo**: cada tenant tiene su propia pila de recursos dedicada (BBDD, y a veces también cómputo, red, incluso cuenta cloud entera). El modelo "separate database" es un caso particular de silo a nivel de datos.
- **Pool**: todos los tenants comparten los mismos recursos (el modelo "shared schema" es el pool más extremo).
- **Bridge**: mezcla — por ejemplo, cómputo compartido pero almacenamiento aislado, o al revés. Es el terreno intermedio real donde vive la mayoría de sistemas SaaS maduros.

---

## 2. Ventajas

- Coste operativo bajo (un único despliegue, un único mantenimiento de código).
- Escalado horizontal más simple si el modelo es *shared schema*.
- Actualizaciones y parches de seguridad se aplican una sola vez para todos los tenants.
- Onboarding de nuevos clientes rápido (no hay que desplegar infraestructura nueva).

---

## 3. Inconvenientes

- El aislamiento lógico (*shared schema*) es más frágil que el físico: un fallo en la lógica de filtrado expone datos de otro tenant.
- **Ruido de vecinos** (*noisy neighbor*): un tenant con carga alta puede degradar el rendimiento de otros si no hay *throttling*/*quotas*.
- La personalización profunda por tenant (esquemas de datos distintos, integraciones específicas) es más difícil de sostener que con instancias separadas.
- Un incidente de seguridad o bug de datos afecta potencialmente a todos los tenants a la vez.

---

## 4. Puntos críticos

- **Filtrado de `tenant_id` olvidado en una query**: el fallo más común y más grave. Si un desarrollador olvida el `WHERE tenant_id = ?` en una sola consulta, hay fuga de datos entre clientes.
- **Migraciones de esquema**: en modelo *schema-per-tenant*, aplicar una migración a 200 tenants sin fallos parciales es una operación delicada.
- **Claves de cifrado y secretos por tenant**: si se comparten claves entre tenants, un compromiso afecta a todos.
- **Borrado de datos (derecho al olvido, RGPD)**: en *shared schema*, borrar los datos de un tenant sin tocar los de otro requiere disciplina absoluta en las queries de borrado.

---

## 5. Costes en tokens

No aplica directamente a la arquitectura multitenant en sí (no consume tokens de LLM). Lo relevante aquí es otro tipo de coste: necesitas trackear el consumo de tokens LLM **por tenant** para facturación/rate-limiting, lo cual añade una tabla de métricas (`tenant_id`, `tokens_used`, `timestamp`) y lógica de agregación. Es un requisito de diseño derivado de ser multitenant + IA.

---

## 6. Buenas prácticas

- **Row-Level Security (RLS)** a nivel de motor de BBDD (PostgreSQL lo soporta nativamente) en vez de confiar solo en el filtrado en código de aplicación. Esto cambia las garantías del modelo *shared schema*: en vez de fiarse de que la aplicación filtre por `tenant_id`, es la propia BBDD la que impone el aislamiento a nivel de motor, aunque una query mal escrita se salte el filtro en la capa de app.
- **Middleware** que inyecte automáticamente el `tenant_id` en cada request/query, nunca dejarlo a discreción manual del desarrollador en cada endpoint.
- **Tests automatizados específicos de "cross-tenant leakage"** (intentar acceder a datos de tenant B autenticado como tenant A).
- **Logging y auditoría separados por tenant** para trazabilidad en caso de incidente.
- **Rate limiting y quotas por tenant** desde el diseño inicial, no como parche posterior.

---

## 7. Cuellos de botella

- En *shared schema* con tablas grandes, los **índices sobre `tenant_id`** son obligatorios; sin ellos, cada query escanea datos de todos los tenants. Una técnica complementaria es el **particionado nativo** (`PARTITION BY LIST/HASH (tenant_id)`): cada tenant vive en una partición física distinta aunque lógicamente sea la misma tabla, lo que mejora el rendimiento y permite operaciones como "borrar todo un tenant" de forma eficiente.
- **Conexiones a BBDD**: con el modelo *database per tenant*, el pool de conexiones puede agotarse rápido si hay muchos tenants activos simultáneamente.
- **Migraciones síncronas** sobre muchos schemas/BBDD pueden bloquear el despliegue completo si una falla a mitad.

---

## 8. Errores comunes

- Confiar el aislamiento únicamente a la capa de aplicación sin RLS ni defensa en profundidad.
- No testear explícitamente el aislamiento (asumir que "funciona" porque no ha fallado en desarrollo).
- Mezclar lógica de negocio con lógica de resolución de tenant (el `tenant_id` debería resolverse en un único punto centralizado, no repetido por todo el código).
- Guardar el `tenant_id` en el JWT sin validarlo contra la sesión/BBDD en cada request sensible (permite manipulación del token para suplantar tenant).

---

## 9. Legalidad / GRC

- **RGPD**: cada tenant es, normalmente, el *responsable del tratamiento* (data controller) de sus propios datos de usuario final, y el SaaS es el *encargado del tratamiento* (data processor). Esto exige un **DPA** (Data Processing Agreement) por tenant y capacidad técnica real de exportar/borrar datos de un tenant sin tocar a otros.
- Si algún tenant opera en sector regulado (salud, legal, financiero), puede exigir **aislamiento físico** (BBDD separada) por contrato o normativa sectorial, no solo lógico.
- **Localización de datos**: si hay tenants en distintas jurisdicciones (UE vs. fuera de UE), la arquitectura debe permitir fijar dónde residen físicamente los datos de cada tenant.

---

## 10. Alternativas de stack/arquitectura

- **Shared schema + RLS (PostgreSQL)** — la opción recomendada por defecto: buen equilibrio coste/aislamiento si se implementa bien. *Trade-off*: el aislamiento depende de que el RLS esté correctamente configurado en cada tabla, sin excepciones.
- **Schema-per-tenant (PostgreSQL)** — mejor aislamiento que *shared schema* sin llegar al coste de BBDD separadas. *Trade-off*: la gestión de migraciones se vuelve más compleja a medida que crecen los tenants (cientos de schemas que mantener sincronizados).
- **Database-per-tenant** — máximo aislamiento y el más fácil de justificar ante auditorías de seguridad/RGPD. *Trade-off*: el coste operativo y de infraestructura crece linealmente con el número de tenants; no escala bien para muchos tenants pequeños.
- **Sharding / particionado horizontal** — distinto de "una BBDD por tenant": aquí hay N instancias de BBDD, y cada tenant se asigna a una de ellas (por hash, rango, o tabla de enrutamiento). Cada *shard* aloja a muchos tenants, no uno solo. Da parte del aislamiento de rendimiento del modelo *separate DB* sin el coste operativo de gestionar miles de instancias. BBDD especializadas como **Citus** (extensión de PostgreSQL) implementan esto de forma nativa, particionando por `tenant_id` de forma distribuida — competitivo si se prevé escalar mucho, aunque añade una pieza de infraestructura más que aprender y mantener.
- **Multitenancy a nivel de Kubernetes (namespace/cluster)** — un enfoque más "infra-heavy": aislamiento a nivel de red y cómputo, no solo de datos. El mismo espectro pool/bridge/silo puede aplicarse más allá de la BBDD: **cómputo** (instancia de app compartida vs. dedicada), **red** (VPC compartida vs. VPC por tenant) y, en el caso extremo, **cuenta cloud** entera por tenant (*account-per-tenant*), típico en sectores muy regulados. *Trade-off*: complejidad operativa alta, normalmente solo justificable en SaaS B2B enterprise, no para un proyecto de prácticas.
- **Estrategia híbrida por tier de cliente** — muy común en la práctica: tenants pequeños/free van en *shared schema* (pool), tenants enterprise o con requisitos de compliance van en BBDD dedicada (silo). No es una arquitectura fija, sino una política de asignación que combina las anteriores según el contrato o el volumen del tenant.

> La pregunta que suele mandar no es "¿cuál es la más segura?", sino los trade-offs concretos: coste operativo de mantener N esquemas/instancias, facilidad de que aparezca *noisy neighbor*, complejidad de migraciones, y requisitos de compliance (algunos clientes enterprise exigen aislamiento físico por contrato, no basta con lógico).