# Changelog

Formato basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y versionado [SemVer](https://semver.org/lang/es/). Versiones por hito, no por fecha.

## [1.1.1] — Diagrama incorporado

### Added
- Diagrama de flujo (`snackcheck-flow.excalidraw`), verificado contra el workflow real.

## [1.1.0] — Auditoría contra el README oficial

Revisión contra los checklists de la tarea; dos gaps corregidos.

### Added
- `Code - Classify Nutrition Level` extrae `nutriments['energy-kcal_100g']` y lo expone como `energy_kcal_100g` (no se muestra en el veredicto, el formato no lo pide).
- Notes en los tres nodos `Model - Groq (...)`.
- TC-011: rama `insufficient_data` probada con datos fijados (pin data).
- TC-012: regresión del fix de `energy_kcal_100g`.
- Sección "Cobertura por tipo de prueba" en el test log.

### Changed
- README: limitación de `insufficient_data` actualizada.
- README: nota sobre calorías extraídas pero no mostradas.

## [1.0.1] — Ampliación de cobertura de pruebas

### Added
- TC-008, TC-009, TC-010: tres productos reales más (Chocapic, Snickers, Basilico).
- TC-008 confirma la rama de IA "moderado" con datos reales, la única sin probar en 1.0.0.
- TC-009 y TC-010 prueban categorías distintas (confitería, salsa).

### Changed
- README: sección "Créditos de prueba" eliminada.
- README: limitación de la rama "moderado" actualizada.

## [1.0.0] — Primera versión funcional (capstone-ready)

Primera versión completa, validada contra Open Food Facts y Groq reales.

### Added
- Webhook `POST /nutrition-check`, `responseMode: responseNode`.
- Validación de entrada: normalización, solicitud vacía (autodocumentación), barcode ausente o no numérico (400).
- Integración con Open Food Facts (sin API key).
- Verificación estricta de "producto encontrado" (`statusCode == 200 AND body.status == 1`), tras detectar que la API puede responder 200 con `status: 0`.
- Clasificación de doble lectura (semáforo + Nutri-Score, `Math.max`), con `insufficient_data` explícito.
- Enrutamiento condicional (`Switch`) a tres cadenas de IA independientes.
- Integración con Groq (`openai/gpt-oss-120b`) para el párrafo de veredicto.
- Formateo de respuesta en texto plano: titular con emoji, marcadores por nutriente, Nutri-Score, párrafo de IA.
- Manejo de errores estructurado en JSON para todos los caminos no exitosos.
- Documentación: README, test log en formato TC-XXX, este changelog.

### Fixed
- Modelo de IA no disponible: `llama-3.3-70b-versatile` devolvía `model_not_found`; reemplazado por `openai/gpt-oss-120b` en los tres nodos `lmChatGroq`. Ver TC-007.

### Known limitations
- Rama `insufficient_data` sin validar con caso real.
- Canvas sin agrupar (restricción de n8n), no afecta la ejecución.

## [0.2.0] — Pre-release: IA y respuesta final

### Added
- Bloque 4 (enrutamiento por IA) y bloque 5 (formateo y respuesta final).
- Primera ejecución end-to-end completa (con el bug de TC-007 aún presente).

## [0.1.0] — Pre-release: estructura inicial

### Added
- Creación inicial del workflow vía SDK de n8n.
- Bloque 1 (entrada y validación) y bloque 2 (Open Food Facts + clasificación), validados nodo por nodo.
- Bloque 3 (verificación de datos suficientes) añadido.
