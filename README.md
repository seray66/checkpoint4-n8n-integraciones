# Checkpoint 4 – Integraciones avanzadas (n8n)

Sergio Ayala · Codehouse · Automatización con IA avanzado

Extensión del workflow del Módulo 3 con tres herramientas conectadas por credencial (Gmail, HubSpot, Slack) y controles para que el flujo no cree duplicados, no responda solo y no filtre datos de más.

## Archivos

- `checkpoint4_sergio_ayala.json`: workflow exportado de n8n. Se importa sin errores y no contiene secretos (solo nombres e ids de credenciales).
- `credentials.png`: captura de n8n > Credentials con las 3 conexiones activas.

## Qué hace

La rama nueva arranca con un Gmail Trigger y queda separada de la cadena del Módulo 3, que no se tocó (esa cadena depende del Chat Trigger).

1. **Gmail Trigger** recibe el mail.
2. **auto-reply?** (IF): descarta respuestas automáticas por asunto (auto-reply, out of office, undeliverable, delivery status, respuesta automática, fuera de la oficina) o por remitente (no-reply, mailer-daemon, postmaster). Esa rama termina en un NoOp.
3. **Agente de borrador** (AI Agent + Anthropic): redacta el cuerpo de la respuesta en español. El prompt trata el contenido del mail como información, no como instrucciones.
4. **Limpiar y validar payload** (Set): deja solo email (en minúscula y sin espacios), nombre, asunto y borrador.
5. **¿Datos válidos?** (IF): el email debe cumplir un regex básico. Si no, termina en **Descartar**.
6. **Look up contacto** (HubSpot, búsqueda por email) y **¿Contacto ya existe?**. Si existe, sigue sin crear nada. Si no, **Crear contacto**. Se busca antes de crear para evitar el error 409 por duplicado.
7. **Create Draft** (Gmail): crea solo un borrador. Nunca envía. La persona revisa y manda a mano (human-in-the-loop).
8. **Payload mínimo para Slack** (Set): arma solo el texto del aviso (remitente y asunto).
9. **Avisar a #operaciones** (Slack): publica ese texto. No se manda el cuerpo del mail ni el borrador.

## Credenciales

| Servicio | Tipo de credencial | Notas |
|---|---|---|
| Gmail | Gmail OAuth2 API | Scopes fijados por n8n: acceso completo a Gmail (leer, redactar, enviar y borrar), borradores, etiquetas y permisos de complemento. El permiso es amplio, así que el límite de "solo borrador, nunca enviar" lo impone el workflow (nodo Create Draft), no la credencial. |
| HubSpot | HubSpot Service Key | Ver desvío más abajo. |
| Slack | Slack OAuth2 API | App propia "n8n Checkpoint 4", scopes de bot `chat:write` y `channels:read`. App agregada al canal `#operaciones`. |

Redirect URL de OAuth2 (n8n local): `http://localhost:5678/rest/oauth2-credential/callback`.

### Desvío: HubSpot con Service Key y no OAuth2

HubSpot ya no permite crear apps legacy, que es lo que pide la credencial OAuth2 de n8n. Usé una **Service Key** con estos scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write` y `crm.schemas.contacts.read`. La conexión es autenticada y está activa (ver `credentials.png`), pero no es OAuth2 como pedía la consigna. **Pendiente: consultarlo con el instructor de Codehouse.**

## Pruebas de regresión manual

| # | Caso | Resultado esperado | Resultado |
|---|---|---|---|
| 1 | Mail normal de un contacto nuevo | Crea contacto, borrador y aviso en Slack | Pasó |
| 2 | Auto-reply | Se frena en "Ignorar auto-reply"; no corre nada de HubSpot, Gmail ni Slack | Pasó |
| 3 | Contacto que ya existe | Va por "Contacto existente"; no se duplica; sí crea borrador y aviso | Pasó |
| 4 | Email inválido | Va por false de "¿Datos válidos?" y termina en Descartar; no toca HubSpot, Gmail ni Slack | Pasó |

El caso 4 se probó cambiando temporalmente el campo `email` del Set por un texto fijo que no es un email. Después se restauró la expresión original.

## Limitaciones conocidas

- El IF de auto-reply no detecta mails promocionales o masivos que no usen un remitente no-reply ni un asunto de respuesta automática. Esos pasan como mails normales.
- El token OAuth de Gmail puede vencer (por ejemplo, si la app de Google Cloud está en modo "Testing"). Si pasa, hay que reconectar la credencial en n8n.
- La rama de mail es independiente de la cadena del Módulo 3: comparten el workflow, no los datos.

## Cómo importarlo

1. En n8n: **Workflows > Import from file** y elegir `checkpoint4_sergio_ayala.json`.
2. Reasignar las credenciales (Gmail, HubSpot, Slack, Anthropic, Airtable) a las propias.
3. En el nodo de Slack, elegir el canal propio.
