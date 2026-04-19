Voy a explorar el directorio y leer el archivo.

------

# Nota Conceptual — Análisis de Tránsito Urbano · Luján, Buenos Aires

## ¿Qué es este proyecto?

Un sistema de **diagnóstico y propuesta de mejoras de tránsito** para Luján (ciudad intermedia, ~120 mil hab.), que combina análisis geoespacial, ciencia de datos y visualización interactiva. El producto final no es un estudio PDF: es una **plataforma web con mapas vivos y simulación de escenarios** orientada a tomadores de decisiones municipales.

------

## Componentes del sistema

```
[Fuentes de datos] → [Motor de análisis] → [Entregables]
```

### Fuentes de datos (Capa de ingestión)

| Fuente                            | Qué aporta                                            | Disponibilidad                     |
| --------------------------------- | ----------------------------------------------------- | ---------------------------------- |
| OpenStreetMap                     | Red vial, jerarquía de calles, intersecciones         | Libre, descargable con `osmnx`     |
| Google Maps / Distance Matrix API | Tiempos de viaje, congestión por franja horaria       | API paga (~gratis en volumen bajo) |
| Google Street View                | Validación de geometría vial, señalización            | API paga                           |
| SINIESTROS VIALES (ANSV/DNVN)     | Accidentes geo-referenciados                          | Datos abiertos Argentina           |
| Municipio de Luján                | Sentidos de circulación, semáforos, cambios recientes | Gestión directa                    |
| Densidad poblacional              | Proxy de demanda de viajes                            | INDEC censo 2022                   |

> **Riesgo principal:** Google Traffic API no expone datos históricos directamente. La estrategia es combinar **Distance Matrix API** (muestreo temporal programado) + **osmnx** para la red y usar datos de ANSV como ancla de seguridad vial.

------

## Arquitectura técnica

```
Python (osmnx, geopandas, pandas)
  └── Análisis de red + diagnóstico → GeoJSON / JSON
  └── Modelo de riesgo → scoring por intersección

JavaScript / Leaflet.js o Mapbox GL
  └── Página web interactiva
      ├── Mapa de congestión (heatmap)
      ├── Mapa de riesgo vial (puntos clasificados)
      ├── Comparador antes/después por intervención
      └── Panel de indicadores (KPIs)

Datos → archivos estáticos (GeoJSON + CSV)
  No requiere backend — desplegable en GitHub Pages / Netlify
```

------

## Plan de implementación — 5 fases

### Fase 1 · Datos base (≈ 1 semana)

- Descargar red vial de Luján con `osmnx`
- Recolectar tiempos de viaje reales con Distance Matrix API (muestreo en horarios pico / valle: 7–9h, 12–14h, 17–19h durante 5 días)
- Obtener datos de siniestros ANSV y georreferenciarlos sobre la red
- Levantar información municipal: cambios de sentido recientes, ubicación de semáforos

### Fase 2 · Diagnóstico (≈ 1 semana)

- Calcular velocidad promedio por tramo (v = distancia / tiempo_viaje)
- Identificar **cuellos de botella**: tramos con v < 15 km/h en horario pico
- Detectar **intersecciones críticas**: cruces con alta densidad de siniestros + alta congestión
- Evaluar si medidas recientes (cambios de sentido) mejoraron o empeoraron métricas

### Fase 3 · Modelo de riesgo vial (≈ 3 días)

- Scoring por intersección: congestión + siniestros históricos + geometría conflictiva + cambios recientes
- Clasificación Alto / Medio / Bajo
- Foco en conflictos moto–auto (ángulo de cruce, velocidad, visibilidad)

### Fase 4 · Propuestas + simulación (≈ 1 semana)

- Diseñar 3–5 intervenciones concretas con ficha: datos que la justifican, costo, impacto esperado
- Simular en red OSM: modelar cambio de sentido o nuevo semáforo → recalcular rutas óptimas → cuantificar delta en tiempo total
- Generar 3 escenarios: actual / mejoras parciales / optimizado

### Fase 5 · Entregables (≈ 1 semana)

- Reporte técnico (Markdown → PDF)
- Presentación ejecutiva (10 slides, foco en decisores)
- **Página web interactiva** (estática, Leaflet + JS puro, sin backend)
- Módulo de seguridad vial integrado en el mapa

------

## Stack recomendado

| Componente             | Herramienta                                       | Por qué                                             |
| ---------------------- | ------------------------------------------------- | --------------------------------------------------- |
| Análisis de red        | `osmnx` + `networkx`                              | Estándar para análisis vial en Python               |
| Geoespacial            | `geopandas` + `shapely`                           | Manejo de GeoJSON y operaciones espaciales          |
| Visualización estática | `matplotlib` + `folium`                           | Generación de mapas para el reporte                 |
| Web interactiva        | `Leaflet.js`                                      | Mapas ligeros sin backend, funciona en GitHub Pages |
| Simulación de flujos   | `networkx` shortest path + re-enrutamiento manual | Suficiente para ciudad intermedia; no requiere SUMO |
| Reporte                | Quarto o Jupyter Book                             | Reproducible, exporta a PDF y HTML                  |

------

## Supuestos y límites explícitos

1. **Sin datos de tránsito en tiempo real histórico completo** → se usa muestreo programado + proxies OSM para calibrar
2. **Simulación de flujos es simplificada** (no microsimulación tipo SUMO): muestra dirección del cambio, no valores exactos
3. **Datos de siniestros ANSV pueden tener subregistro** → se documenta explícitamente como limitación
4. El análisis es diagnóstico + propuesta; la implementación de medidas es responsabilidad municipal

------

## Próximos pasos concretos

1. **¿Tenés acceso a datos municipales?** (sentidos actuales, semáforos, cambios recientes) → es el insumo crítico
2. **¿Hay un período de toma de datos?** → para programar el muestreo con Google API
3. **¿La web debe ser pública o para uso interno?** → define hosting y autenticación
4. Confirmar si querés arrancamos con la descarga de la red OSM y el análisis base ahora

------

El proyecto es ambicioso pero completamente ejecutable. La parte más valiosa y diferencial es la **página web interactiva con comparador de escenarios** — eso es lo que convierte un informe técnico en una herramienta de decisión real.