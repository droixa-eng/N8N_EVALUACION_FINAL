EXPLICACION DEL FLUJO DE TRABAJO IMPLEMENTADO EN TRABAJO FINAL:
Growthive — Automatización de Propuestas Comerciales con IA

Proyecto final del curso de n8n. Un flujo de automatización que integra un formulario web, una base de datos en Airtable, un motor de inteligencia artificial y un canal de salida por correo, con un punto de validación humana antes de contactar al cliente final.

Este flujo fue diseñado para ser funcional en producción: automatiza un proceso real de Growthive, agencia de marketing digital especializada en Meta Ads y Google Ads para el sector inmobiliario y de seguros.

1. Caso de uso

Automatización de propuestas comerciales personalizadas.

Un prospecto interesado en los servicios de Growthive llena un formulario describiendo su necesidad en lenguaje natural (por ejemplo: "tenemos 15 años en el mercado pero casi no generamos leads online"). El sistema interpreta esa descripción con IA y redacta una propuesta comercial formal y personalizada, sin que un humano tenga que escribirla manualmente cada vez.

Este caso de uso requiere interpretación de lenguaje natural real: el sistema no llena una plantilla con variables, sino que la IA lee el contexto del prospecto (sector, servicio de interés, presupuesto, necesidad puntual) y redacta una propuesta distinta cada vez.

2. El "Cerebro" — Base de datos (Airtable)

La memoria del sistema está estructurada en dos tablas relacionadas entre sí, para que ningún dato quede aislado.

Tabla Leads — registro principal de cada prospecto

Campo	Función
Nombre, Email, Sector, Servicio_Interes, Presupuesto, Descripcion	Datos capturados del formulario
Estado	Campo de estado: Pendiente → Enviado / Rechazado / Error
Detalle_Error	Registro del motivo si algo falla en el proceso
Fecha	Timestamp de creación del lead

Tabla Propuestas — historial de propuestas generadas

Campo	Función
Lead (Link to another record)	Relación con la tabla Leads
Texto_Propuesta	Contenido generado por la IA
Fecha_Generacion	Fecha de generación
Aprobado	Estado de aprobación humana

La relación Lead ↔ Propuestas es bidireccional (Airtable genera el campo inverso automáticamente al crear el link), lo que evita datos aislados: cualquier propuesta siempre puede rastrearse hasta su lead de origen, y viceversa.

No hay múltiples fuentes de datos externas en este proyecto — todo el ciclo de vida del registro (creación, actualización de estado, vinculación con su propuesta) lo gestiona exclusivamente el flujo de n8n, nunca se edita manualmente.

3. El "Corazón" — Orquestación lógica (n8n)
Trigger inteligente

El flujo usa un Form Trigger nativo de n8n, no un webhook genérico ni un proceso de polling constante. Solo se ejecuta cuando un prospecto real envía el formulario, evitando el consumo innecesario de operaciones.

Motor de IA

El flujo integra Groq (modelos Llama, vía HTTP Request) como motor de IA, probado y funcionando end-to-end. La variable de respuesta se mapea explícitamente en un nodo dedicado ("Extraer Texto de Propuesta"):

{{ $json.choices[0].message.content }}

Los max_tokens de la petición están limitados a 700 para controlar costos de generación.

Gestión de errores (resiliencia)
El nodo HTTP Request que llama a la IA tiene configurado onError: continueErrorOutput — el equivalente a una directiva de Resume: si la API falla, el flujo no se rompe, sino que redirige la ejecución a una rama de error dedicada.
Se configuraron reintentos automáticos (3 intentos con espera de 3 segundos) ante fallos temporales de la API.
Un nodo de validación (IF) verifica que el formulario tenga los datos críticos completos antes de gastar una llamada a la IA — si faltan, ni siquiera se llama al modelo.
Ambos caminos de falla (datos incompletos o error de la API) convergen en un único nodo, "Registrar Error en Airtable", que guarda el motivo exacto en Detalle_Error y actualiza Estado = Error, dejando un registro persistente y auditable de cualquier fallo. Desde ahí, se notifica automáticamente por correo.
Nodos y variables dinámicas

Todos los nodos están nombrados según su función (Crear Lead en Airtable, Validar Datos Completos, Generar Propuesta con Groq (IA), etc.), y ningún dato de negocio está hardcodeado: cada valor (nombre, email, sector, presupuesto, texto de propuesta) se referencia dinámicamente desde el nodo de origen correspondiente mediante expresiones de n8n.

4. Human-in-the-loop

Antes de contactar al prospecto final, el flujo se detiene por completo y espera aprobación humana.

Se usa el nodo nativo de Gmail "Send and Wait for Response": le envía al responsable de Growthive un correo con la propuesta generada por la IA y pausa la ejecución del flujo hasta que la persona aprueba o rechaza directamente desde ese mismo correo — sin necesidad de volver a abrir n8n.

Solo si la propuesta es aprobada, el flujo continúa hacia el envío real al prospecto. Este punto de control evita el "efecto metralleta" de contactar prospectos automáticamente sin supervisión humana.

5. Salida — Gmail

El canal de salida es Gmail, usado en dos momentos distintos dentro del mismo flujo:

Interno: la solicitud de aprobación enviada al responsable de Growthive.
Externo: la propuesta final enviada al prospecto, y las notificaciones de error cuando algo falla.

Cada correo se dirige dinámicamente al destinatario correspondiente (por ejemplo, {{ $('Crear Lead en Airtable').item.json['Email'] }} para el prospecto), sin direcciones ni contenido hardcodeado.

Diagrama del flujo:
Formulario (Form Trigger)
        │
        ▼
Crear Lead en Airtable (Estado: Pendiente)
        │
        ▼
Validar Datos Completos ──[faltan datos]──► Registrar Error en Airtable ──► Notificar Error
        │ [datos OK]
        ▼
Preparar Prompt para IA
        │
        ▼
Generar Propuesta con Groq (IA) ──[falla API]──► Registrar Error en Airtable ──► Notificar Error
        │ [éxito]
        ▼
Extraer Texto de Propuesta
        │
        ▼
Guardar Propuesta en Airtable (vinculada al Lead)
        │
        ▼
Enviar Propuesta para Aprobación (Gmail — Send and Wait) ◄── Human-in-the-loop
        │
        ├──[Aprobada]──► Enviar Propuesta al Prospecto ──► Estado: Enviado
        │
        └──[Rechazada]──► Estado: Rechazado
