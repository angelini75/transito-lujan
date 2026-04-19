Tu propuesta está **muy bien orientada conceptualmente** y, sobre todo, tiene algo que rara vez aparece en planes municipales: una lógica clara de *“datos → diagnóstico → simulación → decisión”*. Eso ya es una ventaja enorme.

Dicho eso, si lo miro con ojos de implementación real (y con la restricción que vos mismo marcás: *prueba de concepto con datos disponibles*), hay tres planos donde vale la pena afinar bastante:

1. fuentes de datos,
2. solidez metodológica de la “simulación”,
3. narrativa estratégica para convencer al municipio.

Voy por partes, con foco práctico.

------

# 1. Evaluación de fuentes de datos (clave para la PoC)

Tu stack base es correcto, pero hay un punto crítico: **Google Distance Matrix como columna vertebral es frágil** para este tipo de proyecto.

### ✔️ Lo bueno

- OpenStreetMap + `osmnx`: excelente decisión
- ANSV: aporta valor (aunque incompleto)
- Muestreo temporal: bien planteado

### ⚠️ Problemas con Google (importantes)

- No tenés acceso real a histórico → solo snapshots
- Costos escalan rápido si aumentás cobertura espacial/temporal
- No sabés cómo Google estima tráfico (modelo opaco)
- Difícil defenderlo frente a un municipio (“¿de dónde sale esto?”)

👉 Para una PoC está bien, pero **no debería ser la única fuente de “verdad”**

------

## 💡 Alternativas de bajo costo (muy recomendadas)

Acá está una gran oportunidad para fortalecer mucho tu propuesta sin aumentar costos.

### 1. Datos de movilidad pasiva (muy potentes y subutilizados)

- **Datos de telefonía (si conseguís acceso institucional)**
  - No son gratuitos, pero a veces universidades o gobiernos los tienen
  - Sirven para OD matrices aproximadas
- **Google Popular Times (scraping indirecto)**
  - Proxy de actividad por zona/horario
  - Útil para inferir demanda

------

### 2. Datos abiertos en Argentina (infrautilizados)

- **SUBE (si hay acceso parcial o agregados)**
  - Flujos reales de movilidad
  - Muy valioso para validar demanda
- **INDEC + Censo 2022**
  - Ya lo mencionás, pero podés explotarlo más:
    - densidad por radio censal
    - tipo de movilidad laboral
- **Dirección Nacional de Vialidad**
  - Conteos en rutas cercanas
  - Sirve como condición de borde

------

### 3. Sensores “blandos” (casi gratis)

Esto es oro para una PoC:

- **Waze for Cities (Waze CCP)**
  - Gratis para gobiernos (podés simular acceso)
  - Incidentes, congestión, reportes en tiempo real
- **Strava Metro (limitado pero útil)**
  - Flujos de bicicletas
  - Indica corredores alternativos
- **Datos climáticos (SMN)**
  - Impacto en tránsito (lluvia → congestión)

------

### 4. Recolección propia (muy subestimada)

Acá podés diferenciarte mucho:

- Muestreo con celulares (tipo “floating car data” artesanal)
- 5–10 recorridos repetidos → calibración real
- Validación de Google (clave para credibilidad)

👉 Esto solo ya cambia la calidad del proyecto

------

# 2. Sobre la “simulación”: bien, pero cuidado cómo la vendés

Tu enfoque:

> networkx + shortest path + re-enrutamiento

### ✔️ Está perfecto para una PoC

Pero hay que ser muy claro:

👉 **No es una simulación de tránsito, es una simulación de rutas óptimas**

------

## ⚠️ Riesgo conceptual

Podrías sobreestimar impacto porque:

- No modelás:
  - interacción entre vehículos
  - semáforos dinámicos
  - colas
  - saturación

------

## 💡 Mejora clave (sin complejizar demasiado)

En lugar de venderlo como simulación:

👉 Reformular como:

**“modelo de accesibilidad y eficiencia de red”**

Y agregar:

- penalización por congestión (weights dinámicos)
- capacidad teórica por tipo de vía
- escenarios con saturación simplificada

------

## 🚀 Si querés subir un nivel (sin ir a SUMO)

Podrías incorporar:

- **Volume-delay functions (tipo BPR)**
  → ya te mete en transporte serio sin microsimulación

------

# 3. Diagnóstico: muy sólido, pero podés profundizar

Tu diagnóstico es bueno, pero le falta algo clave para convencer:

👉 **medir ineficiencia estructural de la red**

------

## 💡 Métricas que te conviene agregar

- **Circuidad (ratio distancia real / distancia ideal)**
- **Redundancia de rutas**
- **Centralidad de la red (betweenness)**
- **Dependencia de nodos críticos**

Esto te permite decir cosas como:

> “El 35% del tráfico depende de solo 5 intersecciones”

Eso impacta mucho políticamente.

------

# 4. Propuestas: bien, pero falta priorización estratégica

Tus intervenciones están bien, pero:

👉 hoy parecen un listado técnico, no una estrategia

------

## 💡 Mejora clave

Agregar una capa de priorización:

- impacto (alto/medio/bajo)
- costo
- tiempo de implementación
- riesgo político

Y generar:

👉 **“Top 5 intervenciones de alto impacto y bajo costo”**

Eso es lo que un municipio realmente usa.

------

# 5. Narrativa (esto es CLAVE para la PoC)

Vos ya lo dijiste implícitamente:

> “mostrar que con más datos se puede hacer mucho mejor”

Pero eso hay que hacerlo explícito en el diseño.

------

## 💡 Te recomiendo estructurarlo así

### Escenario A — con datos limitados (actual PoC)

- incertidumbre alta
- resultados indicativos
- decisiones con riesgo

### Escenario B — con datos enriquecidos

- menor incertidumbre
- mejores predicciones
- priorización óptima

👉 Mostrar la diferencia es lo que vende la inversión

------

# 6. Aspectos a mejorar (síntesis clara)

Te los dejo bien concretos:

### 🔴 Críticos

- No depender tanto de Google
- Aclarar límites de la simulación
- Validación mínima con datos reales

### 🟡 Importantes

- Incorporar métricas de red (no solo velocidad)
- Mejorar priorización de intervenciones
- Incluir análisis de demanda (aunque sea proxy)

### 🟢 Diferenciales (te pueden destacar mucho)

- Recolección propia de datos
- Comparación “con vs sin datos”
- Métricas de accesibilidad urbana

------

# 7. Conclusión honesta

Tu plan:

👉 **es totalmente viable como prueba de concepto**
👉 **está mejor estructurado que muchos estudios reales**

Pero si lo ejecutás tal cual:

- va a ser convincente técnicamente
- pero **puede quedar débil políticamente**

Si incorporás estas mejoras:

👉 pasa de “buen análisis” a **herramienta de decisión real**

