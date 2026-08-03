# Workflows de n8n

Dos workflows independientes:

| Archivo | Qué hace | Se dispara con |
|---|---|---|
| `workflow-registro.json` | Guarda cada alta en Google Sheets | El formulario, vía webhook |
| `workflow-cumpleanos.json` | Busca quién cumple años hoy | Un horario diario |

## Importarlos

En n8n: menú **⋯** (arriba a la derecha) → **Import from File**. Uno cada vez.

Vienen sin credenciales ni hoja seleccionada — eso se elige después, ver abajo.

---

## 1. Registro

```
Webhook (POST)  →  Aplanar datos  →  Guardar en Sheets
```

**Por qué el nodo de en medio.** El webhook entrega los datos anidados dentro de
`$json.body`. El nodo *Aplanar datos* los sube al primer nivel, y así el nodo de
Sheets puede usar **Map Automatically** y resolver las 17 columnas solo, en vez
de mapearlas a mano una por una.

### Qué configurar tras importar

**Nodo "Guardar en Sheets"**

1. *Credential*: conecta tu cuenta de Google (o elige una que ya tengas).
2. *Document*: elige la hoja de cálculo.
3. *Sheet*: elige la pestaña.
4. *Mapping Column Mode*: debe quedar en **Map Automatically**.

**Nodo "Registro del formulario"**

En *Options* → **Allowed Origins (CORS)** viene `*` para que funcione desde el
primer momento. Cámbialo por tu dominio real cuando publiques el formulario.

### Encabezados de la hoja

Fila 1, con estos nombres exactos (el mapeo automático los busca por nombre):

```
nombre	telefono	telefono_wa	telefono_local	lada	email	cumple_dia	cumple_mes	cumple_anio	cumple_mmdd	cumple_texto	como_nos_conocio	sugerencias	restaurante	sucursal	fecha_registro	origen
```

### Activarlo

Botón **Active**. El formulario ya apunta a la URL de producción
(`/webhook/512fe979-…`), así que en cuanto lo actives empieza a recibir.

### Añadir WhatsApp

Cuelga el nodo de WhatsApp del **mismo** nodo *Aplanar datos*, en paralelo con
Sheets — no en cadena después. Así un fallo de WhatsApp no impide que el
registro se guarde.

Usa `{{ $json.telefono_wa }}`: número sin `+`, que es el formato que espera
WhatsApp Cloud API.

---

## 2. Cumpleaños del día

```
Cada mañana (9:00)  →  Leer comensales  →  ¿Cumple hoy?  →  [tu nodo de WhatsApp]
```

El filtro compara `cumple_mmdd` de cada fila contra la fecha de hoy en formato
`MM-dd`. Por eso el formulario ya envía ese campo calculado: evita parsear
fechas dentro de n8n, que es donde se suelen colar los errores de zona horaria.

### Qué configurar tras importar

1. **Leer comensales**: la misma credencial y la misma hoja que el workflow de registro.
2. **Zona horaria**: *Settings* del workflow → *Timezone* → `America/Mexico_City`.
   Sin esto el nodo usa UTC y a partir de las 18:00 hora de México ya estaría
   comparando contra el día siguiente.
3. Cuelga tu nodo de WhatsApp de la salida del filtro.

Lo que sale del filtro son solo las personas que cumplen años hoy, cada una como
un item aparte. El nodo de WhatsApp se ejecuta una vez por persona
automáticamente — no hace falta un bucle.

Si nadie cumple, el filtro no deja pasar nada y el workflow termina ahí.

### Probarlo sin esperar un año

En el nodo *¿Cumple hoy?*, cambia temporalmente el valor de la derecha por un
`MM-dd` que sí exista en tu hoja (por ejemplo `03-14`), ejecuta a mano con
**Test workflow**, y luego devuélvelo a `{{ $now.toFormat('MM-dd') }}`.

---

## Notas

Las versiones de los nodos (`typeVersion`) corresponden a n8n 1.x reciente. Si tu
instancia es más antigua y algún nodo se importa en rojo, bórralo y añádelo a
mano con los mismos parámetros: la estructura de los tres nodos es la misma.
