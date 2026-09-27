# SnackCheck — Nutrition Verdict API

Proyecto capstone de "Automatizaciones No-Code" (4Geeks Academy). API 100% no-code en n8n: recibe un código de barras y devuelve un veredicto de salud nutricional en lenguaje claro, escrito por IA.

- **Workflow:** `SnackCheck - Nutrition Verdict API`
- **Editor:** https://pgsotos.app.n8n.cloud/workflow/dwuFHftahtya46Na
- **Endpoint:** `POST https://pgsotos.app.n8n.cloud/webhook/nutrition-check`
- **Nodos:** 23, todos nativos de n8n (Webhook, Set, If, HTTP Request, Code, Switch, Basic LLM Chain + Groq, Merge, Respond to Webhook)

## Propósito

Apuntas el código de barras de un producto y la API responde con un veredicto de tres niveles (🟢 saludable / 🟡 moderado / 🔴 no saludable), marcadores rojo/verde por nutriente, y un párrafo de IA con tono según el nivel. Toda la lógica vive en n8n, sin backend propio.

## Cómo funciona

**1. Entrada y validación** — Webhook `POST /nutrition-check` recibe `{ "barcode": "..." }`. Un `Set` normaliza el input; dos `If` distinguen solicitud vacía (autodocumentación), barcode ausente o no numérico (400) y barcode válido.

**2. Consulta a Open Food Facts** — `HTTP Request` (sin API key) trae nombre, marca, Nutri-Score y nutrientes. `fullResponse` + `neverError` para inspeccionar `statusCode` y `body.status` directamente.

**3. Clasificación** — Un `Code` calcula dos lecturas y se queda con la peor:
- **Semáforo:** azúcar >22.5g, sal >1.5g, grasa >17.5g por 100g (FSA/UK). 0 altos = saludable, 1 = moderado, 2-3 = no saludable.
- **Nutri-Score:** A/B = saludable, C = moderado, D/E = no saludable.
- Nivel final = `Math.max()` de ambas (ej. Coca-Cola en TC-005).
- Sin datos suficientes → `insufficient_data`, nunca "saludable" por defecto.

**4. Veredicto con IA** — Un `Switch` enruta a una de tres cadenas de IA independientes (Groq, `openai/gpt-oss-120b`): 🟢 alentador, 🟡 alentador pero honesto, 🔴 cauteloso con alternativa. Nodos separados, no un if dentro de un mismo prompt.

**5. Respuesta** — `Merge` junta la rama ejecutada; `Set` arma el texto final (titular, marcadores, Nutri-Score, párrafo IA). `Respond to Webhook` devuelve `text/plain`.

## Configuración

- **Credencial:** `groqApi` ya configurada en n8n. Open Food Facts no requiere clave.
- **Modelo:** `openai/gpt-oss-120b` vía Groq, temperatura 0.6. Antes `llama-3.3-70b-versatile`, ya no disponible (ver TC-007).
- **Activación:** en n8n Cloud se usa **Publish**, no el toggle Active/Inactive. Sin publicar, solo responde `/webhook-test/nutrition-check` tras "Listen for test event".

## Uso

```
POST https://pgsotos.app.n8n.cloud/webhook/nutrition-check
Content-Type: application/json

{ "barcode": "3017620422003" }
```

Respuesta (200, texto plano):

```
🔴 NO SALUDABLE — Nutella

🔴 Azúcar: 56.3g / 100g
🟢 Sal: 0.107g / 100g
🔴 Grasa: 30.9g / 100g
Nutri-Score: E

Este producto tiene un nivel alto de azúcar y grasa, lo que puede aportar
muchas calorías si se consume de forma habitual; por eso es conveniente
limitar su cantidad. Puedes disfrutarlo en porciones pequeñas y acompañarlo
con alimentos más nutritivos, como fruta fresca o yogur natural, para
equilibrar su aporte. Si buscas una alternativa más ligera, prueba una
crema de frutos secos sin azúcares añadidos o una mantequilla de almendra.
```

Barcodes usados en pruebas: `3017620422003` (Nutella), `5449000000996` (Coca-Cola), `5000159461122` (Snickers), `7613034626844` (Chocapic), `6111242100992` (Perly), `8076809513722` (Basilico). Detalle en el test log.

## Manejo de errores

| Caso | Código | Respuesta |
|---|---|---|
| Body vacío | 200 | Autodocumentación de la API |
| `barcode` ausente o no numérico | 400 | `{ error: "invalid_barcode", message, received, example_request }` |
| Producto no encontrado | 404 | `{ error: "product_not_found", message, barcode_consultado, sugerencia }` |
| Producto sin datos nutricionales suficientes | 200 | `{ status: "insufficient_data", message, product_name, barcode }` |
| Clasificación inesperada (red de seguridad) | 500 | `{ error: "unexpected_classification", message }` |

TC-003: Open Food Facts puede responder 200 con `body.status: 0` para un código inválido. Por eso `Data - Product Found?` valida `statusCode == 200 Y body.status == 1`, no solo el código HTTP.

## Limitaciones

- Depende de lo que tenga Open Food Facts: sin Nutri-Score o nutrientes completos, el sistema responde `insufficient_data`.
- Umbrales fijos (22.5g azúcar / 1.5g sal / 17.5g grasa por 100g), no varían por tipo de producto.
- ~2-3s por solicitud, la mayoría en la llamada al modelo de IA.
