Aquí tienes un prompt bien estructurado, pensado para obtener un análisis serio, basado en datos y con foco en toma de decisiones (no opiniones). Está diseñado para que quien lo ejecute trabaje casi como un equipo técnico de movilidad urbana.

------

### 📌 Prompt

**Rol:**
Eres un equipo multidisciplinario experto en movilidad urbana, análisis geoespacial, ciencia de datos y planificación del transporte, con experiencia en ciudades intermedias de Argentina. Actúas como consultor técnico independiente enfocado en diagnóstico basado en evidencia y diseño de intervenciones de alto impacto.

------

**Objetivo:**
Realizar un análisis integral del tránsito en Luján, Provincia de Buenos Aires, utilizando datos reales y proxies derivados de Google Traffic y otras fuentes, para:

1. Identificar ineficiencias en las medidas actuales de gestión del tránsito.
2. Evaluar si las decisiones existentes están respaldadas por datos o no.
3. Proponer intervenciones concretas para mejorar la fluidez del tránsito.
4. Identificar zonas de alto riesgo de accidentes (especialmente moto–auto).
5. Simular escenarios futuros mostrando cómo cambiaría el tránsito si se aplicaran las medidas propuestas.

------

**Tarea:**

Realizar un análisis estructurado en las siguientes etapas:

### 1. Recolección y estructuración de datos

Utilizar y/o simular el uso de las siguientes fuentes:

- **Google Traffic / Google Maps API**
  - Velocidad promedio por tramo
  - Nivel de congestión por franja horaria
  - Tiempos de viaje históricos
  - Variabilidad temporal (peak vs off-peak)
- **Datos complementarios**
  - OpenStreetMap (red vial, jerarquías, intersecciones)
  - Datos de accidentes (si disponibles; si no, estimación basada en patrones típicos)
  - Datos municipales (sentidos de calles, semáforos, cambios recientes)
  - Imágenes satelitales o street view (para validar geometría vial)
  - Datos de movilidad (ej: origen-destino si se pueden inferir)
  - Densidad poblacional y actividad económica (proxies)

------

### 2. Diagnóstico del sistema de tránsito

- Identificar:
  - Cuellos de botella principales
  - Intersecciones críticas
  - Calles subutilizadas vs sobrecargadas
  - Problemas de sincronización semafórica
  - Impacto de medidas recientes (cambios de sentido, cortes, etc.)
- Evaluar:
  - Si las decisiones actuales parecen basadas en evidencia o no
  - Posibles efectos negativos no anticipados

------

### 3. Análisis de ineficiencia

- Cuantificar:
  - Pérdida de tiempo promedio por conductor
  - Incremento de distancia recorrida por mala planificación
  - Ineficiencia en la red (ej: rutas no óptimas forzadas)
- Detectar:
  - Patrones de congestión evitables
  - Diseño vial inconsistente con la demanda real

------

### 4. Identificación de zonas de riesgo

- Detectar intersecciones con alta probabilidad de accidentes, especialmente moto–auto, considerando:
  - Alta congestión + giros conflictivos
  - Falta de visibilidad
  - Intersecciones sin control adecuado
  - Cambios de sentido recientes
- Clasificar zonas por nivel de riesgo:
  - Alto / Medio / Bajo

------

### 5. Propuesta de mejoras

Diseñar intervenciones concretas, como por ejemplo:

- Cambios de sentido de calles
- Optimización semafórica (coordinación, tiempos)
- Rediseño de intersecciones
- Carriles exclusivos o restricciones
- Medidas de pacificación de tráfico en zonas de riesgo
- Señalización y control

Cada propuesta debe incluir:

- Justificación basada en datos
- Impacto esperado
- Nivel de costo (bajo / medio / alto)
- Facilidad de implementación

------

### 6. Simulación de escenarios

- Modelar al menos 2–3 escenarios:
  - Escenario actual
  - Escenario con mejoras parciales
  - Escenario optimizado
- Mostrar:
  - Cambios en tiempos de viaje
  - Reducción de congestión
  - Reducción de riesgo de accidentes

------

**Entregables:**

### 1. 📄 Reporte técnico

- Diagnóstico detallado
- Metodología
- Análisis de datos
- Mapas y visualizaciones
- Recomendaciones priorizadas

------

### 2. 📊 Presentación ejecutiva

- Principales hallazgos
- Problemas críticos
- Soluciones propuestas
- Impacto esperado
- Mensaje claro para tomadores de decisión

------

### 3. 🌐 Página web interactiva

Debe incluir:

- Mapa interactivo de Luján con:
  - Niveles de congestión
  - Zonas de riesgo
  - Intervenciones propuestas
- Comparador de escenarios:
  - Antes vs después
  - Simulación de flujo de tránsito
- Visualizaciones:
  - Heatmaps
  - Tiempos de viaje
  - Indicadores clave
- Opcional:
  - Selector de medidas (encender/apagar intervenciones y ver impacto)

------

### 4. ⚠️ Módulo de seguridad vial

- Mapa de zonas de alto riesgo
- Tipología de accidentes probables (especialmente moto–auto)
- Medidas específicas de mitigación:
  - Infraestructura
  - Regulación
  - Educación

------

**Criterios clave:**

- Basar todo en datos o supuestos explícitos
- Evitar opiniones sin respaldo
- Priorizar claridad para la toma de decisiones
- Enfocar en soluciones implementables (no solo ideales)