# SnackCheck — Microservicio de Verificación Nutricional

SnackCheck es un flujo de automatización en n8n diseñado para analizar códigos de barras de productos alimenticios empaquetados, evaluar su calidad nutricional mediante una lógica combinada de semáforo de advertencias y Nutri-Score, y generar un veredicto accesible asistido por inteligencia artificial para consumidores.

---

## Propósito

El propósito de **SnackCheck** es ofrecer una herramienta rápida, objetiva y comprensible para evaluar la calidad nutricional de un alimento empaquetado directamente en el punto de compra. 

En lugar de abrumar al usuario con tablas técnicas de información nutricional, SnackCheck:
1. Consulta la base de datos abierta global **Open Food Facts**.
2. Aplica umbrales nutricionales validados (estilo semáforo nutricional de la FSA británica) para azúcares, sal y grasa.
3. Evalúa la calificación **Nutri-Score** oficial del producto.
4. Aplica el principio de seguridad preventiva: **"la peor lectura gana"**.
5. Emplea un modelo de lenguaje (LLM) a través de **Groq** para redactar una recomendación empática, clara y contextualizada en español, adaptando el tono según si el producto es saludable, moderado o no saludable.

---

## Cómo Funciona

El servicio opera como un pipeline determinista con enriquecimiento mediante IA, implementado íntegramente en n8n.

### Diagrama Lógico del Flujo

```
[POST /webhook/nutrition-check]
          │
          ├──> ¿Cuerpo vacío? ─── SÍ ───> [Respond - Self Documentation] (200 JSON)
          │
          └──> NO
                │
                ├──> ¿Barcode numérico? ─── NO ───> [Respond - Invalid Barcode] (400 JSON)
                │
                └──> SÍ
                      │
                      ▼
        [API Call - Open Food Facts]
                      │
                      ├──> ¿Existe producto? ─── NO ───> [Respond - Product Not Found] (404 JSON)
                      │
                      └──> SÍ
                            │
                            ├──> ¿Nutrientes presentes? ─── NO ───> [Respond - Insufficient Data] (200 JSON)
                            │
                            └──> SÍ
                                  │
                                  ▼
                    [Extract - Product & Nutriments]
                                  │
                                  ▼
                  [Classify - Traffic Light & Verdict]
                                  │
                                  ▼
                  [Code - Calculate Concern Score]
                                  │
                                  ▼
                        [Route - Verdict Tone]
                       /                      \
        (No saludable)                        (Saludable / Moderado)
             ▼                                          ▼
[AI - Health Verdict (Cautionary)]       [AI - Health Verdict (Encouraging)]
     [Groq: openai/gpt-oss-20b]               [Groq: openai/gpt-oss-20b]
                       \                      /
                        ▼                    ▼
                      [Merge - Combine Verdict Branches]
                                     │
                                     ▼
                      [Format - Final Verdict Message]
                                     │
                                     ▼
                      [Respond - Final Verdict Text] (200 text/plain)
```

### Reglas de Semáforo y Clasificación Nutricional

El nodo `Classify - Traffic Light & Verdict` clasifica los tres macronutrientes críticos por cada 100 g:

| Nutriente | Umbral Bajo (🟢) | Umbral Medio (🟡) | Umbral Alto (🔴) |
| :--- | :--- | :--- | :--- |
| **Azúcares (`sugars`)** | `< 5.0 g` | `5.0 g` a `22.5 g` | `> 22.5 g` |
| **Sal (`salt`)** | `≤ 1.5 g` | — | `> 1.5 g` |
| **Grasa (`fat`)** | `≤ 17.5 g` | — | `> 17.5 g` |

A partir de estas señales se calculan dos lecturas independientes:
1. **Lectura por Semáforos (`trafficReading`)**:
   - `0` alertas altas (🔴) $\rightarrow$ `saludable`
   - `1` alerta alta (🔴) $\rightarrow$ `moderado`
   - `≥ 2` alertas altas (🔴) $\rightarrow$ `no saludable`
2. **Lectura por Nutri-Score (`nutriscoreReading`)**:
   - `A` o `B` $\rightarrow$ `saludable`
   - `C` $\rightarrow$ `moderado`
   - `D` o `E` $\rightarrow$ `no saludable`

### Regla "La Peor Lectura Gana"
Para evitar que un producto con deficiencias graves pase desapercibido, el veredicto final selecciona la categoría más restrictiva entre ambas evaluaciones:
$$\text{Veredicto Final} = \min(\text{trafficReading}, \text{nutriscoreReading}) \quad \text{con orden } \text{No Saludable} > \text{Moderado} > \text{Saludable}$$

*Ejemplo*: Si una bebida carbonatada no dispara los semáforos individuales en 100 g porque su azúcar es del 10.9% (nivel medio 🟡), pero su Nutri-Score es **E**, el veredicto adoptado es **NO SALUDABLE**.

### Puntuación de Preocupación (`concernScore`)
El nodo `Code - Calculate Concern Score` evalúa cuántos de los tres nutrientes críticos están en nivel alto (🔴), arrojando una puntuación de `0/3`, `1/3`, `2/3` o `3/3`.

### Enrutamiento de Tono y Generación con Groq LLM
El nodo `Route - Verdict Tone` bifurca el flujo hacia:
- **Rama Cautelosa (`AI - Health Verdict (Cautionary)`)**: Para veredictos `NO SALUDABLE`. Solicita a Groq un párrafo corto (máximo 3 oraciones), empático y cercano, explicando qué nutriente dispara la alerta y sugiriendo consumirlo como gusto ocasional sin culpabilizar.
- **Rama Alentadora (`AI - Health Verdict (Encouraging)`)**: Para veredictos `SALUDABLE` y `MODERADO`. Destaca los aspectos positivos y, de ser moderado, menciona amablemente el punto de atención.

Ambas ramas usan el modelo `openai/gpt-oss-20b` orquestado a través de Groq.

### Nodos Utilizados en el Flujo n8n (Total: 21)
1. `Webhook - Nutrition Check` (`n8n-nodes-base.webhook`): Recibe peticiones HTTP POST.
2. `Validate - Empty Body` (`n8n-nodes-base.if`): Detecta solicitudes sin cuerpo.
3. `Respond - Self Documentation` (`n8n-nodes-base.respondToWebhook`): Devuelve la documentación JSON ante cuerpo vacío.
4. `Validate - Numeric Barcode` (`n8n-nodes-base.if`): Valida que `barcode` esté presente y sea puramente numérico (`^\d+$`).
5. `Respond - Invalid Barcode` (`n8n-nodes-base.respondToWebhook`): Retorna error 400 ante código inválido.
6. `API Call - Open Food Facts Product` (`n8n-nodes-base.httpRequest`): Petición GET a `https://world.openfoodfacts.org/api/v2/product/{barcode}.json`.
7. `Validate - Product Found` (`n8n-nodes-base.if`): Verifica si el producto fue hallado (`status == 1`).
8. `Respond - Product Not Found` (`n8n-nodes-base.respondToWebhook`): Retorna error 404 si el código no está registrado.
9. `Validate - Nutriments Present` (`n8n-nodes-base.if`): Comprueba que existan los datos de azúcar, sal y grasa.
10. `Respond - Insufficient Data` (`n8n-nodes-base.respondToWebhook`): Retorna estado informativo 200 `INSUFFICIENT_DATA`.
11. `Extract - Product & Nutriments` (`n8n-nodes-base.set`): Extrae y normaliza nombre, marca, Nutri-Score, azúcares, sal, grasa y calorías.
12. `Classify - Traffic Light & Verdict` (`n8n-nodes-base.set`): Computa las etiquetas de semáforo, lecturas y veredicto compuesto.
13. `Code - Calculate Concern Score` (`n8n-nodes-base.code`): Calcula la suma de advertencias de preocupación.
14. `Route - Verdict Tone` (`n8n-nodes-base.if`): Bifurca según veredicto hacia rama cautelosa o alentadora.
15. `AI - Health Verdict (Cautionary)` (`@n8n/n8n-nodes-langchain.chainLlm`): Cadena LLM para veredicto no saludable.
16. `Groq - Cautionary Model` (`@n8n/n8n-nodes-langchain.lmChatGroq`): Conexión con modelo Groq `openai/gpt-oss-20b`.
17. `AI - Health Verdict (Encouraging)` (`@n8n/n8n-nodes-langchain.chainLlm`): Cadena LLM para veredicto saludable o moderado.
18. `Groq - Encouraging Model` (`@n8n/n8n-nodes-langchain.lmChatGroq`): Conexión con modelo Groq `openai/gpt-oss-20b`.
19. `Merge - Combine Verdict Branches` (`n8n-nodes-base.merge`): Unifica los resultados de ambas ramas de IA.
20. `Format - Final Verdict Message` (`n8n-nodes-base.set`): Estructura la ficha de salida con emojis, datos y síntesis de IA.
21. `Respond - Final Verdict Text` (`n8n-nodes-base.respondToWebhook`): Entrega la respuesta final al cliente como texto plano.

---

## Configuración

### 1. Requisitos Previos
- Instancia activa de n8n (versión 1.0+ en n8n Cloud o auto-alojada).
- Cuenta en Groq Cloud con una clave de API válida (`gsk_...`).
- Acceso a Internet sin restricciones para conectar con `world.openfoodfacts.org` y `api.groq.com`.

### 2. Configuración de Credenciales
- En n8n, dirígete a **Credentials** $\rightarrow$ **New Credential** $\rightarrow$ **Groq API**.
- Asigna el nombre de la credencial y pega tu API Key de Groq.
- Vincula esta credencial en los nodos `Groq - Cautionary Model` y `Groq - Encouraging Model`.

### 3. Importación del Flujo
1. Abre tu interfaz de n8n y crea un nuevo workflow.
2. Haz clic en el menú superior derecho $\rightarrow$ **Import from File...**
3. Selecciona el archivo `snackcheck-workflow.json`.
4. Verifica que los nodos Groq tengan seleccionada tu credencial configurada.
5. Activa el flujo con el interruptor **Active** (o **Publish**).

### 4. URL del Webhook
El webhook queda expuesto en la siguiente URL de producción:
```
https://gaboq.app.n8n.cloud/webhook/nutrition-check
```

---

## Uso

### Método y Endpoint
- **Método**: `POST`
- **URL**: `https://gaboq.app.n8n.cloud/webhook/nutrition-check`
- **Content-Type**: `application/json`

### Ejemplo 1: Consulta de Producto (Nutella)

```bash
curl -X POST https://gaboq.app.n8n.cloud/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "3017620422003"}'
```

**Respuesta recibida (`200 OK`, `text/plain`):**
```text
🔴 VEREDICTO SNACKCHECK: NO SALUDABLE
Producto: Nutella (Ferrero, Nutella)
Nutri-Score: E | Puntuación de preocupación: 2/3

📊 NUTRIENTES POR 100 g:
• Azúcar: 56.3 g 🔴 (alto)
• Sal: 0.107 g 🟢 (bajo)
• Grasa: 30.9 g 🔴 (alto)
• Energía: 539 kcal

💬 OPINIÓN DE SNACKCHECK:
La Nutella tiene un Nutri‑Score e y contiene 56,3 g de azúcar y 30,9 g de grasa por cada 100 g, niveles que la sitúan en la categoría “no saludable”. Con estos valores altos de azúcar y grasa, es mejor considerarla un capricho ocasional más que un alimento diario. Disfrútala con moderación y acompáñala de opciones más equilibradas para mantener un buen balance nutricional.
```

### Ejemplo 2: Consulta de Producto Saludable (Skyr Danone)

```bash
curl -X POST https://gaboq.app.n8n.cloud/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "3033490004743"}'
```

**Respuesta recibida (`200 OK`, `text/plain`):**
```text
🟢 VEREDICTO SNACKCHECK: SALUDABLE
Producto: Skyr (Danone)
Nutri-Score: A | Puntuación de preocupación: 0/3

📊 NUTRIENTES POR 100 g:
• Azúcar: 3.9 g 🟢 (bajo)
• Sal: 0.09 g 🟢 (bajo)
• Grasa: 0.2 g 🟢 (bajo)
• Energía: 57 kcal

💬 OPINIÓN DE SNACKCHECK:
¡Este Skyr de Danone es una opción saludable con Nutri‑Score A! Contiene solo 3.9 g de azúcar, 0.09 g de sal y 0.2 g de grasa por cada 100 g, lo que lo hace perfecto para mantener una dieta equilibrada.
```

### Ejemplo 3: Petición sin cuerpo (Autodocumentación)

```bash
curl -X POST https://gaboq.app.n8n.cloud/webhook/nutrition-check
```

**Respuesta recibida (`200 OK`, `application/json`):**
```json
{
  "service": "SnackCheck API",
  "description": "Analiza el código de barras de un producto empaquetado usando Open Food Facts y devuelve un veredicto nutricional claro.",
  "method": "POST",
  "endpoint": "/nutrition-check",
  "required_fields": {
    "barcode": "Cadena o número con solo dígitos (ej. 3017620422003)"
  },
  "example": {
    "barcode": "3017620422003"
  }
}
```

---

## Manejo de Errores

SnackCheck implementa un manejo preventivo de excepciones en cada etapa crítica del flujo:

| Código HTTP | Error / Estado | Causa | Formato de Respuesta |
| :---: | :--- | :--- | :--- |
| **`200 OK`** | *Autodocumentación* | Se envió una petición `POST` con cuerpo vacío o sin parámetros. Permite el auto-descubrimiento de la API. | `application/json`<br>`{"service": "SnackCheck API", ...}` |
| **`400 Bad Request`** | `INVALID_BARCODE` | El campo `barcode` no fue provisto o contiene caracteres alfanuméricos/espacios no numéricos. | `application/json`<br>`{"error": "INVALID_BARCODE", "message": "...", "example": {"barcode": "3017620422003"}}` |
| **`404 Not Found`** | `PRODUCT_NOT_FOUND` | El código de barras es válido en formato pero no existe en la base de datos de Open Food Facts. | `application/json`<br>`{"error": "PRODUCT_NOT_FOUND", "message": "...", "example_valid_barcode": "3017620422003"}` |
| **`200 OK`** | `INSUFFICIENT_DATA` | El producto existe en Open Food Facts pero carece de valores declarados para `sugars_100g`, `salt_100g` o `fat_100g`. | `application/json`<br>`{"status": "INSUFFICIENT_DATA", "message": "El producto fue encontrado, pero carece de datos nutricionales suficientes..."}` |

---

## Limitaciones

1. **Dependencia de Open Food Facts**: La exactitud del análisis depende de la integridad de los datos cargados por los usuarios en Open Food Facts. Si un producto contiene errores tipográficos en el etiquetado original o no está registrado, el servicio no puede validar la información.
2. **Evaluación fija por 100 g**: Los umbrales de semáforo se aplican a 100 g de producto estándar. No consideran el tamaño real de la porción consumida (un producto moderado por 100 g puede convertirse en excesivo si la porción unitaria es de 250 g).
3. **Disponibilidad y Límites de Tasa**: Al consultar APIs externas (Open Food Facts y Groq API), la latencia de respuesta puede oscilar entre 0.4 s y 3.5 s. Si Open Food Facts aplica limitación de tasa (HTTP 503/429) o Groq agota su cuota por minuto, la solicitud fallará a nivel de transporte.
4. **Restricción de Método HTTP**: El endpoint webhook solo acepta solicitudes de tipo `POST`. Peticiones `GET`, `PUT` o `DELETE` son rechazadas por la pasarela de n8n.
