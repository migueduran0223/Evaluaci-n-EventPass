# EventPass — Backend n8n

Backend académico de EventPass construido en n8n Cloud con Google Sheets como persistencia, Gmail y Telegram como canales de notificación y un asistente informativo con IA. No incluye frontend por el alcance definido para esta entrega.

## Arquitectura

Cliente o herramienta HTTP → webhooks y chat alojado de n8n → reglas de negocio → archivos Google Sheets independientes → Gmail / Telegram / modelo IA.

Toda lectura y escritura del negocio pasa por n8n. Las credenciales OAuth están guardadas en n8n y no se incluyen en estos archivos.

## Workflows

| ID | Responsabilidad | Trigger / ruta |
|---|---|---|
| WF01 | CRUD lógico de usuarios; PBKDF2 + salt; perfil privado requiere sesión del mismo usuario | Webhook POST `/eventpass/usuarios` |
| WF02 | Login, logout, validación y expiración de sesiones de 24 h | Webhook POST `/eventpass/auth` |
| WF03a | Emisión de código temporal de vinculación | Webhook POST `/eventpass/telegram/codigo` |
| WF03b | Vinculación por comando `/vincular CODIGO` | Telegram Trigger |
| WF04 | Crear, consultar, editar, cerrar o cancelar eventos; acceso de administrador en n8n | Form Trigger |
| WF05 | Catálogo, detalle, filtro por categoría y disponibilidad calculada | Webhook GET `/eventpass/catalogo` |
| WF06 | Crear/listar/actualizar/cancelar inscripciones, cupos, duplicados y lista de espera | Webhook POST `/eventpass/inscripciones` |
| WF07 | Promoción FIFO de lista de espera | Schedule cada 5 minutos (America/Bogota) |
| WF08 | Recordatorios idempotentes de 24 h y 1 h | Schedule cada 30 minutos (America/Bogota) |
| WF09 | Notificación central por Gmail y Telegram, estado por canal y reintento idempotente | Execute Sub-workflow Trigger |
| WF10 | Soporte conversacional informativo; eventos publicados de solo lectura; historial persistido | Chat Trigger alojado por n8n |
| WF11 | Check-in; estados EXITOSO, DUPLICADO y RECHAZADO; cambia inscripción a ASISTIO | Webhook POST `/eventpass/checkin` |

Los webhooks de producción siguen el formato `https://miguelrosario0221.app.n8n.cloud/webhook/<ruta>` una vez publicado cada workflow. La ruta de prueba de n8n es distinta.

## Persistencia

Cada dominio conserva su archivo independiente:

| Archivo | Hoja(s) |
|---|---|
| [EP01_Usuarios](https://docs.google.com/spreadsheets/d/1--0E5mKFWKdowk0MgFPNhUevMPGJGMZ7WzsEzCkqa3Q/edit) | Usuarios, Auditoria_Usuarios |
| [EP02_Sesiones](https://docs.google.com/spreadsheets/d/11kqf6drIGzfc-SoLxOWoML9rIASGF7i-T5-bFeXkZa4/edit) | Sesiones |
| [EP03_Telegram](https://docs.google.com/spreadsheets/d/1tmNkyXAXnoSXLMaDc-yKMKDGuj3obmMPONUlQrhvLVs/edit) | Codigos_Vinculacion, Vinculaciones |
| [EP04_Eventos](https://docs.google.com/spreadsheets/d/1poduWlID1AnfihGcgojYTl6gF61-QpJAfqS8tWJywVY/edit) | Eventos, Auditoria_Eventos |
| [EP05_Catalogo_Log](https://docs.google.com/spreadsheets/d/1HYzJF8RPMn9qUkWRcxNu6O_mBf9dy4qXXKjv88iyFRc/edit) | Consultas_Catalogo |
| [EP06_Inscripciones](https://docs.google.com/spreadsheets/d/12cYmDkPbowVQC3dCOlnWL5LXeFmo0wXQU6EWFtYcD7I/edit) | Inscripciones, Auditoria_Inscripciones |
| [EP07_Reasignaciones](https://docs.google.com/spreadsheets/d/1kX3n1LpoDUF3VD5DeM3uIG9LJSr70x8yh0ItWvuRltA/edit) | Reasignaciones |
| [EP08_Recordatorios](https://docs.google.com/spreadsheets/d/1rYnYdNmomX0P1DEzDh1R1aw0eMi5OzmXUQMiiHEPA4U/edit) | Recordatorios |
| [EP09_Notificaciones](https://docs.google.com/spreadsheets/d/1w5ABRhvi3-p_vdjyfI-yZTZTJHGxgTvkJHZTcGyP-Jk/edit) | Notificaciones |
| [EP10_Soporte](https://docs.google.com/spreadsheets/d/1BOWniK6PrLZS_xmdUS0l3lqky4F5U1r_WVXqrUAQzIw/edit) | Conversaciones, Mensajes |
| [EP11_Checkin](https://docs.google.com/spreadsheets/d/1bDlqOU0XlYEBhgK02j6XYHqBeeoNoc1xm9D5MrDkcgk/edit) | Checkins |

EP04 contiene seis eventos de demostración en cinco categorías, con estados PUBLICADO, CERRADO y CANCELADO. EP06 incluye dos inscripciones sintéticas con `origen=DEMO` para mostrar disponibilidad cero en el evento `EVT-DEMO-002`; no son personas reales.

## Reglas de negocio

- Usuarios: ACTIVO / INACTIVO. No se eliminan físicamente.
- Password: PBKDF2-SHA256, salt aleatorio individual de 16 bytes y 120.000 iteraciones.
- Sesiones: token aleatorio; vence a las 24 horas; se valida contra usuario activo.
- Eventos: BORRADOR / PUBLICADO / CERRADO / CANCELADO.
- Inscripciones: CONFIRMADA / LISTA_ESPERA / CANCELADA / ASISTIO. Se evita otra inscripción activa por usuario y evento.
- Cupos: capacidad menos inscripciones CONFIRMADA; no existe contador manual de disponibilidad.
- Reasignación: promoción FIFO por fecha de inscripción.
- Recordatorios: clave usuario-evento-tipo; se registra PENDIENTE antes de notificar.
- Notificaciones: WF09 busca el último estado de la clave idempotente. No reenvía un canal ya ENVIADO y puede reintentar el otro canal. Los estados Gmail y Telegram se guardan por separado.
- Asistente: no tiene herramientas de escritura; informa solo sobre eventos PUBLICADO y el funcionamiento de EventPass.
- Check-in: exige inscripción CONFIRMADA, evento coincidente y ausencia de check-in EXITOSO previo; el estado pasa a ASISTIO.

## WF11 — Check-in digital

Endpoint: `POST https://miguelrosario0221.app.n8n.cloud/webhook/eventpass/checkin`

Solicitud mínima:

```json
{
  "inscripcion_id": "INS-123",
  "evento_id": "EVT-001"
}
```

Resultado: HTTP 200 para EXITOSO; HTTP 409 para DUPLICADO; HTTP 422 para RECHAZADO. Cada intento queda en EP11_Checkin/Checkins con resultado y detalle. En un ingreso exitoso se actualiza EP06 y se llama WF09.

## JSON de workflows

Los archivos están en `n8n/`. WF03 está dividido en dos flujos (solicitar código y procesar el bot).

- [WF01_usuarios_crud.json](n8n/WF01_usuarios_crud.json)
- [WF02_auth_sesiones.json](n8n/WF02_auth_sesiones.json)
- [WF03_vinculacion_telegram_codigo.json](n8n/WF03_vinculacion_telegram_codigo.json)
- [WF03_vinculacion_telegram_bot.json](n8n/WF03_vinculacion_telegram_bot.json)
- [WF04_eventos_crud.json](n8n/WF04_eventos_crud.json)
- [WF05_catalogo_publico.json](n8n/WF05_catalogo_publico.json)
- [WF06_inscripciones_crud.json](n8n/WF06_inscripciones_crud.json)
- [WF07_reasignacion_lista_espera.json](n8n/WF07_reasignacion_lista_espera.json)
- [WF08_recordatorios.json](n8n/WF08_recordatorios.json)
- [WF09_notificaciones.json](n8n/WF09_notificaciones.json)
- [WF10_eventpass_assistant.json](n8n/WF10_eventpass_assistant.json)
- [WF11_checkin_digital.json](n8n/WF11_checkin_digital.json)

## Publicación y uso

Los workflows están guardados como borradores inactivos. Esto evita abrir endpoints públicos o iniciar envíos automáticos antes de la revisión. Para poner en marcha la solución:

1. Publica WF09 y después los workflows consumidores que lo llaman.
2. Publica WF01–WF06 y WF11 para exponer los webhooks; publica WF04 para obtener el formulario administrativo.
3. Activa WF03b para que el bot reciba `/vincular CODIGO`.
4. Publica WF10 para habilitar el Chat alojado de n8n. Usa créditos Gateway de n8n para GPT-5 mini.
5. Publica WF07 y WF08 cuando quieras habilitar promociones y recordatorios periódicos.

Las credenciales conectadas en n8n son Google Sheets, Gmail y Telegram. El modelo de WF10 usa credenciales administradas por n8n Gateway. No se requiere `.env.example` porque este alcance no incluye código frontend ni secretos locales.
