# Club de Comensales

Formulario web de registro para el club de comensales de un restaurante. Página
única y autocontenida (`index.html`): sin dependencias, sin build, sin servidor.
Se sube tal cual a GitHub Pages, Netlify, Vercel o cualquier hosting estático.

El comensal deja sus datos en 3 pasos y el registro se envía a **un webhook de
n8n**, que desde ahí lo reparte a WhatsApp y a Google Sheets.

```
Formulario ──POST JSON──> Webhook n8n ──┬──> WhatsApp
                                        └──> Google Sheets (Append Row)
```

## Configuración

Todo se edita en el objeto `CONFIG`, al inicio del `<script>` de `index.html`:

| Clave | Para qué sirve |
|---|---|
| `N8N_BASE` | URL de tu instancia de n8n, sin barra final |
| `WEBHOOK_ID` | El ID del nodo Webhook |
| `MODO_PRUEBA` | `true` → `/webhook-test/` · `false` → `/webhook/` |
| `RESTAURANTE`, `KICKER`, `SUBTITULO`, `SUCURSAL`, `PIE` | Textos e identidad |
| `EXITO_TITULO`, `EXITO_TEXTO` | Pantalla final |
| `MODO_DEMO` | Si no hay webhook configurado, simula el envío y saca el JSON por consola |

### Pasar a producción

Activa el workflow en n8n (botón **Active**) y pon `MODO_PRUEBA: false`. Es el
único cambio. Con `true` la URL de prueba solo recibe datos mientras tengas el
workflow abierto con *Listen for test event*, y únicamente una vez por clic.

`SUCURSAL` permite reutilizar el mismo formulario en varios locales: el valor
viaja en el payload y puedes filtrar por él en n8n o en la hoja.

## Montar el workflow en n8n

**Nodo 1 — Webhook**

- HTTP Method: `POST`
- Path: el mismo valor de `WEBHOOK_ID`
- Respond: `Immediately`
- En *Options* → **Allowed Origins (CORS)**: el dominio donde publicas el
  formulario (o `*` para probar). Esto importa, ver la nota más abajo.

**Nodo 2 — Google Sheets**

- Operation: `Append Row`
- Los datos llegan en `{{ $json.body.nombre }}`, `{{ $json.body.email }}`, etc.
- Si las columnas de la hoja se llaman igual que los campos, *Map Automatically*
  hace el mapeo solo.

**Nodo 3 — WhatsApp**, colgando del mismo Webhook, en paralelo con Sheets. Usa
`{{ $json.body.telefono_wa }}` (número sin `+`, formato Cloud API).

### Encabezados para la hoja de cálculo

```
nombre	telefono	telefono_wa	telefono_local	lada	email	cumple_dia	cumple_mes	cumple_anio	cumple_mmdd	cumple_texto	como_nos_conocio	sugerencias	restaurante	sucursal	fecha_registro	origen
```

## Datos que se envían

Se manda como JSON, así que los números llegan como números y las cadenas
largas no se truncan.

| Campo | Ejemplo | Nota |
|---|---|---|
| `nombre` | `Juan Pérez` | |
| `telefono` | `+525512345678` | E.164 |
| `telefono_wa` | `525512345678` | sin `+`, para WhatsApp Cloud API |
| `telefono_local` | `5512345678` | |
| `lada` | `+52` | |
| `email` | `juan@mail.com` | siempre en minúsculas |
| `cumple_dia` | `14` | número |
| `cumple_mes` | `3` | número |
| `cumple_anio` | `1990` | número o `null` — el año es opcional |
| `cumple_mmdd` | `03-14` | para comparar cumpleaños a diario en n8n |
| `cumple_texto` | `14 de Marzo de 1990` | listo para meter en un mensaje |
| `como_nos_conocio` | `Redes sociales` | |
| `sugerencias` | | hasta 600 caracteres |
| `restaurante` | `Sabor & Casa` | |
| `sucursal` | `Matriz` | |
| `fecha_registro` | `2026-08-03T18:22:10.431Z` | ISO 8601 |
| `origen` | `formulario_web` | |

### Felicitaciones de cumpleaños

Un workflow aparte con un nodo Schedule diario que lea la hoja y compare
`cumple_mmdd` contra la fecha de hoy en formato `MM-DD`. Por eso el campo viene
ya calculado: evita tener que parsear fechas dentro de n8n.

## Sobre CORS

El formulario intenta primero un `POST` normal, que permite leer la respuesta y
saber de verdad si el registro se guardó. Si el navegador lo bloquea por CORS,
reintenta en modo `no-cors`: los datos **sí** llegan a n8n, pero el navegador ya
no deja leer la respuesta, así que un fallo posterior se le mostraría al
comensal como éxito.

Por eso conviene configurar *Allowed Origins* en el nodo Webhook: es lo que hace
que el camino bueno funcione y que ese respaldo no llegue a usarse.

Si n8n no responde en 15 segundos la petición se corta y el comensal ve un aviso
con opción de reintentar, en lugar de un botón girando indefinidamente.

## Desarrollo

No hay nada que instalar. Abre `index.html` en el navegador, o levanta un
servidor local para que el envío no choque con restricciones de `file://`:

```sh
python3 -m http.server 8000
```

Con `MODO_DEMO: true` y sin webhook configurado, el formulario simula el envío y
escribe el payload completo en la consola: útil para revisar los datos antes de
conectar n8n.
