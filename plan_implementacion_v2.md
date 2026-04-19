# Plan de Implementación v2 — Diagnóstico de Tránsito Urbano · Luján, Buenos Aires
**Versión:** 2.0  
**Fecha:** 2026-04-19  
**Clasificación:** Prueba de Concepto (PoC) técnica

---

## 1. Resumen Ejecutivo

Este documento integra las directrices del Experto 1 (Memorándum Técnico Operativo), las observaciones metodológicas del Experto 2 y la estrategia detallada de recolección de datos, traducidas en un plan de ejecución concreto que **aprovecha la infraestructura ya desplegada** en la VM `small-20250401-n8n` (GCP, southamerica-east1-c).

**Objetivo del proyecto:** Construir una herramienta de diagnóstico y simulación vial para el casco urbano de Luján que opere con datos reales, muestre sus propias limitaciones de forma transparente, y sea el argumento técnico para una inversión municipal de segunda fase.

**Lo que lo diferencia de un informe técnico estándar:**
- Pipeline de datos automatizado que sigue recolectando datos en producción
- Página web interactiva con comparador de escenarios
- Honestidad metodológica explícita: las limitaciones son parte del argumento de venta
- Métricas de red (no solo velocidad) que permiten frases con impacto político

---

## 2. Infraestructura Disponible — Inventario Real

> Exploración realizada el 2026-04-19 sobre la VM `small-20250401-n8n`.

### Servicios Docker activos en `/opt/mi-stack/`

| Contenedor | Imagen | URL/Puerto | Rol en el proyecto |
|---|---|---|---|
| `traefik` | traefik:v2.10 | :80/:443/:8080 | **Proxy + SSL automático** → no requiere config manual |
| `n8n` | docker.n8n.io/n8nio/n8n | `n8n.soildecisions.com` | **Orquestador del pipeline de recolección** |
| `leaflet_server` | nginx + Leaflet | `map.soildecisions.com` | **Candidato para el mapa interactivo** |
| `postgres_iuser` | postgres:15-alpine | interno :5432 | **Base de datos de fricción vial** |
| `geoenv_platform` | geoenv-geoenv | interno :8000 | **Entorno Python geoespacial** |
| `validation_dashboard` | Node.js Express | `validate.soildecisions.com` | Referencia de arquitectura para dashboard |

### Decisión de deployment

La web interactiva del proyecto se desplegará como **nuevo subdominio `transito.soildecisions.com`**, siguiendo exactamente el patrón del `leaflet_server` ya funcional:

```yaml
# Agregar al docker-compose.yml existente:
transito_server:
  image: nginx:alpine
  container_name: transito_server
  volumes:
    - ./transito-app:/usr/share/nginx/html:ro
  networks:
    - webnet
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.transito-secure.rule=Host(`transito.soildecisions.com`)"
    - "traefik.http.routers.transito-secure.entrypoints=websecure"
    - "traefik.http.routers.transito-secure.tls.certresolver=myresolver"
    - "traefik.http.services.transito.loadbalancer.server.port=80"
    - "traefik.docker.network=mi-stack_webnet"
```

**SSL, dominio y proxy son gratuitos y automáticos via Traefik.** Sin costo de hosting adicional.

---

## 3. Arquitectura del Sistema

```
[Google Routes API]         [OpenStreetMap]        [ANSV / Scraping noticias]
       │                          │                          │
       ▼                          ▼                          ▼
  n8n (n8n.soildecisions.com)  ── osmnx/geopandas ──  BeautifulSoup
  (cron: 12x/día L-V, 8x/día S-D)   (en geoenv_platform:8000)
       │                          │
       ▼                          ▼
  PostgreSQL: esquema transito_db (postgres_iuser)
  Tablas: friction_raw, od_vectors, red_vial, riesgo_intersecciones
       │
       ▼
  Scripts Python (geoenv_platform)
  → Índice de Fricción por segmento
  → Betweenness Centrality
  → Modelo de Riesgo Vial
  → Exportación a GeoJSON
       │
       ▼
  /opt/mi-stack/transito-app/  (archivos estáticos)
  → Leaflet.js + Chart.js
  → transito.soildecisions.com (Traefik + SSL automático)
```

**Principio rector:** Todo el frontend es estático (HTML + GeoJSON). Sin API calls en producción. Toda la lógica pesada ocurre en el backend durante la fase de análisis.

---

## 4. Datos y Fuentes — Stack Validado

### 4.1. Fuente primaria: Google Routes API (gratis)

Adoptando la estrategia técnica del Experto 1 y el documento de recolección:

- **Endpoint:** `POST https://routes.googleapis.com/directions/v2:computeRoutes`
- **Campo crítico del header:** `X-Goog-FieldMask: routes.duration,routes.staticDuration`  
  → Evita cobrar por geometrías y pasos de giro. Solo se paga por lo que se usa.
- **Costo calculado:** ~USD 49 /mes → dentro del crédito gratuito de USD 200.

### 4.2. Los 30 Vectores Origen-Destino de Luján

Organizados en tres tipologías según el documento de recolección:

**Tipología A — Embudos Físicos (Puentes del Río Luján)**

| ID Vector | Tramo | Lógica |
|---|---|---|
| `v_pte_muniz` | Rotonda Av. España → Bmé. Mitre | Cuello de botella norte |
| `v_pte_brown` | Puente Almirante Brown (acceso / salida) | Acceso sur |
| `v_pte_heras` | Puente Las Heras (acceso / salida) | Acceso oeste |

**Tipología B — Barreras Ferroviarias (FC Sarmiento)**

| ID Vector | Tramo | Lógica |
|---|---|---|
| `v_fc_casey` | Cruce Lorenzo Casey / Beschtedt | Barrera intermitente |
| `v_fc_mitre` | Cruce Bmé. Mitre / Av. España | Alto tráfico + barrera |

**Tipología C — Corredores Troncales (×5 segmentos)**

| ID Vector | Tramo | Lógica |
|---|---|---|
| `v_humberto_01` | Av. Pellegrini → San Martín | Eje central norte |
| `v_humberto_02` | San Martín → Carlos Pellegrini | Eje central sur |
| `v_basilical` | Av. NSra. de Luján (completo) | Sábados/domingos crítico |
| `v_sanmartin` | San Martín / Constitución | Distribución este-oeste |
| `v_av_españa` | Av. España (tramo urbano) | Arteria principal |

*Los ~20 vectores restantes se calibran tras el diagnóstico visual inicial con osmnx.*

### 4.3. Fuentes complementarias (bajo costo / sin costo)

| Fuente | Qué aporta | Prioridad |
|---|---|---|
| **OpenStreetMap** vía `osmnx` | Red vial base, jerarquía, topología | CRÍTICA |
| **INDEC Censo 2022** | Densidad por radio censal → proxy de demanda | ALTA |
| **ANSV datos abiertos** | Siniestros geocodificados (con subregistro conocido) | ALTA |
| **Scraping noticias locales** (El Civismo, etc.) | Accidentes moto-auto no registrados en ANSV | MEDIA |
| **Floating car data artesanal** | 5-10 recorridos repetidos en picos → calibración real de Google | MUY ALTA — diferencial |
| **Dirección Nacional de Vialidad** | Conteos en rutas de acceso → condición de borde | MEDIA |
| **Waze for Cities** | Incidentes en tiempo real (gestión requiere acuerdo municipal) | FUTURA FASE |

> **Nota metodológica (Experto 2):** Google no es "la verdad". Los recorridos propios con celular son el ancla de calibración. Esto también es el argumento de credibilidad frente al municipio: "validamos los datos de API con mediciones físicas".

---

## 5. Pipeline de Recolección en n8n

### 5.1. Cronograma de muestreo asimétrico

```
Días Hábiles (Lunes–Viernes):
  07:00 | 07:30 | 08:00 | 08:30   ← Pico mañana
  12:00 | 13:00 | 14:00           ← Valle mediodía
  17:00 | 17:30 | 18:00 | 18:30 | 19:00  ← Pico tarde
  → 12 disparos × 30 vectores = 360 peticiones/día

Fines de Semana (Sábado y Domingo):
  11:00 | 12:00 | 13:00 | 14:00 | 15:00 | 16:00 | 17:00 | 18:00  ← Eje basilical activo
  → 8 disparos × 30 vectores = 240 peticiones/día

Consumo mensual: (360×22) + (240×8) = 9.840 peticiones → USD 49,20
Crédito disponible: USD 200 → Margen: USD 150,80
```

### 5.2. Estructura del flujo n8n

```
[Cron Trigger] → [Set Variables: timestamp, is_weekend, hora_pico] 
    → [SplitInBatches: 30 vectores OD]
    → [HTTP Request: Google Routes API]
        Headers: X-Goog-FieldMask: routes.duration,routes.staticDuration
    → [Code Node: Calcular friction_index]
        friction = (duration_s - static_duration_s) / static_duration_s
    → [PostgreSQL Insert: tabla friction_raw]
    → [IF error] → [Slack/Gmail notificación]
```

### 5.3. Esquema de base de datos PostgreSQL

```sql
-- Esquema transito en postgres_iuser (ya existente)
CREATE SCHEMA IF NOT EXISTS transito;

CREATE TABLE transito.od_vectors (
    id_vector    VARCHAR(30) PRIMARY KEY,
    nombre       TEXT,
    tipologia    CHAR(1),  -- A, B o C
    lat_origen   NUMERIC(9,6),
    lon_origen   NUMERIC(9,6),
    lat_destino  NUMERIC(9,6),
    lon_destino  NUMERIC(9,6)
);

CREATE TABLE transito.friction_raw (
    id              SERIAL PRIMARY KEY,
    id_vector       VARCHAR(30) REFERENCES transito.od_vectors(id_vector),
    timestamp       TIMESTAMPTZ NOT NULL,
    is_weekend      BOOLEAN,
    hora_pico       VARCHAR(10),  -- 'mañana', 'mediodia', 'tarde', 'otro'
    duration_s      INTEGER,
    static_duration_s INTEGER,
    friction_index  NUMERIC(5,3),  -- (duration - static) / static
    fuente          VARCHAR(20) DEFAULT 'google_routes'
);

CREATE INDEX ON transito.friction_raw (id_vector, timestamp);
CREATE INDEX ON transito.friction_raw (timestamp);
```

---

## 6. Análisis y Diagnóstico (Python · geoenv_platform)

### 6.1. Procesamiento de la red base (osmnx)

```python
import osmnx as ox
import geopandas as gpd
import networkx as nx

# Descarga y simplificación topológica (crítico per Experto 1)
G = ox.graph_from_place("Luján, Buenos Aires, Argentina", network_type="drive")
G_simplified = ox.simplify_graph(G)  # elimina nodos falsos en rotondas

# Asignación de velocidades teóricas por jerarquía OSM
# primary: 60 km/h | secondary: 50 | tertiary/residential: 40
G_with_speeds = ox.add_edge_speeds(G_simplified)
G_with_speeds = ox.add_edge_travel_times(G_with_speeds)

# Exportar a GeoJSON para PostGIS / visualización
edges = ox.graph_to_gdfs(G_simplified, nodes=False)
edges.to_file("red_vial_lujan.geojson", driver="GeoJSON")
```

### 6.2. Métricas de estructura de red (Experto 2 — diferencial)

Estas métricas permiten argumentar con datos frente al municipio:

```python
# Betweenness Centrality — identifica nodos críticos
bc = nx.betweenness_centrality(G_simplified, weight='travel_time', normalized=True)
# → "El 35% del tráfico depende de solo 5 intersecciones"

# Circuidad (Circuity Average)
# ratio entre distancia real recorrida y distancia euclidiana
circuity = ox.stats.circuity_avg(G_simplified)
# → Cuantifica cuánto obliga la red a rodear obstáculos (puentes, FFCC)

# Centralidad de carga en cada arista
ec = nx.edge_betweenness_centrality(G_simplified, weight='travel_time')
# → Identifica calles sobreexplotadas vs subutilizadas
```

### 6.3. Índice de Fricción interpolado sobre la red

```python
# Para vectores no cubiertos por la API → interpolación espacial
# 1. Calcular friction_index promedio por vector OD (desde PostgreSQL)
# 2. Asignar el índice a los segmentos OSM que intersectan el vector
# 3. Para segmentos sin cobertura: kriging o IDW sobre los vecinos
# Resultado: cada arista del grafo tiene un friction_index estimado
```

### 6.4. Modelo de Riesgo Vial

Scoring ponderado por intersección (0–100):

```
Riesgo = 0.35 × Fricción_pico + 0.25 × Siniestros_ANSV_normalizados
       + 0.20 × Siniestros_scraping_normalizados + 0.10 × Centralidad
       + 0.10 × Cambios_recientes_sentido
```

**Clasificación:** Alto (>70) · Medio (40–70) · Bajo (<40)

**Foco moto–auto:** bonus de riesgo en intersecciones con:
- Giros en T sin señal
- Visibilidad reducida (vegetación, edificación en esquina)
- Velocidades de circulación > 50 km/h + peatones

---

## 7. Simulación de Escenarios — "Modelo de Accesibilidad y Eficiencia de Red"

> Adoptando la reformulación del Experto 2: **no es una simulación de tránsito** (no modela colas ni saturación). Es un **modelo de accesibilidad** que cuantifica la dirección y magnitud del cambio.

### 7.1. Metodología: Volume-Delay Functions (BPR)

```python
# Bureau of Public Roads: t = t0 × (1 + α × (V/C)^β)
# t0: tiempo a flujo libre | V: volumen estimado | C: capacidad de la vía
# α = 0.15, β = 4 (valores estándar)

# Volumen estimado: proxy del Modelo de Gravedad O-D
# basado en densidad censal INDEC 2022 por radio censal
```

### 7.2. Los tres escenarios

**Escenario 0 — Línea de base (actual)**
- Red OSM + tiempos reales de Google Routes API
- Sin modificaciones

**Escenario A — Intervenciones de bajo costo (señalización + semáforos)**
- Modificar atributos direccionales en el grafo para simular cambios de sentido propuestos
- Ajustar pesos de aristas en intersecciones con nueva coordinación semafórica
- Recalcular matriz O-D sintética → delta en tiempo total de red

**Escenario B — Intervenciones estructurales (mediano plazo)**
- Idem + restricciones en cruces de FFCC (horario de barreras)
- Carriles exclusivos en corredores troncales
- Nuevos puntos de cruce del Río Luján (si aplica)

**Métrica de comparación:**
```
ΔFricción_red = (Σ travel_time_escenario - Σ travel_time_base) / Σ travel_time_base × 100
```
→ "La intervención A reduce la fricción promedio de la red en un X% durante el pico vespertino."

---

## 8. Frontend Web — `transito.soildecisions.com`

### 8.1. Stack técnico (100% estático)

- **Motor de mapas:** Leaflet.js 1.9+ con plugins `leaflet.heat` y `leaflet-choropleth`
- **Gráficos:** Chart.js 4 (tiempos de viaje por horario, evolución del índice de fricción)
- **Datos:** GeoJSON pre-generados por los scripts Python, embebidos en `/data/`
- **Sin backend ni API calls desde el browser** → deploy en nginx en 1 minuto

### 8.2. Estructura de la interfaz

```
┌─────────────────────────────────────────────┐
│  TRÁNSITO LUJÁN · Diagnóstico Vial PoC      │
├─────────────────┬───────────────────────────┤
│                 │  ESCENARIO: [●Base ○A ○B]  │
│   MAPA          │  CAPA:  [Congestión] [Riesgo] [Red] │
│   INTERACTIVO   │─────────────────────────────│
│                 │  📊 Fricción promedio: 1.42  │
│   Leaflet.js    │  ⚡ Nodos críticos: 5       │
│                 │  🔴 Intersecciones alto riesgo: 12 │
│                 │─────────────────────────────│
│                 │  [Gráfico: fricción por hora]│
├─────────────────┴───────────────────────────┤
│  🔒 Módulo Tiempo Real (requiere Waze CCP)   │
│  🔒 Integración Semafórica (requiere acceso  │
│     al centro de monitoreo municipal)        │
└─────────────────────────────────────────────┘
```

### 8.3. Módulo de Seguridad Vial (integrado)

- Capa de calor de riesgo (Alto/Medio/Bajo) con popup por intersección
- Tipología de conflicto probable (moto–auto, peatón, vehículo pesado)
- Ficha de cada punto crítico: datos que sustentan el riesgo + medida propuesta

### 8.4. Estrategia de "upselling" (Experto 1 — imprescindible)

Los módulos bloqueados en la UI hacen visible exactamente qué falta para la siguiente fase:
- `🔒 Módulo de Datos en Tiempo Real (Requiere API Waze CCP/TomTom)`
- `🔒 Integración de Red Semafórica (Requiere acceso al centro de monitoreo local)`
- `🔒 Simulación de Demanda con Datos SUBE/Telefonía`
- `🔒 Calibración con Conteos Vehiculares Oficiales`

---

## 9. Entregables del Proyecto

| # | Entregable | Formato | Destino |
|---|---|---|---|
| 1 | **Reporte Técnico** | PDF (Quarto → PDF) | Municipio / archivo |
| 2 | **Presentación Ejecutiva** | Google Slides (10 slides) | Tomadores de decisión |
| 3 | **Web interactiva** | HTML estático en Docker/Nginx | `transito.soildecisions.com` |
| 4 | **Pipeline de datos** | Flujo n8n exportado (.json) | `n8n.soildecisions.com` |
| 5 | **Código fuente** | Repositorio GitHub | `github.com/angelini75/transito-lujan` |
| 6 | **Módulo de seguridad vial** | Integrado en la web | Ídem |

---

## 10. Plan de Trabajo — 5 Fases

```
Semana 1    DATOS BASE
  ─ Descargar red OSM, simplificar topología (geoenv_platform)
  ─ Crear esquema transito en PostgreSQL
  ─ Configurar flujo n8n + tabla od_vectors con los 30 pares
  ─ Activar cron → comienza recolección de datos reales
  ─ Recolección artesanal: 5-10 recorridos con celular (calibración)

Semana 2    DIAGNÓSTICO
  ─ Calcular Índice de Fricción por vector
  ─ Betweenness centrality + circuidad de red
  ─ Interpolación de fricción a toda la red
  ─ Diagnóstico preliminar de cuellos de botella

Semana 3    RIESGO + SIMULACIÓN
  ─ Modelo de riesgo vial (scoring por intersección)
  ─ Geocodificación de siniestros ANSV + noticias locales
  ─ Diseño de intervenciones (ficha: datos / costo / impacto / riesgo político)
  ─ Top 5 intervenciones de alto impacto / bajo costo
  ─ Simulación escenarios A y B (shortest path con pesos ajustados + BPR)

Semana 4    FRONTEND
  ─ Deploy `transito_server` en docker-compose
  ─ Mapa Leaflet con capas de congestión / riesgo / red
  ─ Comparador de escenarios (toggle Base / A / B)
  ─ Panel de KPIs + gráficos Chart.js
  ─ Módulos bloqueados como "upsell"

Semana 5    ENTREGABLES
  ─ Reporte técnico (Quarto)
  ─ Presentación ejecutiva
  ─ Revisión final del web
  ─ Publicación en GitHub
```

---

## 11. Presupuesto Estimado

| Ítem | Costo mensual | Notas |
|---|---|---|
| Google Routes API | USD ~49 | Dentro del crédito gratuito (USD 200/mes) |
| VM (GCP small) | Ya activa | Compartida con otros proyectos |
| Dominio `transito.soildecisions.com` | USD 0 | Subdominio de dominio existente |
| SSL | USD 0 | Let's Encrypt via Traefik |
| Hosting web | USD 0 | Nginx + Docker en VM existente |
| **Total incremental** | **USD 0** | Todo dentro de la infraestructura actual |

---

## 12. Supuestos y Limitaciones Explícitas

1. **Google Routes API no es "la verdad"** (Experto 2): es un snapshot con sesgo de modelo opaco. Se valida con recorridos propios. Se documenta explícitamente en el reporte.

2. **La "simulación" no es microsimulación:** no modela colas, ni interacción entre vehículos, ni saturación. Es un modelo de accesibilidad y eficiencia de red. Se presentará así en todos los entregables.

3. **ANSV tiene subregistro severo a nivel municipal:** se complementa con scraping de noticias locales, pero el mapa de riesgo tiene incertidumbre alta. Se representa visualmente (bandas de confianza).

4. **Sin datos de demanda oficial (SUBE, telefonía):** se usa modelo de gravedad O-D basado en Censo 2022. Es una estimación con supuestos explícitos.

5. **La PoC no requiere acceso municipal.** Puede ejecutarse íntegramente con datos abiertos. El acceso municipal (sentidos oficiales, semáforos, cambios recientes) enriquece la calidad pero no es bloqueante.

---

## 13. Narrativa Estratégica para el Municipio

Siguiendo la recomendación del Experto 2, el proyecto se presenta con dos escenarios de datos:

| | PoC (este proyecto) | Segunda fase |
|---|---|---|
| **Datos de demanda** | Modelo de gravedad proxy | SUBE + telefonía |
| **Datos de tránsito** | Google API + calibración manual | Waze CCP + conteos oficiales |
| **Precisión simulación** | Indicativa (±30%) | Cuantitativa (±10%) |
| **Incertidumbre** | Alta → visible en el mapa | Baja |
| **Decisiones posibles** | Diagnóstico + priorización | Optimización fina + monitoreo |

> **Mensaje clave:** "Con los datos disponibles hoy podemos identificar dónde está el problema y qué hacer. Con una inversión de [X], podemos decirles con precisión cuánto va a mejorar."

La diferencia entre ambos escenarios **es el argumento de venta**. Mostrarla en la web es más convincente que explicarla en un informe.

---

*Documento generado a partir de: `role_objetivo_tarea_entregables.md`, `Opiniones experto 1.md`, `Opiniones experto 2.md`, `estrategia_data_collection.md` + exploración directa de la infraestructura VM (`small-20250401-n8n`, GCP southamerica-east1-c, 2026-04-19).*
