Coderhouse — AI Automation Avanzado
Entregas del curso AI Automation Avanzado (Coderhouse).
Autor: Camilo Honorato
Este repositorio acumula los checkpoints del curso. El proyecto integrador es un asistente
de triaje comercial construido en n8n, que crece módulo a módulo: cada entrega parte del
`.json` de la anterior y le suma la capa del módulo nuevo.
Infraestructura: n8n 1.108.2 self-hosted (Docker Compose con PostgreSQL, Gotenberg y
Caddy) sobre un VPS Ubuntu 24.04. La versión está fijada a propósito: el salto a la rama 2.x
es de versión mayor y la instancia comparte espacio con otro flujo en producción.
Caso de negocio: empresa chilena de timbres de goma, timbres automáticos, grabado láser
y señalética corporativa. Todos los datos (catálogo, clientes, pedidos) son ficticios.
Índice
Checkpoint	Tema	Estado
1	Agente base con herramienta y observabilidad	Corregido y reenviado
2	Arquitectura multi-agente Manager-Worker	Entregado
3	Memoria de largo plazo por Session_ID y summarization	Entregado
---
Checkpoint 1 — Agente base y motor de razonamiento
Archivo: `checkpoint1_camilo_honorato.json`
Caso de negocio
Asistente de triaje comercial que atiende el primer contacto de clientes que escriben por
el sitio web: entiende qué necesitan, consulta el catálogo oficial y entrega producto
sugerido, precio referencial y plazo estimado. Cuando el caso excede su alcance, deriva al
equipo humano.
Tomé un caso de negocio concreto en vez de uno genérico porque me permitía escribir reglas
de negocio reales y una descripción de herramienta con criterios verificables.
Estructura del flujo
```
[Chat Trigger] → [AI Agent · Tools Agent] → [Gmail · Log de observabilidad]
                        ├── Chat Model: OpenAI gpt-4o
                        └── Tool: Google Sheets (catálogo de productos)
```
Cómo se resolvió cada punto de la consigna
Requisito	Implementación
Disparador	Chat Trigger, conectado directamente al agente
Modo del agente	`"agent": "toolsAgent"`, declarado de forma explícita en los parámetros del nodo
Modelo	OpenAI `gpt-4o`, temperatura 0.3
Guardrail de iteraciones	`maxIterations` fijado en 8, más Retry On Fail con 5 reintentos
System Prompt	Estructura modular: Rol → Ámbito → Objetivo → Reglas → Escalamiento
Herramienta	Google Sheets conectado al puerto lateral Tool, nunca como nodo secuencial
Descripción semántica	Qué contiene la planilla, cuándo activarla, cuándo no, y la regla de no entregar datos que no provengan de ella
Sin decisión lineal previa	No hay IF ni Switch antes del agente: la ramificación la decide el modelo en el ciclo ReAct
Observabilidad	Nodo Gmail final con consulta, respuesta y pasos intermedios del razonamiento
Validación de ejecución manual
Prompt usado desde el chat interno de n8n:
> Hola, necesito 80 timbres automáticos para mi oficina, ¿cuánto me saldría y en cuánto tiempo los tienen?
Recorrido auditado en el panel de ejecución:
```
OpenAI Chat Model → Consultar Catálogo Oficial → OpenAI Chat Model
```
Las dos pasadas por el modelo confirman el ciclo ReAct completo: el agente razona, decide
invocar la herramienta, recibe el resultado y vuelve a razonar antes de responder. Todos los
nodos cerraron en verde y el log de observabilidad llegó por Gmail.
Decisiones de diseño
Sin nodo de decisión antes del agente. Un IF previo convertiría al agente en el final
de una cadena determinista; la consigna pide ramificación probabilística.
Descripción de herramienta extensa. Es lo único que el modelo lee para decidir si la
activa. Una descripción corta produce el problema de la instrucción huérfana.
Temperatura 0.3. El agente entrega precios y plazos: menos variabilidad es mejor.
Problema encontrado: Gemini y `thought_signature`
Intenté usar Google Gemini, como se mostró en clase. Los modelos Gemini 3 devuelven un
`thought_signature` en las llamadas a herramientas que la API exige recibir de vuelta, y
n8n 1.108.2 no lo propaga: error 400 apenas se conecta una herramienta (el chat simple sí
funciona). Es un bug conocido y reportado en el repositorio de n8n. Migré a OpenAI `gpt-4o`.
Historial

Versión	Cambio
Inicial	Primera entrega del flujo
Corregida	Se explicitó `"agent": "toolsAgent"` y `maxIterations`; se documentó la validación manual con el prompt usado; se removieron los identificadores de credenciales del archivo publicado
---
Checkpoint 2 — Arquitectura multi-agente Manager-Worker
Archivos:
`manager_modulo2_honorato_camilo.json`
`worker1_catalogo_honorato_camilo.json`
`worker2_redaccion_honorato_camilo.json`
Formato de entrega: PDF con capturas subido a la plataforma (los JSON fueron opcionales).
Arquitectura
Patrón Manager-Worker con sub-workflows en lienzos separados: tres workflows independientes.
Manager (orquestador)
```
Chat Trigger
  → AI Agent "Router de Triaje"      sin herramientas; solo clasifica
       ├── Chat Model: OpenAI gpt-4o (temperatura 0.1)
       └── Output Parser estructurado
  → Set "Contrato de Datos"          limpieza de payload (evita Data Stuffing)
  → Switch "Enrutamiento"
       ├── CONSULTA_CATALOGO    → Execute Workflow → Worker 1
       ├── SOLICITUD_COTIZACION → Execute Workflow → Worker 2
       └── ESCALAMIENTO_HUMANO  → Set "Ruta de Escape" (fallback)
  → Gmail "Log de Trazabilidad"      las tres ramas convergen aquí
```
Worker 1 — Especialista de Catálogo
```
Execute Workflow Trigger → Google Sheets → Aggregate → Contrato de salida OK
                                  └── (salida de error) → Contrato de salida ERROR
```
Worker 2 — Especialista de Redacción
```
Execute Workflow Trigger → LLM Chain (redacta correo de cotización) → Contrato OK
                                  └── (salida de error) → Contrato ERROR
```
Los dos nodos Execute Workflow tienen Wait for Sub-Workflow Completion activado, y los
tres workflows tienen timeout de 5 minutos configurado en Settings.
Contratos de datos
Manager → Workers. El Set normaliza la salida del Router en un objeto plano de siete
campos: `categoria`, `consulta_cliente`, `categoria_producto`, `cantidad`,
`nombre_cliente`, `detalle_pedido`, `session_id`. Cada Execute Workflow envía solo el
subconjunto que su especialista necesita:
Worker 1 recibe: `consulta_cliente`, `categoria_producto`, `cantidad`, `session_id`
Worker 2 recibe: `nombre_cliente`, `detalle_pedido`, `cantidad`, `session_id`
Workers → Manager. Ambos devuelven siempre la misma estructura:
```json
{
  "status": "success",
  "worker": "especialista_catalogo",
  "session_id": "...",
  "data": { }
}
```
Ante fallo, el mismo esquema con `status: "error"`, un objeto `error` con código y mensaje
legible, y `data` en `null`. Los Workers nunca mueren en silencio: los nodos críticos tienen
salida de error hacia una rama que construye el JSON de contingencia.
Taxonomía cerrada del Router
Categoría	Cuándo se asigna
`CONSULTA_CATALOGO`	Productos, precios, materiales, medidas, plazos
`SOLICITUD_COTIZACION`	El cliente ya definió qué quiere y pide la propuesta formal
`ESCALAMIENTO_HUMANO`	Ambigüedad, reclamos, facturación, crédito, pedidos sobre 100 unidades, temas fuera de ámbito
Regla de sesgo conservador: ante la duda, escalar.
Validación de ejecución
Prompt	Ruta	Resultado
"¿Cuánto cuesta un timbre automático mediano y en cuánto tiempo lo tienen?"	CONSULTA_CATALOGO	Delegó al Worker 1, devolvió el catálogo
"Soy Camilo Honorato, necesito una cotización formal por 40 timbres automáticos"	SOLICITUD_COTIZACION	Delegó al Worker 2, devolvió el borrador
"Necesito 300 unidades con factura y crédito a 30 días"	ESCALAMIENTO_HUMANO	Cayó en la ruta de escape sin invocar Workers
Decisión pensando en el módulo siguiente
El campo `session_id` viaja en todos los contratos, de ida y de vuelta, para que la memoria
del Checkpoint 3 pudiera engancharse sin rediseñar los contratos.
---
Checkpoint 3 — Memoria persistente y resumen agéntico
Archivo: `manager_modulo3_honorato_camilo.json` (los Workers del Checkpoint 2 no cambian)
Formato de entrega: PDF `PreEntrega_Modulo3_CamiloHonorato.pdf` subido a la plataforma.
Arquitectura
```
Chat Trigger
 → Buscar Memoria (Airtable · Search)       filtro exacto por Session_ID · Always Output Data
 → IF ¿Usuario recurrente?
     ├─ true  → Contexto para el Agente
     └─ false → Crear Registro Inicial (Airtable · Create) → Contexto para el Agente
 → AI Agent "Router de Triaje"              toolsAgent explícito · memoria en el System Prompt
     ├── Chat Model: gpt-4o
     ├── Memoria de Corto Plazo: Postgres Chat Memory (últimos 10 mensajes)
     └── Autocorrector de Formato (Auto-fixing Output Parser · gpt-4o-mini)
           └── Formato de Clasificacion (esquema JSON estricto: required + enum)
 → Contrato de Datos → Switch → Worker 1 / Worker 2 / Ruta de Escape    (igual que CP2)
 → Log de Trazabilidad (Gmail)              ahora incluye la memoria recuperada
 → Actualizar Memoria (Airtable · Update)   nombre, estado, datos clave, contador +1
 → IF ¿Supera 5 intercambios?
     └─ true → Leer Historial (Postgres) → Generar Resumen (gpt-4o-mini + Output Parser)
               → Guardar Resumen (Airtable · Update, sobrescribe)
```
Memoria híbrida
La consigna pide resumir el historial al superar 5 mensajes, pero prohíbe guardar
transcripciones en la base. Por eso se separaron dos memorias:
Memoria	Dónde vive	Qué guarda	Para qué sirve
Corto plazo	PostgreSQL, base dedicada `memoria_agentes`, tabla `n8n_chat_histories`	Historial crudo de la sesión	Que el Router siga el hilo ("40 de esos") y que el modelo mini tenga qué resumir
Largo plazo	Airtable, base `Memoria Agente Timbres`, tabla `Sesiones`	Solo resumen consolidado e indicadores	Se inyecta en el System Prompt del Router
Esquema de Airtable (tabla `Sesiones`)
Campo	Tipo	Escrito por	Contenido
`Session_ID`	Single line text (primario)	Crear Registro	`sessionId` del Chat Trigger
`Nombre del Cliente`	Single line text	Actualizar Memoria	Nombre extraído; si un mensaje no lo trae, se conserva el anterior
`Estado del Caso`	Single select	Crear / Actualizar	`NUEVO`, `CONSULTA_CATALOGO`, `SOLICITUD_COTIZACION`, `ESCALAMIENTO_HUMANO`
`Resumen Consolidado`	Long text	Guardar Resumen	JSON `{asunto_principal, puntos_clave, accion_requerida}`
`Datos Clave`	Long text	Actualizar Memoria	Producto, cantidad y detalle del pedido
`Contador de Intercambios`	Number (entero)	Crear / Actualizar	Dispara la summarization cuando es mayor que 5
`Fecha de Actualización`	Last modified time	Airtable	Automático
Acceso con Personal Access Token de permisos mínimos (`data.records:read`,
`data.records:write`, `schema.bases:read`), restringido a esta única base.
Inyección en el agente (Context Engineering)
El Set Contexto para el Agente recibe las dos ramas del IF y siempre entrega los mismos
campos (`user_name`, `last_summary`, `last_status`, `contador_previo`). Para un usuario nuevo
usa textos neutros, así el prompt nunca muestra variables vacías. El bloque inyectado en el
System Prompt del Router es:
```
[INICIO DE CONTEXTO COMPARTIDO]
CONFIGURACIÓN DE IDENTIDAD Y MEMORIA DE LARGO PLAZO
El usuario se llama {{ $json.user_name }}. El contexto de vuestra última charla es: {{ $json.last_summary }}.
Estado del ultimo caso registrado: {{ $json.last_status }}.
Intercambios previos en esta sesion: {{ $json.contador_previo }}.
FIN DEL CONTEXTO COMPARTIDO
[FIN DEL CONTEXTO COMPARTIDO]
```
Blindaje contra memoria envenenada: el prompt declara ese bloque como datos y no como
instrucciones. El prompt de summarization, a su vez, registra cualquier intento de cambiar
las reglas como un hecho, nunca como un acuerdo comercial.
Summarization
Modelo `gpt-4o-mini`, temperatura 0, en una Basic LLM Chain con Output Parser estructurado
(tres claves obligatorias) y 3 reintentos.
El historial se lee con una consulta parametrizada (`$1`), sin concatenar el Session_ID en
el SQL.
Idempotencia: la actualización es por Record ID y reemplaza el campo completo; repetir
la ejecución no acumula resúmenes.
"Cierre de sesión": el chat de n8n no emite un evento de fin de conversación, así que la
persistencia ocurre al final de cada ejecución.
Validación de ejecución
#	Objetivo	Mensaje	Resultado
1	Usuario nuevo	"Hola, soy Camilo Honorato. ¿Cuánto cuesta un timbre automático mediano?"	Rama false, registro creado; Airtable: CONSULTA_CATALOGO, contador 1
2	Recurrente con referencia previa	"Perfecto, entonces quiero la cotización formal por 40 de esos"	Rama true; memoria visible en el System Prompt enviado al modelo; misma fila con SOLICITUD_COTIZACION, contador 2
3	Summarization	Cuatro consultas más sobre plazos, archivo de arte y letrero de acrílico	Resumen con datos duros (cliente, 40 timbres, 1 letrero) guardado y reinyectado como `last_summary`
4	Aislamiento entre sesiones	"Hola, ¿cuál es mi nombre y qué había pedido?" (sesión nueva)	Nueva fila sin nombre; Router escala a ESCALAMIENTO_HUMANO sin datos de la sesión 1
5	Verificación de la incidencia 6	Sesión larga con mensajes dependientes del historial ("Quiero cotizar 30 de esos", "Confírmame lo que te pedí")	El autocorrector corrigió una respuesta rechazada; estado válido y resumen guardado
Incidencias encontradas en las pruebas y su corrección
#	Síntoma	Causa	Corrección
1	Buscar Memoria: `Unknown field name: ""`	Regla de orden (Sort) vacía enviada a la API	Se eliminó la regla
2	Crear Registro: `expects one of []` en Estado del Caso	Opciones del Single select no cargadas en el nodo	Refresh Column List
3	Generar Resumen: `Model output doesn't fit required format`	El prompt exigía un JSON "sin nada alrededor" y chocaba con el envoltorio del Output Parser	El prompt remite al formato del parser; esquema con tres claves obligatorias
4	Resumen con campos vacíos	La consulta leía `message.data.content`; esta versión guarda `message.content`	`COALESCE` sobre ambas rutas
5	Router: `Model output doesn't fit required format` ante una pregunta directa	El modelo respondió en texto libre	Reglas explícitas: siempre devolver el objeto; lo no clasificable va a ESCALAMIENTO_HUMANO
6	Estado del Caso guardado como `undefined`; con esquema estricto, el Router falla desde el 3er mensaje de sesiones con historial	Causa probable: la memoria muestra al modelo sus clasificaciones previas como JSON plano y las imita sin el formato que exige el parser. El parser generado desde ejemplo no tenía claves obligatorias y dejó pasar un objeto sin `categoria`	Esquema manual con `required` y `enum`, Auto-fixing Output Parser con gpt-4o-mini y respaldo `ESCALAMIENTO_HUMANO` en el Contrato de Datos. Verificado: en una sesión larga el autocorrector se activó en el 4º mensaje y el flujo completó
Limitaciones conocidas
La identidad es por sesión de chat, no por cliente: otro navegador equivale a una sesión
nueva. Mejora futura: correlacionar por correo o RUT.
El chat muestra la salida técnica del último nodo; falta un nodo de respuesta final al
cliente.
El autocorrector mitiga el síntoma, pero la causa de fondo (el modelo imita el historial en JSON)
sigue presente. Mejora futura: guardar en memoria un texto neutro o migrar a una versión
más reciente del nodo AI Agent.
`Datos Clave` se sobrescribe con "no informado" si el último mensaje no trae producto
(el resumen consolidado conserva el detalle).
---
Nota sobre parámetros explícitos
Al guardar, n8n elimina del archivo los parámetros cuyo valor coincide con el valor por
defecto (por ejemplo `"agent": "toolsAgent"` o `"tableName"`). Los `.json` publicados se
revisaron y esos parámetros se volvieron a escribir de forma explícita.
Notas sobre credenciales y seguridad
Los archivos `.json` publicados no contienen secretos. n8n exporta únicamente el nombre
visible y un identificador interno de cada credencial; nunca la API key, el Client Secret ni
los tokens, que permanecen cifrados en la base de datos de la instancia. Aun así, los
identificadores se removieron de los archivos publicados y las direcciones de correo se
reemplazaron por genéricas.
Checkpoint 4 — Integraciones avanzadas e interconexión de sistemas

Archivo: checkpoint4_camilo_honorato.json · Documentación con capturas: PreEntrega_Modulo4_CamiloHonorato.pdf

Evolución del Manager del Módulo 3. El Chat Trigger se reemplaza por una casilla de soporte en Gmail y el flujo se integra, vía OAuth2, con tres herramientas externas:

Herramienta	Rol en el caso	Uso en el flujo
Gmail	Casilla de soporte al cliente	Trigger de correos entrantes y creación de borradores
HubSpot	CRM de la tienda (fuente única de verdad)	Búsqueda y alta/actualización de contactos
Slack	Canal del equipo de operaciones (#operaciones)	Aviso corto por cada correo procesado (reemplaza el log de Gmail del M3)

HubSpot, Slack y la casilla de soporte son entornos de demostración creados para el caso. No corresponden a sistemas reales de la empresa y todos los datos son ficticios.

Arquitectura
Gmail Trigger (casilla demo, sin adjuntos, filtro -from:me)
 → ① IF anti auto-reply ──true──▶ Stop (corta el bucle)
 → ④ Set limpieza y validación → IF payload válido ──false──▶ Stop (evita el 400)
 → Buscar Memoria (Airtable, por correo) → IF recurrente → Contexto para el Agente
 → AI Agent "Router de Triaje" (igual que el M3)
 → Contrato de Datos → Switch → Worker 1 / Worker 2 / Ruta de Escape
 → Redactor de Respuesta (gpt-4o-mini)
 → ② HubSpot Look up por email → IF existe → Update | Create
 → ③ Gmail Create Draft (Human-in-the-loop)
 → Set Payload para Slack → Slack #operaciones
 → Actualizar Memoria → IF > 5 intercambios → Resumen (igual que el M3)
Los cuatro guardrails
Nodo	Qué hace	Riesgo que neutraliza
① IF anti auto-reply	Justo después del trigger. Corta si el asunto contiene auto-reply, autoreply, automatic reply, out of office, undeliverable, respuesta automática o fuera de la oficina, o si el remitente contiene no-reply, noreply o mailer-daemon	Bucle infinito de auto-respuestas
② Look up antes del Create	HubSpot Search por email (Always Output Data) y un IF que decide entre Update y Create	Error 409 (contactos duplicados)
③ Create Draft	La respuesta de la IA queda como borrador en el hilo del cliente. No existe ningún nodo de envío	Envío sin aprobación humana
④ Set de limpieza	Deja solo from_email, from_name, subject, body_text, thread_id, message_id (sin HTML, citas ni adjuntos). Un IF valida email no vacío, formato válido y cuerpo presente. Otro Set arma un texto corto para Slack	Error 400 y saturación del canal
Cambios respecto del Módulo 3
Identidad por correo: el session_id de Airtable, Postgres y el contrato con los Workers es el email del remitente (resuelve la limitación de identidad por sesión del M3).
Redactor de Respuesta: nuevo LLM que escribe el correo al cliente con los datos del Worker, sin inventar precios; en escalamientos redacta un acuse de recibo.
Log a Slack: un log por Gmail en una casilla vigilada por el trigger podía generar un bucle.
Mínimo privilegio

Los scopes de las credenciales OAuth2 de Gmail, HubSpot y Slack están fijados en el código de n8n 1.108.2 y no se pueden recortar desde la interfaz (se obtuvieron del parámetro scope de la URL de autorización). El mínimo privilegio se aplica en los nodos:

Conector	La credencial pide	El flujo usa
Gmail	Lectura, modificación y borradores	Solo lee entrantes y crea borradores
HubSpot	17 scopes (contacts, companies, deals, owners, schemas, lists, forms, tickets)	Solo crm.objects.contacts.read y .write, campos email, firstname, lastname y lifecyclestage
Slack	17 user scopes + chat:write (bot)	Solo chat:write en #operaciones
Test de regresión

Prueba manual con Test step nodo por nodo y luego Execute Workflow. Correo de prueba:

Asunto: Consulta timbres automáticos
Cuerpo: Hola, soy Andrea Soto de Ferretería Los Aromos. ¿Qué precio tiene el timbre automático y cuánto demora la fabricación? Saludos.

Resultado: rama false del IF anti auto-reply → payload válido → cliente nuevo en Airtable → CONSULTA_CATALOGO → Worker 1 → respuesta redactada → Look up sin resultados → contacto creado → borrador en Gmail → mensaje en Slack → memoria actualizada. En una segunda ejecución con el mismo remitente, el Look up encontró el contacto y el flujo pasó por la rama Update (HubSpot mantuvo un único contacto). Capturas en el PDF.

Incidencias
#	Incidencia	Causa	Corrección
1	Google Error 400: redirect_uri_mismatch	Se usaron datos de "Claves de API" / cuenta de servicio en vez del cliente OAuth 2.0	Usar ID y secreto del cliente OAuth tipo Aplicación web
2	HubSpot sin botón "Crear app"	HubSpot desactivó la creación de apps públicas legacy en junio de 2026	App creada con la CLI (hs project create), privada y con OAuth
3	Deploy fallido por scope tickets	HubSpot retiró tickets y lo reemplazó por crm.objects.tickets.read/write y crm.schemas.tickets.read/write	Reemplazo en app-hsmeta.json y nuevo deploy
4	HubSpot: redirect URL no coincide	El deploy fallido dejó la app con http://localhost:3000	Se resolvió con el deploy de la incidencia 3
5	n8n: The OAuth callback state is invalid	Autorización iniciada fuera del botón Connect de n8n o desde una ventana antigua	Recargar n8n, guardar la credencial y conectar desde n8n
6	Nodo HubSpot sin Create/Update separados	En n8n 1.108.2 solo existe "Create or Update" (upsert)	Ambas ramas usan upsert después del Look up; Create marca lead, Update solo el nombre
7	Parámetros por defecto eliminados al exportar	n8n omite valores por defecto al guardar	Reescritos a mano en el JSON publicado
Limitaciones
Scopes OAuth2 más amplios que el uso real (fijados por n8n).
El test recorrió la ruta CONSULTA_CATALOGO; las ramas de corte (auto-reply, payload inválido) se verificaron por configuración, sin un correo automático real.
Proyecto de Google Cloud en modo Testing: los tokens de Gmail vencen cada 7 días.
El workflow se deja inactivo: activarlo haría que cada correo entrante consuma saldo de OpenAI.
Para importar cualquier workflow: crear las credenciales propias en n8n (OpenAI, Google
Sheets, Gmail y, desde el Checkpoint 3, Airtable y Postgres) y seleccionarlas en cada nodo.
## Checkpoint 5 — Base de conocimiento documental (RAG)

**Entrega:** `PreEntrega_Modulo5_CamiloHonorato.pdf`

El Redactor de Respuesta pasa de LLM Chain a **AI Agent (Tools Agent) con RAG**: antes de responder un correo consulta el *Manual de Políticas Comerciales* (documento ficticio de 22 secciones) mediante la herramienta `manual_politicas_tyg`.

- **Parseo:** LlamaParse (nivel Cost Effective), con curación manual del markdown (jerarquía de títulos y tablas partidas por salto de página).
- **Chunking:** semántico por título, un fragmento por sección.
- **Almacén:** Simple Vector Store de n8n + embeddings `text-embedding-3-small`. El plan gratuito de LlamaCloud no permite índices.
- **Recuperación:** Top-K = 3. Sin Minimum Score (no disponible en el nodo), compensado con la regla "No sé".
- **System Prompt:** respuestas basadas al 100% en fuentes, citación de sección obligatoria y frase de escape "No sé".
- **Validación:** 5 preguntas ciegas con vocabulario informal; precisión y análisis en el PDF.
- **Limitación:** el almacén vive en memoria; tras reiniciar n8n hay que re-ejecutar la ingesta.
