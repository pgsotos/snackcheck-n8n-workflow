# SnackCheck — Registro de Pruebas (Test Log)

Pruebas end-to-end contra el workflow real en n8n (`workflowId dwuFHftahtya46Na`), sin mocks: llamadas reales a Open Food Facts y al modelo de IA vía Groq. Se referencian por `executionId` de n8n.

## Resumen

| ID | Caso | Resultado |
|---|---|---|
| TC-001 | Solicitud vacía → autodocumentación | ✅ PASA |
| TC-002 | Barcode no numérico → error 400 | ✅ PASA |
| TC-003 | Barcode inexistente → error 404 (statusCode 200 con status:0) | ✅ PASA |
| TC-004 | No saludable por ambas lecturas (Nutella) | ✅ PASA |
| TC-005 | Peor-de-dos-lecturas es necesario (Coca-Cola) | ✅ PASA |
| TC-006 | Saludable, rama de IA "alentadora" (Perly) | ✅ PASA |
| TC-007 | Bug real: modelo Groq no disponible | ✅ DETECTADO Y CORREGIDO |
| TC-008 | Moderado, última rama de IA sin cobertura previa (Chocapic) | ✅ PASA |
| TC-009 | No saludable, snack distinto a los ya probados (Snickers) | ✅ PASA |
| TC-010 | Saludable, categoría distinta — salsa (Basilico) | ✅ PASA |
| TC-011 | Datos insuficientes, con datos fijados | ✅ PASA |
| TC-012 | Regresión — extracción de `energy-kcal_100g` | ✅ PASA |

## Cobertura por tipo de prueba

| Tipo requerido | Casos | Cubierto |
|---|---|---|
| Funcional (código válido → veredicto) | TC-004, TC-005, TC-006, TC-008, TC-009, TC-010, TC-012 | ✅ |
| Integración (API en vivo, sin mocks) | TC-001 a TC-010, TC-012 | ✅ |
| Error (código desconocido, entrada mal formada, cuerpo vacío, sin datos) | TC-001, TC-002, TC-003, TC-011 | ✅ |
| Rendimiento | Nota al final de este documento | ✅ |

---

### TC-001 — Solicitud vacía → autodocumentación

**Tipo:** Funcional / entrada · **Execution ID:** 1

`POST /nutrition-check` con `{}`. Esperado: 200 con autodocumentación, nunca un error en blanco.

**Resultado:** se ejecutó `Respond - Self-Documenting Info`, con nombre del servicio, uso, ejemplo de request y códigos de prueba conocidos. ✅ PASA

### TC-002 — Barcode no numérico rechazado con error estructurado

**Tipo:** Funcional / validación · **Execution ID:** 2

`{ "barcode": "abc123" }`. Esperado: 400, `invalid_barcode`.

**Resultado:** el regex `^[0-9]+$` rechazó el valor; se ejecutó `Respond - Invalid Barcode Error`. ✅ PASA

### TC-003 — Producto no encontrado (caso límite de la API externa)

**Tipo:** Integración / manejo de errores de terceros · **Execution ID:** 4

`{ "barcode": "0000000000000" }`. Open Food Facts respondió **HTTP 200**, no 404, con `status: 0` — normalizó el código a `00000000`, inexistente en su base.

**Resultado:** `Data - Product Found?` evalúa `statusCode == 200 Y body.status == 1`, así que enrutó igual a `Respond - Product Not Found` (404) con el barcode original y una sugerencia. Confirma que validar solo el código HTTP no bastaba. ✅ PASA

### TC-004 — No saludable, ambas lecturas de acuerdo (Nutella)

**Tipo:** Funcional / integración completa · **Execution ID:** 8

`{ "barcode": "3017620422003" }` — azúcar 56.3g, sal 0.107g, grasa 30.9g / 100g, Nutri-Score E.

**Resultado:** semáforo 2 altos + Nutri-Score E → `overall_level: no_saludable`. Marcadores 🔴 azúcar, 🟢 sal, 🔴 grasa; rama de IA "Cautious" bien enrutada. ✅ PASA

### TC-005 — Por qué "peor de las dos lecturas" es necesario, no cosmético (Coca-Cola)

**Tipo:** Funcional / lógica de negocio crítica · **Execution ID:** 9

`{ "barcode": "5449000000996" }` — azúcar 10.6g, sal 0g, grasa 0g / 100g (todo bajo el umbral alto), Nutri-Score E.

**Resultado:** `semaforo_level: 0`, `nutriscore_level: 2`, `overall_level: no_saludable` (`Math.max` de ambas). Marcadores 🟢🟢🟢 por nutriente pero titular 🔴 NO SALUDABLE. Solo con el semáforo el veredicto habría sido "saludable" — incorrecto. ✅ PASA

### TC-006 — Saludable, rama de IA "alentadora" (Perly)

**Tipo:** Funcional / integración completa · **Execution ID:** 11

`{ "barcode": "6111242100992" }` (yogur bebible) — azúcar 9.4g, sal 0g, grasa 3g / 100g, Nutri-Score A.

**Resultado:** `overall_level: saludable`, marcadores 🟢🟢🟢, párrafo de IA con tono celebratorio. Cierra la última rama del `Switch` sin cobertura hasta este punto (`AI - Health Verdict (Healthy)` + `Model - Groq (Encouraging)`). ✅ PASA

### TC-007 — Bug real detectado en pruebas: modelo Groq no disponible

**Tipo:** Error / regresión · **Execution ID:** 6 (falló), corregido y reverificado en 8, 9, 11

Con `llama-3.3-70b-versatile` en los tres nodos `lmChatGroq`, la primera ejecución completa terminó en `error` (`model_not_found`).

**Corrección:** se cambió `model` a `openai/gpt-oss-120b` en los tres nodos (`Model - Groq (Encouraging)`, `(Encouraging-Cautious)`, `(Cautious)`) vía `update_workflow`. Reverificado en TC-004, TC-005 y TC-006. ✅ DETECTADO Y CORREGIDO

### TC-008 — Moderado, última rama de IA sin cobertura previa (Céréales Chocapic)

**Tipo:** Funcional / integración completa · **Execution ID:** 20

`{ "barcode": "7613034626844" }` — azúcar 22.4g, sal 0.22g, grasa 4.8g / 100g (los tres justo bajo el umbral alto), Nutri-Score C.

**Resultado:** semáforo 0 altos + Nutri-Score C → `overall_level: moderado`. Marcadores 🟢🟢🟢 pero titular 🟡 MODERADO — otro caso real de "peor de las dos lecturas". Cierra la rama `AI - Health Verdict (Moderate)` + `Model - Groq (Encouraging-Cautious)`. ✅ PASA

### TC-009 — No saludable, snack de categoría distinta a los ya probados (Snickers)

**Tipo:** Funcional / diversidad de datos reales · **Execution ID:** 21

`{ "barcode": "5000159461122" }` — azúcar 51.8g, sal 0.629g, grasa 22.5g / 100g, Nutri-Score E.

**Resultado:** `overall_level: no_saludable`. Mismo resultado que TC-004 pero con chocolate/confitería en vez de crema untable, confirmando que la lógica generaliza. ✅ PASA

### TC-010 — Saludable, categoría distinta a un snack dulce (Basilico, salsa)

**Tipo:** Funcional / diversidad de datos reales · **Execution ID:** 22

`{ "barcode": "8076809513722" }` (salsa Barilla) — azúcar 5.9g, sal 0.9g, grasa 2.6g / 100g, Nutri-Score A.

**Resultado:** `overall_level: saludable`, marcadores 🟢🟢🟢. Confirma que la clasificación funciona fuera de "snack dulce", pese al nombre del proyecto. ✅ PASA

### TC-011 — Datos insuficientes (producto encontrado sin nutrientes ni Nutri-Score válido)

**Tipo:** Error / caso límite, con datos fijados (pin data) · **Execution ID:** 24

No se encontró un barcode real con `status: 1` + `nutriments: {}` + `nutriscore_grade: unknown` a la vez, así que se fijó la salida de `API Call - Open Food Facts Product` con esos datos sintéticos pero realistas.

**Resultado:** `Code - Classify Nutrition Level` devolvió `sugar_high/salt_high/fat_high: null`, `overall_level: null`, `insufficient_data: true`; `Data - Sufficient Data?` enrutó a `Respond - Insufficient Data` (200). Confirma que `nutriscore_grade: "unknown"` se trata igual que ausente. ✅ PASA

### TC-012 — Regresión: extracción de `energy-kcal_100g` con notación de corchetes

**Tipo:** Funcional / regresión · **Execution ID:** 23 (re-ejecución de Perly)

Al revisar el flujo contra el README oficial se detectó que `Code - Classify Nutrition Level` nunca extraía `nutriments['energy-kcal_100g']` — requisito explícito de la tarea. Se agregó `const energyKcal = typeof nutriments['energy-kcal_100g'] === 'number' ? nutriments['energy-kcal_100g'] : null;`, expuesto como `energy_kcal_100g`.

**Resultado:** `energy_kcal_100g: 97`, coincide con la API (97 kcal/100g). El resto de la clasificación no cambió respecto a TC-006. ✅ PASA

---

## Nota de rendimiento

En las ejecuciones exitosas registradas, una solicitud completa (validación + Open Food Facts + clasificación + IA + formateo) tomó entre ~2.1s y ~3.0s. La llamada al modelo de IA (Groq) fue el paso más costoso (~800ms en TC-005); Open Food Facts, típicamente 300-400ms. La ruta de clasificación tomó ~1.7s en las primeras ejecuciones (posible cold start del motor de expresiones) y bajó a <20ms después.

## Casos no cubiertos

Con TC-008 y TC-011, las tres ramas de IA y las cuatro rutas de error/estado especial quedaron todas probadas end-to-end (TC-011 con datos fijados, el resto con datos reales). Sin caso de prueba positivo:

- `Respond - Unexpected Classification` (500) es una red de seguridad defensiva. Con la lógica actual del nodo `Code`, `overall_level` solo puede ser `saludable`, `moderado`, `no_saludable` o (manejado aparte) indefinido — este camino no debería alcanzarse en producción, y no puede fabricarse un caso positivo sin forzar un valor inválido en el código del nodo. Queda documentado como diseño defensivo, no como hueco de cobertura.
