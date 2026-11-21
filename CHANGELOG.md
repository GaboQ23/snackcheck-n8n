# Registro de Cambios (Changelog)

Todos los cambios notables en este proyecto están documentados en este archivo.
El formato se basa en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/) y este proyecto se adhiere a [Semantic Versioning](https://semver.org/lang/es/).

---

## [1.0.0] - 2026-11-21

### Añadido
- **Flujo de automatización n8n (`snackcheck-workflow.json`)**:
  - Endpoint webhook en producción expuesto en `POST /webhook/nutrition-check`.
  - Integración síncrona con la API pública de **Open Food Facts** (v2) para consulta de datos nutricionales y etiquetas por código de barras.
  - Lógica de clasificación nutricional dual basada en:
    - Umbrales de semáforo de la FSA británica para azúcares (>22.5g alto, >=5g medio, <5g bajo), sal (>1.5g alto) y grasa (>17.5g alto).
    - Calificación oficial de Nutri-Score (grados A, B, C, D, E).
  - Implementación de la regla de seguridad preventiva **"la peor lectura gana"**, asegurando que alimentos con advertencias severas o Nutri-Score desfavorable no pasen desapercibidos.
  - Cálculo de la puntuación de preocupación acumulada (`concernScore` de 0 a 3).
  - Enrutamiento dinámico de tono (`Route - Verdict Tone`):
    - Rama cautelosa (`Cautionary`) para productos con veredicto NO SALUDABLE.
    - Rama alentadora (`Encouraging`) para productos con veredicto SALUDABLE o MODERADO.
  - Integración de inferencia con **Groq** utilizando el modelo `openai/gpt-oss-20b` para la generación de recomendaciones en lenguaje natural empático.
  - Formato de respuesta legible por humanos (`text/plain`) con indicadores visuales emoji (🔴, 🟡, 🟢), desglose de macronutrientes por 100 g y opinión contextualizada.
- **Manejo integral de errores y autodescubrimiento**:
  - Autodocumentación interactiva (`200 OK`, JSON) ante solicitudes con cuerpo vacío.
  - Validación estricta con regex numérico (`400 Bad Request`, `INVALID_BARCODE`) para códigos no numéricos o ausentes.
  - Manejo de productos no encontrados en la base de datos externa (`404 Not Found`, `PRODUCT_NOT_FOUND`).
  - Detección de productos con macronutrientes incompletos (`200 OK`, `INSUFFICIENT_DATA`).
- **Documentación y pruebas de certificación**:
  - `README.md` completo con arquitectura, nodos del flujo, configuración, ejemplos de uso, tabla de errores y limitaciones.
  - `TEST_CASES.md` con matriz de 9 casos de prueba ejecutados en vivo (`TC-001` a `TC-009`) y métricas reales de rendimiento.
  - Diagrama visual del flujo `snackcheck-diagram.excalidraw`.
