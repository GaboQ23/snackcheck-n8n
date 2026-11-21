# Matriz y Detalle de Casos de Prueba — SnackCheck API

Este documento certifica la ejecución real y exhaustiva de la batería de pruebas para el microservicio **SnackCheck** (`https://gaboq.app.n8n.cloud/webhook/nutrition-check`), ejecutada el **21 de noviembre de 2026**.

---

## Tabla Resumen de Ejecución (TC-001 a TC-009)

| Caso | Categoría | Descripción | Payload Enviado | HTTP Esperado | HTTP Real | Tiempo (s) | Estado |
| :---: | :--- | :--- | :--- | :---: | :---: | :---: | :---: |
| **TC-001** | Funcional | Nutri-Score E cauteloso (Nutella) | `{"barcode": "3017620422003"}` | `200` | `200` | **3.02 s** | **APROBADO** |
| **TC-002** | Funcional | Peor lectura gana (Coca-Cola) | `{"barcode": "5449000000996"}` | `200` | `200` | **1.56 s** | **APROBADO** |
| **TC-003** | Funcional | Nutri-Score / Tono alentador (Cruesli Quaker) | `{"barcode": "3168930010265"}` | `200` | `200` | **1.43 s** | **APROBADO** |
| **TC-004** | Integración | Open Food Facts + Groq en vivo (Galletas Prince LU) | `{"barcode": "7622210449283"}` | `200` | `200` | **1.17 s** | **APROBADO** |
| **TC-005** | Error / Validación | Cuerpo vacío (Autodocumentación) | *Sin cuerpo (POST vacío)* | `200` | `200` | **0.77 s** | **APROBADO** |
| **TC-006** | Error / Validación | Código no numérico | `{"barcode": "abc123"}` | `400` | `400` | **0.53 s** | **APROBADO** |
| **TC-007** | Error / Integración | Producto no encontrado en base de datos | `{"barcode": "0000000000000"}` | `404` | `404` | **0.50 s** | **APROBADO** |
| **TC-008** | Error / Integración | Producto sin datos nutricionales mínimos | `{"barcode": "6111246651261"}` | `200` | `200` | **0.42 s** | **APROBADO** |
| **TC-009** | Funcional | Nutri-Score A 100% Saludable (Skyr Danone) | `{"barcode": "3033490004743"}` | `200` | `200` | **1.38 s** | **APROBADO** |

---

## Detalle Exhaustivo de Pruebas

### TC-001: Funcional — Nutri-Score E Cauteloso (Nutella)
- **Tipo**: Funcional.
- **Objetivo**: Verificar que un producto alto en azúcares y grasas, clasificado con Nutri-Score E, sea calificado como `NO SALUDABLE`, active la rama cautelosa de IA y devuelva la ficha completa en `text/plain`.
- **Payload**:
  ```json
  {
    "barcode": "3017620422003"
  }
  ```
- **Tiempo de Respuesta**: **3.02 s**
- **Código HTTP**: `200 OK` (`Content-Type: text/plain; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
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
- **Conclusión**: Tanto el azúcar (56.3 g > 22.5 g) como la grasa (30.9 g > 17.5 g) disparan alertas rojas (`2/3`). Sumado al Nutri-Score E, el veredicto es `NO SALUDABLE`. Groq generó el consejo con tono empático y cauteloso sin juzgar al usuario.

---

### TC-002: Funcional — Peor Lectura Gana (Coca-Cola Original)
- **Tipo**: Funcional.
- **Objetivo**: Validar el principio de seguridad preventiva ("la peor lectura gana"). En este caso, la Coca-Cola no alcanza el umbral de azúcar alto por 100 g (> 22.5 g), resultando en `0/3` banderas rojas en semáforos, pero su Nutri-Score oficial es **E**. El veredicto final debe adoptar forzosamente `NO SALUDABLE`.
- **Payload**:
  ```json
  {
    "barcode": "5449000000996"
  }
  ```
- **Tiempo de Respuesta**: **1.56 s**
- **Código HTTP**: `200 OK` (`Content-Type: text/plain; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```text
  🔴 VEREDICTO SNACKCHECK: NO SALUDABLE
  Producto: Coca-Cola Original (COCA-COLA SERVICES SA/NV)
  Nutri-Score: E | Puntuación de preocupación: 0/3

  📊 NUTRIENTES POR 100 g:
  • Azúcar: 10.9 g 🟡 (medio)
  • Sal: 0 g 🟢 (bajo)
  • Grasa: 0 g 🟢 (bajo)
  • Energía: 44 kcal

  💬 OPINIÓN DE SNACKCHECK:
  El Coca‑Cola Original tiene un Nutri‑Score **e** y contiene **10,9 g de azúcar por cada 100 g**, lo cual es un nivel medio y la razón principal por la que se considera no saludable (el azúcar es el nutriente que dispara la alerta). Si buscas algo más ligero, podrías optar por una bebida con menos azúcar, pero si te apetece, disfrútala como un gusto ocasional sin sentirte culpable.
  ```
- **Conclusión**: Regla validada con éxito. Aunque la puntuación de preocupación es `0/3`, el Nutri-Score E fuerza el veredicto final a `NO SALUDABLE`.

---

### TC-003: Funcional — Nutri-Score B con Punto de Atención (Cruesli Quaker)
- **Tipo**: Funcional.
- **Objetivo**: Evaluar el comportamiento frente a un cereal de desayuno con Nutri-Score B y contenido graso que supera el umbral de semáforo (> 17.5 g), verificando la alerta en grasa y el enrutamiento hacia la rama alentadora.
- **Payload**:
  ```json
  {
    "barcode": "3168930010265"
  }
  ```
- **Tiempo de Respuesta**: **1.43 s**
- **Código HTTP**: `200 OK` (`Content-Type: text/plain; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```text
  🟡 VEREDICTO SNACKCHECK: MODERADO
  Producto: cruesly mélange de noix (Quaker, Cruesli)
  Nutri-Score: B | Puntuación de preocupación: 1/3

  📊 NUTRIENTES POR 100 g:
  • Azúcar: 12 g 🟡 (medio)
  • Sal: 0 g 🟢 (bajo)
  • Grasa: 19 g 🔴 (alto)
  • Energía: 462 kcal

  💬 OPINIÓN DE SNACKCHECK:
  ¡Este cruesly mélange de noix de Quaker es una opción deliciosa y nutritiva! Con 12 g de azúcar por cada 100 g, cero gramos de sal y 19 g de grasa, ofrece una buena fuente de energía y sabor sin exceso de sodio. Solo recuerda que su contenido de grasa es algo elevado, así que disfrútalo con moderación para mantener un equilibrio saludable.
  ```
- **Conclusión**: El contenido de grasa (19 g > 17.5 g) activa la bandera roja en el semáforo nutricional (`Grasa: 🔴 (alto)`, puntuación de preocupación 1/3), determinando el veredicto `MODERADO` a pesar del Nutri-Score B. El LLM redacta con tono positivo destacando las bondades del cereal y señalando el punto a vigilar.

---

### TC-004: Integración — Open Food Facts + Groq en Vivo (Galletas Prince LU)
- **Tipo**: Integración extremo a extremo.
- **Objetivo**: Confirmar la integración síncrona en tiempo real entre el webhook de n8n, la API REST de Open Food Facts y los modelos de inferencia de Groq (`openai/gpt-oss-20b`).
- **Payload**:
  ```json
  {
    "barcode": "7622210449283"
  }
  ```
- **Tiempo de Respuesta**: **1.17 s**
- **Código HTTP**: `200 OK` (`Content-Type: text/plain; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```text
  🔴 VEREDICTO SNACKCHECK: NO SALUDABLE
  Producto: Prince (LU, Prince)
  Nutri-Score: E | Puntuación de preocupación: 1/3

  📊 NUTRIENTES POR 100 g:
  • Azúcar: 32 g 🔴 (alto)
  • Sal: 0.49 g 🟢 (bajo)
  • Grasa: 17 g 🟢 (bajo)
  • Energía: 467 kcal

  💬 OPINIÓN DE SNACKCHECK:
  El Prince (LU, Prince) lleva un Nutri‑Score e y, aunque su grasa y sal son bajas, el azúcar está bastante alto (32 g cada 100 g), lo que lo sitúa en la categoría no saludable. No es malo, solo que lo mejor es consumirlo de vez en cuando como un capricho. Así podrás disfrutarlo sin sentirte culpable y mantener un equilibrio nutricional.
  ```
- **Conclusión**: Integración completada con éxito. Tiempo de ejecución sobresaliente de 1.17 s para consulta de API externa y generación de lenguaje natural.

---

### TC-005: Error / Manejo — Petición POST sin Cuerpo (Autodocumentación)
- **Tipo**: Manejo de Errores / Descubrimiento.
- **Objetivo**: Asegurar que una llamada `POST` sin cuerpo no genere un fallo 500, sino una respuesta amigable `200 OK` con la especificación y ejemplo de uso de la API.
- **Payload**: Ninguno (cuerpo vacío `b""`).
- **Tiempo de Respuesta**: **0.77 s**
- **Código HTTP**: `200 OK` (`Content-Type: application/json; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
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
- **Conclusión**: El nodo `Validate - Empty Body` capturó la petición y entregó la documentación estructurada.

---

### TC-006: Error / Validación — Código de Barras No Numérico
- **Tipo**: Validación de Entrada.
- **Objetivo**: Verificar que caracteres alfanuméricos sean rechazados inmediatamente con código `400 Bad Request` antes de realizar llamadas innecesarias a Open Food Facts.
- **Payload**:
  ```json
  {
    "barcode": "abc123"
  }
  ```
- **Tiempo de Respuesta**: **0.53 s**
- **Código HTTP**: `400 Bad Request` (`Content-Type: application/json; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```json
  {
    "error": "INVALID_BARCODE",
    "message": "El campo 'barcode' es obligatorio y debe ser estrictamente numérico.",
    "example": {
      "barcode": "3017620422003"
    }
  }
  ```
- **Conclusión**: El regex `^\d+$` del nodo `Validate - Numeric Barcode` bloqueó la entrada correctamente.

---

### TC-007: Error / Integración — Producto No Encontrado (404)
- **Tipo**: Manejo de Errores de API Externa.
- **Objetivo**: Comprobar el manejo de códigos de barras válidos en sintaxis pero inexistentes en la base de datos de Open Food Facts.
- **Payload**:
  ```json
  {
    "barcode": "0000000000000"
  }
  ```
- **Tiempo de Respuesta**: **0.50 s**
- **Código HTTP**: `404 Not Found` (`Content-Type: application/json; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```json
  {
    "error": "PRODUCT_NOT_FOUND",
    "message": "El producto no existe en la base de datos de Open Food Facts.",
    "example_valid_barcode": "3017620422003"
  }
  ```
- **Conclusión**: El nodo `Validate - Product Found` detectó `status == 0` y emitió el error 404 tipificado.

---

### TC-008: Error / Integración — Producto con Datos Nutricionales Insuficientes
- **Tipo**: Manejo de Errores de Datos Faltantes.
- **Objetivo**: Comprobar el comportamiento cuando el producto está registrado en Open Food Facts pero carece de la información obligatoria de macronutrientes (`sugars_100g`, `salt_100g`, `fat_100g`).
- **Payload**:
  ```json
  {
    "barcode": "6111246651261"
  }
  ```
  *(Código real identificado en Open Food Facts bajo el estado `en:nutrition-facts-to-be-completed`).*
- **Tiempo de Respuesta**: **0.42 s**
- **Código HTTP**: `200 OK` (`Content-Type: application/json; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
  ```json
  {
    "status": "INSUFFICIENT_DATA",
    "message": "El producto fue encontrado, pero carece de datos nutricionales suficientes (azúcar, sal o grasa por 100 g) para emitir un veredicto."
  }
  ```
- **Conclusión**: El nodo `Validate - Nutriments Present` evitó un fallo de cálculo posterior o alucinaciones en el LLM, respondiendo con el estado informativo `INSUFFICIENT_DATA`.

---

### TC-009: Funcional — Veredicto 100% Saludable con Nutri-Score A (Skyr Danone)
- **Tipo**: Funcional.
- **Objetivo**: Verificar el flujo completo con un producto óptimo nutricionalmente: Nutri-Score A, todas las señales en verde (azúcar bajo, sal baja, grasa baja) y puntuación de preocupación `0/3`.
- **Payload**:
  ```json
  {
    "barcode": "3033490004743"
  }
  ```
- **Tiempo de Respuesta**: **1.38 s**
- **Código HTTP**: `200 OK` (`Content-Type: text/plain; charset=utf-8`)
- **Cuerpo de Respuesta Obtenido**:
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
  ¡Este Skyr de Danone es una opción saludable con Nutri‑Score A! Contiene solo 3.9 g de azúcar, 0.09 g de sal y 0.2 g de grasa por cada 100 g, lo que lo hace perfecto para mantener una dieta equilibrada. Aprovecha su textura cremosa y su alto contenido de proteína para un snack reconfortante y bajo en calorías.
  ```
- **Conclusión**: Veredicto `🟢 SALUDABLE` confirmado. La IA utilizó la plantilla de tono positivo y alentador.

---

## Análisis de Rendimiento y Tiempos de Respuesta

Los tiempos de respuesta fueron medidos de extremo a extremo (round-trip time desde el cliente hacia el webhook de n8n):

```
TC-008 (Datos insuficientes):      0.42 s  ████▌
TC-007 (404 No encontrado):        0.50 s  █████
TC-006 (400 Barcode inválido):     0.53 s  █████▍
TC-005 (Autodocumentación):        0.77 s  ████████
TC-004 (Galletas Prince LU + LLM): 1.17 s  ████████████
TC-009 (Skyr Danone + LLM):        1.38 s  ██████████████
TC-003 (Cruesli Quaker + LLM):     1.43 s  ██████████████▍
TC-002 (Coca-Cola + LLM):          1.56 s  ████████████████
TC-001 (Nutella + LLM):            3.02 s  ██████████████████████████████
```

### Estadísticas Clave:
- **Tiempo Mínimo**: **0.42 s** (`TC-008`, salida temprana sin invocación a LLM).
- **Tiempo Máximo**: **3.02 s** (`TC-001`, cold-start inicial de la conexión Groq / Open Food Facts).
- **Tiempo Promedio de Rutas Cortas (Validación y Errores)**: **0.55 s** (`TC-005`, `TC-006`, `TC-007`, `TC-008`).
- **Tiempo Promedio de Rutas con IA (Groq LLM + OFF)**: **1.71 s** (`TC-001`, `TC-002`, `TC-003`, `TC-004`, `TC-009`).
- **Tasa de Éxito**: **100% (9 de 9 pruebas aprobadas)**.
