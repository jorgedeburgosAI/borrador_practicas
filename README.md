# 🎙️ Plataforma SaaS de Agentes de Voz con IA — Visión de Producto y Arquitectura

## 1. Por qué este proyecto: la oportunidad de negocio

Atender llamadas sigue siendo, para la mayoría de empresas, uno de los procesos más caros y menos escalables que existen: cada llamada nueva requiere una persona más, un horario más que cubrir, una curva de formación que repetir. Al mismo tiempo, el cliente que llama espera respuesta inmediata, coherente y disponible a cualquier hora — algo que un equipo humano, por bueno que sea, difícilmente sostiene de forma lineal a medida que crece el volumen. Ya que es importante cuidar la conciliación de nuestros trabajadores y esta es una herramienta ideal con innumerables ventajas.

La convergencia de tres tecnologías (reconocimiento de voz en streaming, modelos de lenguaje capaces de razonar con contexto, y síntesis de voz casi indistinguible de la humana) abre una ventana de oportunidad: automatizar la conversación telefónica sin sacrificar naturalidad. Este proyecto nace para capturar esa ventana, no como un experimento técnico aislado, sino como un producto SaaS pensado para venderse a organizaciones que hoy dependen de call centers costosos o de líneas de atención que no dan abasto — soporte al cliente, gestión de reservas, prospección comercial.

**La apuesta de negocio, en una frase:** convertir un coste operativo que crece con cada cliente nuevo (personal humano) en un coste de infraestructura que escala con el uso (cómputo e IA), sin que el usuario final note la diferencia en calidad de la conversación.

---

## 2. Decisión estratégica de fondo: por qué multi-tenant desde el día uno

Construir una instancia aislada por cada cliente sería más simple a corto plazo, pero condena el negocio a un coste operativo que crece al mismo ritmo que la cartera de clientes — justo lo contrario de lo que se busca vender. Por eso la plataforma se diseña **multi-tenant desde el primer commit**, no como una migración futura:

- **Aislamiento de datos vía `tenant_id` + Row Level Security (RLS):** cada cliente vive lógicamente aislado dentro de la misma infraestructura, lo que mantiene el coste operativo por debajo del de una arquitectura de instancias dedicadas, sin renunciar a la garantía de que un cliente jamás pueda ver datos de otro.
- **Configuración independiente por cliente:** personalidad del agente, base de conocimiento, voz, números de teléfono y reglas de negocio se gestionan por *tenant* — esto es lo que permite vender el mismo producto a un despacho de abogados y a una clínica dental sin tocar una línea de código por cliente nuevo.
- **RBAC (control de acceso basado en roles):** necesario en cuanto el cliente deja de ser una sola persona y pasa a ser una organización con distintos perfiles (administrador, agente, solo lectura de métricas).

Esta decisión no es únicamente técnica: es la que determina si el negocio puede escalar con márgenes de software o si se queda atrapado en márgenes de servicio.

Es la decisión que nos permite escalar la cartera de clientes dentro de unas necesidades similares pudiendo ofrecerles un servicio excelente, rápido de implementar además de poder asegurar la seguridad y consistecia de sus datos.

---

## 3. El pipeline conversacional: diseñado para que la latencia no delate a la máquina

El umbral psicológico donde una conversación deja de sentirse "natural" y empieza a sentirse "con un bot" está, aproximadamente, en el segundo de silencio. Todo el diseño del pipeline gira en torno a mantenerse por debajo de ese umbral:

```text
[El usuario habla]
        │
        ▼
1. Voz → Texto (streaming)   ── transcripción incremental, sin esperar a que la persona termine de hablar
        │
        ▼
2. Motor de razonamiento     ── el LLM interpreta intención, recupera contexto/RAG y decide la respuesta
        │
        ▼
3. Texto → Voz (síntesis)    ── generación de audio con entonación natural, lista para reproducirse
        │
        ▼
[El usuario escucha la respuesta]
```

Cada eslabón de esta cadena se apoya en un proveedor especializado en vez de construirse desde cero — y esa es, en sí misma, una decisión de negocio: el tiempo de ingeniería que costaría igualar el rendimiento de estos proveedores es tiempo que no se está invirtiendo en lo que realmente diferencia el producto frente a la competencia.

---

## 4. Stack tecnológico: qué se eligió y por qué tiene sentido para el negocio, no solo para el código

| Capa | Tecnología | La razón de negocio detrás de la elección |
| :--- | :--- | :--- |
| **Orquestación de voz IA** | Vapi | Delegar la parte más difícil de optimizar (latencia, interrupciones, manejo de turnos) en un proveedor especializado acelera el *time-to-market* meses frente a construir un pipeline de voz propio desde cero. |
| **Telefonía** | Twilio | Cobertura global de numeración y fiabilidad probada a escala — imprescindible si la estrategia comercial contempla vender fuera de un único país sin renegociar infraestructura cada vez. |
| **Backend** | Python + FastAPI | Ecosistema maduro en IA/LLM y soporte asíncrono nativo, necesario para orquestar webhooks y sesiones en tiempo real sin que el backend se convierta en el cuello de botella de latencia. |
| **Datos y autenticación** | Supabase (PostgreSQL) | Auth, base de datos relacional y RLS multi-tenant ya resueltos de fábrica — reduce el equipo de infraestructura necesario en las fases tempranas, cuando cada mes de desarrollo cuenta. |
| **Panel de cliente** | React | Ecosistema amplio y curva de contratación/incorporación de desarrolladores corta, relevante en cuanto el equipo necesite crecer para atender más clientes. |

El hilo común de estas decisiones es "comprar en vez de construir" en todo lo que no es el diferencial del producto (la calidad y personalización del agente conversacional), y "construir" únicamente donde está el valor que el cliente paga.

---

## 5. Funcionalidades del producto, vistas desde lo que resuelven para el cliente

### 🏢 Gestión de inquilinos y usuarios
No es solo una tabla de organizaciones: es lo que permite vender autoservicio en vez de requerir onboarding manual por cada cliente nuevo, reduciendo el coste de adquisición.

### 🤖 Configuración del agente de voz
El cliente define personalidad, tono, voz e idioma del agente, y conecta su propia base de conocimiento (FAQs, documentos, endpoints externos). Esto traslada el trabajo de personalización al propio cliente, en vez de convertirlo en un proyecto de consultoría por cada contrato.

### 📞 Telefonía y enrutamiento
Vinculación de números a agentes, reglas de desvío, horarios y transferencia a un humano cuando el caso lo requiere — la vía de escape necesaria para que el producto sea vendible incluso a organizaciones que aún no confían del todo en dejar el 100% de la conversación en manos de una IA.

### 📊 Registro, transcripción y analítica
Cada llamada queda transcrita y medida (duración, tasa de resolución, latencia media). Esto no es solo trazabilidad técnica: es el dato que permite demostrarle al cliente el retorno de su inversión y justificar la renovación del contrato.

---

## 6. Hoja de ruta: de validar la idea a poder venderla con garantías

La secuencia de fases está pensada para reducir riesgo antes de comprometer más inversión en cada etapa siguiente:

1. **Cimientos de datos:** esquema multi-tenant en Supabase y políticas RLS — sin esto, nada de lo posterior es seguro de construir sobre ello.
2. **Backend y orquestación:** API en FastAPI e integración por webhooks con Vapi — aquí se valida que el pipeline conversacional funciona con la latencia objetivo.
3. **Telefonía real:** conexión de Twilio con Vapi para llamadas entrantes y salientes — primer punto donde el producto puede probarse con usuarios reales, no solo en local.
4. **Panel de cliente:** dashboard en React conectado al backend — necesario para que el producto sea autoservicio y no dependa de intervención manual del equipo por cada cambio de configuración.
5. **MVP listo para vender:** pruebas de carga y optimización de latencia — el punto en el que el producto deja de ser una demo y pasa a ser algo que se puede poner delante de un cliente de pago.
