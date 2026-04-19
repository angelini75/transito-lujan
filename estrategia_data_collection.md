Aquí tienes el documento en formato Markdown, estructurado con un nivel técnico profundo para su implementación directa.

Markdown

```
# Plan Operativo: Ingesta de Datos de Fricción Vial mediante Google Routes API
**Proyecto:** Diagnóstico de Tránsito Urbano · Luján, Buenos Aires
**Alcance:** Prueba de Concepto (PoC) operativa bajo capa gratuita (Free Tier).

---

## 1. Estrategia de Definición de Pares Origen-Destino (O-D)

Dado que no se puede extraer la capa de tráfico de forma masiva, la captura debe ser quirúrgica. La estrategia no consiste en mapear viajes largos (ej. "de casa al trabajo"), sino en establecer **Vectores de Segmento** que aíslen el comportamiento de los cuellos de botella geográficos e infraestructurales de la ciudad.



### 1.1. Criterios de Selección de Vectores
Para evitar que el algoritmo de ruteo de Google busque vías alternativas y distorsione la medición del tramo objetivo, los pares O-D deben ubicarse en los extremos exactos del tramo crítico, sin intersecciones mayores intermedias que ofrezcan rutas de escape equivalentes.

Se definirán un máximo de **25 a 30 vectores**, categorizados en tres tipologías:

* **Tipología A: Embudos Físicos (Puentes del Río Luján)**
    * *Objetivo:* Medir la demora de cruce.
    * *Ejemplos:* * Puente Dr. Muñiz (Vector: Desde rotonda Av. España hasta Bmé. Mitre).
        * Puente Almirante Brown.
        * Puente Las Heras.
* **Tipología B: Fricción por Barrera Urbana (Ferrocarril Sarmiento)**
    * *Objetivo:* Capturar la demora intermitente de las barreras bajas.
    * *Ejemplos:* * Paso a nivel de Lorenzo Casey / Beschtedt.
        * Paso a nivel de Bmé. Mitre y Av. España.
* **Tipología C: Corredores de Distribución Troncal**
    * *Objetivo:* Medir la onda verde (o falta de ella) y la saturación por flujo comercial.
    * *Ejemplos:*
        * Humberto I (Segmentado en dos vectores: desde Av. Pellegrini hasta San Martín, y de San Martín hasta Carlos Pellegrini).
        * Av. Nuestra Señora de Luján (Eje turístico/basilical).
        * San Martín / Constitución.

### 1.2. Coordenadas Estrictas
Cada nodo (Origen y Destino) debe definirse mediante coordenadas geográficas (Lat/Lon) con precisión de al menos 4 decimales. Estas coordenadas se almacenarán en un diccionario o tabla base que servirá de *seed* (semilla) para el orquestador.

---

## 2. Arquitectura de Ingesta Automatizada

La captura requiere un pipeline desatendido que se ejecute a intervalos regulares. 

### 2.1. Stack Tecnológico
* **Orquestador:** Contenedor Docker ejecutando `n8n` (Node-Based Workflow Automation).
* **API:** Google Routes API (Endpoint: `computeRoutes`).
* **Persistencia:** Base de datos PostgreSQL (idealmente con extensión PostGIS) o exportación iterativa a archivos `.csv` en un volumen montado de Docker.

### 2.2. Parámetros de la Petición HTTP (Optimización de Carga)
El nodo HTTP de n8n debe configurarse para consumir exclusivamente los tiempos de viaje, descartando geometrías completas (polylines) e indicaciones *turn-by-turn* para reducir latencia y tamaño de respuesta.

**Cabeceras Requeridas (Headers):**
* `Content-Type`: `application/json`
* `X-Goog-Api-Key`: `[TU_API_KEY]`
* `X-Goog-FieldMask`: `routes.duration,routes.staticDuration` *(Crítico para no procesar ni pagar por datos innecesarios).*

**Payload de Ejemplo (JSON):**
```json
{
  "origin": {
    "location": {
      "latLng": {
        "latitude": -34.5684,
        "longitude": -59.1234
      }
    }
  },
  "destination": {
    "location": {
      "latLng": {
        "latitude": -34.5712,
        "longitude": -59.1121
      }
    }
  },
  "travelMode": "DRIVE",
  "routingPreference": "TRAFFIC_AWARE"
}
```

------

## 3. Cronograma de Muestreo y Presupuesto

La capa gratuita de Google Maps Platform otorga USD 200 mensuales. El costo de la Routes API es de USD 5 por cada 1.000 peticiones. Esto establece un **límite estricto de 40.000 peticiones al mes (aprox. 1.300 diarias)**.

### 3.1. Diseño del Cron Trigger en n8n

Para capturar la variabilidad del tráfico optimizando el presupuesto, se configurará una matriz de disparos asimétrica:

- **Días Hábiles (Lunes a Viernes):**
  - Pico Mañana: 07:00, 07:30, 08:00, 08:30
  - Valle Mediodía: 12:00, 13:00, 14:00
  - Pico Tarde: 17:00, 17:30, 18:00, 18:30, 19:00
  - *Total: 12 mediciones x 30 vectores = 360 peticiones/día.*
- **Fines de Semana (Sábado y Domingo):**
  - La zona histórico-basilical altera el patrón. El muestreo debe intensificarse en el horario de 11:00 a 18:00 (una medición por hora).
  - *Total: 8 mediciones x 30 vectores = 240 peticiones/día.*

**Cálculo de Consumo Mensual:**

- Días hábiles (22 días): 7.920 peticiones.
- Fines de semana (8 días): 1.920 peticiones.
- **Total Mensual: 9.840 peticiones.**
- *Gasto equivalente:* USD 49,20.
- *Estado:* **Holgadamente dentro del límite gratuito de USD 200.**

------

## 4. Estructura de Persistencia y Pre-cálculo

El flujo de n8n no solo debe guardar el JSON en crudo, sino procesarlo e insertar una fila estructurada.

### 4.1. Esquema de Tabla de Captura

| **id_vector** | **timestamp**       | **is_weekend** | **duration_s** | **static_duration_s** | **friction_index** |
| ------------- | ------------------- | -------------- | -------------- | --------------------- | ------------------ |
| v_humberto_01 | 2026-04-20 08:30:00 | false          | 420            | 180                   | 1.33               |
| v_pte_muniz   | 2026-04-20 08:30:00 | false          | 150            | 45                    | 2.33               |

### 4.2. Cálculo del Índice de Fricción

La métrica derivada principal que alimentará el análisis de datos (Fase 2) es el Índice de Fricción:

$$Friccion = \frac{(duration\_s - static\_duration\_s)}{static\_duration\_s}$$

Un índice de `0` indica flujo libre perfecto. Un índice de `1.0` indica que el tiempo de viaje se duplicó debido a la congestión. Este índice es el valor que luego se interpolará sobre la red espacial descargada con `osmnx` para renderizar los mapas de calor en la herramienta de visualización.