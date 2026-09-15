Coderhouse — AI Automation Avanzado
Entregas del curso AI Automation Avanzado (Coderhouse).
Autor: Camilo Honorato
Este repositorio acumula los checkpoints del curso. El proyecto integrador es un solo
workflow de n8n que va creciendo módulo a módulo: cada entrega parte del `.json` de la
anterior y le suma los nodos del módulo nuevo.
---
Checkpoint 1 — Agente base y motor de razonamiento
Archivo: `checkpoint1_camilo_honorato.json`
Caso de negocio
Asistente de triaje comercial para una empresa de timbres de goma, grabado láser y
señalética. Atiende el primer contacto de clientes que escriben por el sitio web:
entiende qué necesitan, consulta el catálogo oficial y entrega producto sugerido,
precio referencial y plazo estimado. Cuando el caso excede su alcance, deriva al
equipo humano.
Tomé un caso de negocio concreto en vez de uno genérico porque me permitía escribir
reglas de negocio reales y una descripción de herramienta con criterios verificables.
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
Guardrail de iteraciones	`maxIterations` fijado en 8 dentro de las opciones del nodo, más Retry On Fail con 5 reintentos
System Prompt	Estructura modular: Rol → Ámbito → Objetivo → Reglas → Escalamiento, con límites explícitos de lo que el agente NO puede hacer
Herramienta	Google Sheets conectado al puerto lateral Tool, nunca como nodo secuencial
Descripción semántica	Detalla qué contiene la planilla, en qué casos de negocio activarla, en cuáles no, y la regla de no entregar ningún dato que no provenga de ella
Sin decisión lineal previa	El Chat Trigger entra directo al agente. No hay IF ni Switch antes: la ramificación la decide el modelo en el ciclo ReAct
Observabilidad	Nodo Gmail final que envía consulta, respuesta y pasos intermedios del razonamiento
Validación de ejecución manual
Prueba realizada desde el chat interno de n8n con el siguiente prompt:
> Hola, necesito 80 timbres automáticos para mi oficina, ¿cuánto me saldría y en cuánto tiempo los tienen?
Resultado auditado en el panel de ejecución:
El agente abrió la ramificación de la herramienta de forma autónoma, ejecutó la consulta
a Google Sheets y construyó la respuesta con esos datos. El recorrido registrado fue:
```
OpenAI Chat Model → Consultar Catálogo Oficial → OpenAI Chat Model
```
Las dos pasadas por el modelo confirman el ciclo ReAct completo: el agente razona,
decide invocar la herramienta, recibe el resultado y vuelve a razonar sobre él antes de
responder. Todos los nodos cerraron en verde y el log de observabilidad se envió por
Gmail sin errores de sintaxis en las expresiones.
Capturas del panel de ejecución, de la respuesta del agente y del correo recibido en
la carpeta `/capturas`.
Decisiones de diseño
Por qué no hay nodo de decisión antes del agente. Anteponer un IF convertiría al
agente en el final de una cadena determinista. La consigna pide lo contrario: que la
ramificación sea probabilística y la decida el modelo.
Por qué la descripción de la herramienta es tan extensa. Es lo único que el modelo
lee para decidir si la activa. Una descripción corta produce el problema de la
instrucción huérfana: la herramienta está conectada pero el agente no sabe cuándo
usarla y responde de memoria. Por eso incluye casos de activación, casos de no
activación y la prohibición explícita de inventar precios.
Temperatura en 0.3. El agente entrega precios y plazos. Menos variabilidad es mejor.
Problema encontrado durante la implementación
Intenté usar Google Gemini como modelo de razonamiento, siguiendo la implementación
mostrada en clase. Los modelos de la generación Gemini 3 devuelven un `thought_signature`
en las llamadas a herramientas que la API exige recibir de vuelta, y la versión de n8n
que tengo instalada no lo estaba propagando. El resultado era un error 400 que aparecía
únicamente al conectar herramientas: el chat simple funcionaba sin problema.
Es un bug conocido y está reportado en el repositorio de n8n. Como actualizar el servidor
implicaba un salto de versión mayor sobre una instancia con otro flujo en producción,
migré a OpenAI `gpt-4o`, que además es uno de los modelos que la consigna sugiere.
Notas sobre credenciales y seguridad
El archivo `.json` publicado no contiene secretos. n8n exporta únicamente el nombre
visible y un identificador interno por cada credencial; nunca la API key, el Client
Secret ni el token OAuth2, que permanecen cifrados en la base de datos de la instancia.
Ese identificador no otorga acceso a nada fuera de la instalación que lo generó.
Aun así, y por prolijidad al publicar el repositorio, los identificadores fueron
removidos del archivo antes de subirlo y la dirección de correo del nodo de
observabilidad fue reemplazada por una genérica.
Otras notas
El catálogo de productos usado como fuente de datos es una planilla de ejemplo con
datos ficticios, creada para esta entrega.
Ejecutado sobre una instancia de n8n self-hosted.
Historial de la entrega
Versión	Cambio
Inicial	Primera entrega del flujo
Corregida	Se explicitó `"agent": "toolsAgent"` en los parámetros del nodo en lugar de depender del valor por defecto de la versión; se explicitó `maxIterations`; se documentó la validación manual con el prompt utilizado; se removieron los identificadores de credenciales del archivo publicado
