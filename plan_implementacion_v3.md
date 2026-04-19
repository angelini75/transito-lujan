# Plan de Implementación v3 — Diagnóstico de Tránsito Urbano · Luján, Buenos Aires
**Versión:** 3.0 · Revisión estratégica  
**Fecha:** 2026-04-19  
**Supersede:** plan_implementacion_v2.md

---

## Cambios principales respecto a v2

| Dimensión | v2 | v3 |
|---|---|---|
| **Fuente de datos principal** | Google Routes API | Conocimiento local + GPS tracks + Google API |
| **Orquestación** | n8n (propuesto) | n8n (confirmado, ya corriendo con credenciales completas) |
| **Claude Code Routines** | Alternativa | Rol específico diferenciado (análisis, no recolección) |
| **Input del residente** | No incluido | Protocolo formal: Telegram bot + GPX + bitácora |
| **Noticias locales** | Genérico | Workflow n8n para lujanenlinea.com.ar + civismo.com.ar |

---

## 1. Arquitectura General Revisada

```
╔══════════════════════════════════════════════════════════════╗
║           CAPA 1 — CONOCIMIENTO LOCAL (Fuente primaria)       ║
║                                                               ║
║  Residente-conductor (vos)                                    ║
║  ├── Telegram bot → n8n → PostgreSQL                          ║
║  │   (observaciones de campo desde el celular)                ║
║  ├── Archivos GPX → script Python → tabla gps_tracks          ║
║  │   (recorridos reales calibradores)                         ║
║  └── Bitácora estructurada → YAML → importada a DB            ║
║       (lomas de burro, conflictos, anomalías)                 ║
╠══════════════════════════════════════════════════════════════╣
║           CAPA 2 — DATOS AUTOMATIZADOS (n8n en VM)            ║
║                                                               ║
║  Workflow A: Google Routes API                                 ║
║  └── Cron cada 30 min → 30 vectores OD → friction_raw         ║
║                                                               ║
║  Workflow B: Scraping noticias locales                        ║
║  └── Cron diario 6:00 → lujanenlinea + civismo → incidentes   ║
╠══════════════════════════════════════════════════════════════╣
║           CAPA 3 — ANÁLISIS (Claude Code Routine semanal)     ║
║                                                               ║
║  Routine: "transito-lujan-analisis" (claude.ai/code/routines) ║
║  └── Trigger: semanal (lunes 8am)                             ║
║  └── Lee DB → actualiza GeoJSON → abre PR en repo             ║
╠══════════════════════════════════════════════════════════════╣
║           CAPA 4 — FRONTEND (transito.soildecisions.com)      ║
║  nginx estático · Traefik SSL · GeoJSON pre-generados         ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 2. Capa 1 — Protocolo de Conocimiento Local

### 2.1. Por qué esto es la fuente más valiosa

Un habitante-conductor conoce cosas que ninguna API puede inferir:

- La barrera del FFCC en Casey baja 10 minutos antes de que pase el tren y el chofer del 57 dobla igual
- La lomada frente a la escuela No. 3 no está en OSM
- Los sábados de peregrinación el Paseo del Río es intransitable desde las 9am
- El semáforo de San Martín y Humberto I tiene fase verde de 8 segundos para la transversal → genera cola en ambas direcciones
- Horarios del Hospital Zonal + UNLu generan picos secundarios invisibles para Google

Esta información tiene **altísima precisión** y **costo cero**. Estructurarla correctamente es la diferencia entre una PoC genérica y un diagnóstico genuino de Luján.

---

### 2.2. Canal A — Observaciones instantáneas vía Telegram

n8n ya tiene configurados **tres bots de Telegram**. Usaremos uno como canal de campo.

**Flujo:**
```
Vos (en el auto, en tiempo real) → Telegram bot → n8n Webhook → PostgreSQL
```

**Comandos del bot:**
```
/obs [texto libre] → Observación general (se geocodifica por dirección si la mencionás)
/loma [dirección] → Registrar loma de burro con timestamp
/conflicto [intersección] [descripción] → Punto de conflicto
/anomalia [descripción] → Evento que altera el patrón (peregrinación, feria, etc.)
/gps [inicio|fin] → Marcar inicio/fin de recorrido GPS manual
```

**Esquema de tabla:**
```sql
CREATE TABLE transito.observaciones_locales (
    id              SERIAL PRIMARY KEY,
    timestamp       TIMESTAMPTZ DEFAULT NOW(),
    tipo            VARCHAR(20),  -- 'obs', 'loma', 'conflicto', 'anomalia'
    texto           TEXT,
    interseccion    VARCHAR(200),
    lat             NUMERIC(9,6),
    lon             NUMERIC(9,6),
    dia_semana      VARCHAR(10),
    hora_pico       VARCHAR(10),  -- 'mañana', 'tarde', 'noche', 'otro'
    fuente          VARCHAR(20) DEFAULT 'telegram_local'
);
```

---

### 2.3. Canal B — GPS Tracks (calibración de Google API)

**Objetivo:** Validar que los tiempos de Google Routes API coinciden con la realidad local. Si hay desvío sistemático en un vector, ajustar el modelo.

**Protocolo de recolección:**

| Paso | Acción | Herramienta |
|---|---|---|
| 1 | Instalar OsmAnd (Android/iOS, gratuito) o usar Google Maps con "Compartir viaje" | Celular |
| 2 | Activar grabación GPX antes de salir | OsmAnd → "Grabar viaje" |
| 3 | Recorrer el vector OD exacto (mismos extremos que definimos) | Vehículo propio |
| 4 | Exportar el archivo `.gpx` al finalizar | OsmAnd → Exportar |
| 5 | Copiar el `.gpx` a `/opt/mi-stack/transito-app/gps-tracks/` | scp o Google Drive → VM |
| 6 | Script Python extrae: duración, velocidad media, stops detectados | `geoenv_platform` |

**Cobertura mínima recomendada:**
- 3 recorridos por vector de Tipología A (puentes) en horario pico mañana
- 2 recorridos por vector de Tipología B (FFCC) en horario pico tarde
- 1 recorrido por vector de Tipología C en horario normal

**Total estimado:** 15–20 recorridos. Se pueden hacer en el transcurso normal de la semana.

**Tabla de almacenamiento:**
```sql
CREATE TABLE transito.gps_tracks (
    id              SERIAL PRIMARY KEY,
    id_vector       VARCHAR(30),
    timestamp_ini   TIMESTAMPTZ,
    timestamp_fin   TIMESTAMPTZ,
    duracion_s      INTEGER,
    distancia_m     NUMERIC(8,1),
    vel_media_kmh   NUMERIC(5,1),
    vel_max_kmh     NUMERIC(5,1),
    n_stops         INTEGER,
    archivo_gpx     VARCHAR(200),
    notas           TEXT,
    fuente          VARCHAR(20) DEFAULT 'gpx_local'
);
```

---

### 2.4. Canal C — Bitácora Estructurada (YAML → DB)

Para el conocimiento que requiere más detalle que un mensaje de Telegram: lomas de burro, diseño de intersecciones conflictivas, calles con anomalías crónicas.

**Formato del archivo** (`conocimiento_local.yaml`):

```yaml
# ──────────────────────────────────────────
# CONOCIMIENTO LOCAL — Luján · Conductor residente
# Última actualización: 2026-04-19
# ──────────────────────────────────────────

lomas_de_burro:
  - id: lb_001
    calle: "Humberto I"
    altura: "altura 500"
    lat: -34.5710
    lon: -59.1130
    tipo: "reductor de velocidad"
    contexto: "frente a escuela"
    efecto_trafico: "provoca frenada brusca en pico mañana → cola de 3-4 vehículos"

conflictos_conocidos:
  - id: cf_001
    interseccion: "San Martín y Humberto I"
    tipo: "semaforo_mal_calibrado"
    descripcion: >
      Fase verde de ~8 segundos para la transversal. En pico tarde
      genera cola de 6-8 vehículos sobre Humberto I. Conductores
      de moto esquivan por la vereda derecha → riesgo peatonal.
    horario_critico: "17:00-19:00"
    dias: ["lunes", "martes", "miercoles", "jueves", "viernes"]
    nivel_riesgo_estimado: "alto"

  - id: cf_002
    interseccion: "Lorenzo Casey y FC Sarmiento"
    tipo: "barrera_ferroviaria"
    descripcion: >
      La barrera baja 10+ min antes del paso del tren. En pico
      mañana (7:30-8:30) coincide con entrada UNLu. Cola llega
      hasta Bmé. Mitre. El 30% de motos dobla sobre la vereda
      para buscar el puente Muñiz como alternativa.
    horario_critico: "07:30-08:30"
    dias: ["lunes", "martes", "miercoles", "jueves", "viernes"]
    nivel_riesgo_estimado: "alto"

anomalias_calendario:
  - evento: "Peregrinación Virgen de Luján"
    frecuencia: "primer sábado de octubre + fechas variables"
    impacto: >
      Av. NSra. de Luján CERRADA desde las 8am. Todo el tráfico
      se redistribuye por Humberto I y San Martín. Congestión
      severa en puentes Muñiz y Brown desde las 10am.
    vectores_afectados: ["v_basilical", "v_pte_muniz", "v_pte_brown"]
    marcar_como_outlier: true

  - evento: "Días de feria americana (Hipódromo)"
    frecuencia: "domingos eventuales"
    impacto: "Alto tráfico en acceso hipódromo + estacionamiento en calles aledañas"
    vectores_afectados: ["v_av_españa"]
    marcar_como_outlier: true

  - evento: "Inicio/fin de semestre UNLu"
    frecuencia: "marzo, julio, diciembre"
    impacto: "Pico adicional de 07:45-08:15 y 18:45-19:15 en accesos norte"
    marcar_como_outlier: false  # es un patrón, no outlier

puntos_ciegos_de_visibilidad:
  - interseccion: "completar con tu conocimiento"
    descripcion: "árbol / edificio / camión estacionado que tapa la visual"
    riesgo: "alto"
```

**Script de importación** (Python, corre en `geoenv_platform`):
```python
import yaml, psycopg2
# Leer conocimiento_local.yaml → insertar en observaciones_locales + lomas_burro
```

---

### 2.5. Validación cruzada: GPS vs Google API

```python
# Para cada vector donde tenemos GPS tracks:
# 1. Calcular friction_index real desde GPX:
#    friction_gpx = (duracion_gpx - static_duration_google) / static_duration_google
# 2. Comparar con friction_index de Google API en mismo horario:
#    delta = friction_google - friction_gpx
# 3. Si |delta| > 0.3 → bandera "datos_inconsistentes" → revisar manual
# 4. Si delta consistente (bias sistemático) → factor de corrección por vector
```

---

## 3. Capa 2 — Workflows n8n

### 3.1. Decisión: n8n para recolección automatizada

n8n ya está corriendo en `n8n.soildecisions.com` con las siguientes credenciales disponibles:

| Credencial | ID | Uso en el proyecto |
|---|---|---|
| `Postgres account` | 7EQhAGuGJ3cMAkvc | Insertar registros de fricción + noticias |
| `Google Service Account` | OEALkY7AghoY4iAR | Google Routes API (via HTTP Request) |
| `Google Gemini API` | OZFXPVE1qCbnVHsN | Clasificar noticias de accidentes con IA |
| `Telegram bot` | VkzQTJpvmKaXzt8C | Canal de observaciones locales |

✅ **No requiere configurar credenciales nuevas para empezar.**

---

### 3.2. Workflow A — Recolección Google Routes API

**Nombre:** `Transito Luján · Fricción Vial`  
**Trigger:** Cron asimétrico (ver Cronograma en v2, sección 5.1)  
**Credenciales:** `Google Service Account` (ya configurada) + `Postgres account`

```
[Schedule Trigger]
  ↓ (12x/día L-V, 8x/día S-D)
[Set: timestamp, is_weekend, hora_pico]
  ↓
[Loop over items: array de 30 vectores OD]
  ↓
[HTTP Request: POST routes.googleapis.com/directions/v2:computeRoutes]
  Headers: X-Goog-FieldMask: routes.duration,routes.staticDuration
  Auth: Service Account API Key
  ↓
[Code node: calcular friction_index]
  friction = (duration_s - static_duration_s) / static_duration_s
  ↓
[Postgres: INSERT INTO transito.friction_raw]
  ↓ (on error)
[Telegram: notificar falla → bot VkzQTJpvmKaXzt8C]
```

---

### 3.3. Workflow B — Scraping Noticias de Accidentes

**Nombre:** `Transito Luján · Noticias Accidentes`  
**Trigger:** Cron diario, 6:00 AM (America/Argentina/Buenos_Aires)  
**Credenciales:** `Postgres account` + `Google Gemini API`

**Fuentes:**
- `https://www.lujanenlinea.com.ar/` — portal de noticias local
- `https://www.civismo.com.ar/` — Semanario El Civismo

```
[Schedule Trigger: 6:00 AM diario]
  ↓
[HTTP Request: GET lujanenlinea.com.ar]          [HTTP Request: GET civismo.com.ar]
  ↓                                                    ↓
[Code node: extraer artículos + títulos + URLs]  [Code node: ídem]
  ↓                                                    ↓
[Merge: unir resultados de ambas fuentes]
  ↓
[Code node: filtrar por keywords]
  → "accidente" OR "choque" OR "colisión" OR "moto" OR "atropell*" OR "siniestro"
  ↓
[HTTP Request: Gemini API · Clasificar incidente]
  Prompt: "Dado este titular y fragmento de noticia de Luján, Buenos Aires:
  - ¿Es un accidente de tránsito? (sí/no)
  - Tipo: moto-auto / auto-auto / atropello / otro
  - Intersección mencionada (extraer texto exacto)
  - Víctimas reportadas: (número o 'no mencionadas')
  - Gravedad: leve/grave/fatal/no indicada
  Responder en JSON."
  ↓
[Code node: geocodificar intersección → lat/lon usando OSM Nominatim]
  Endpoint: https://nominatim.openstreetmap.org/search?q={interseccion}+Luján+Buenos+Aires
  ↓
[IF: es_accidente = true]
  → [Postgres: INSERT INTO transito.incidentes_noticias]
  ↓ else → [discard]
```

**Esquema tabla:**
```sql
CREATE TABLE transito.incidentes_noticias (
    id              SERIAL PRIMARY KEY,
    timestamp_pub   DATE,
    timestamp_carga TIMESTAMPTZ DEFAULT NOW(),
    fuente          VARCHAR(30),  -- 'lujanenlinea' | 'civismo'
    titulo          TEXT,
    url             TEXT,
    tipo_accidente  VARCHAR(30),
    interseccion_raw TEXT,
    lat             NUMERIC(9,6),
    lon             NUMERIC(9,6),
    victimas        INTEGER,
    gravedad        VARCHAR(10),
    gemini_raw      JSONB,
    geocod_confianza VARCHAR(10)  -- 'alta'/'media'/'baja'
);
```

---

### 3.4. Workflow C — Receptor Telegram (Observaciones Locales)

**Nombre:** `Transito Luján · Campo`  
**Trigger:** Telegram Trigger (bot VkzQTJpvmKaXzt8C)

```
[Telegram Trigger: mensaje recibido]
  ↓
[Code node: parsear comando]
  /obs → tipo='observacion'
  /loma → tipo='loma_burro'
  /conflicto → tipo='conflicto'
  /anomalia → tipo='anomalia'
  ↓
[IF: texto contiene dirección o intersección]
  → [HTTP Request: Nominatim geocodificar]
  ↓
[Postgres: INSERT INTO transito.observaciones_locales]
  ↓
[Telegram: responder "✅ Registrado [{tipo}] — {timestamp}"]
```

---

## 4. Capa 3 — Claude Code Routine (análisis semanal)

### 4.1. Cuándo usar Routine vs n8n

| Tarea | Herramienta | Por qué |
|---|---|---|
| Polling Google API (cada 30 min) | **n8n** | Intervalo < 1h; ya tiene credenciales; más estable para tareas repetitivas simples |
| Scraping noticias (diario) | **n8n** | HTTP + parsing + clasificación Gemini; todo ya disponible |
| Observaciones locales vía Telegram | **n8n** | Webhook receiver; respuesta instantánea |
| Análisis profundo + generación GeoJSON | **Claude Code Routine** | Requiere razonamiento, lectura de DB, decisiones contextuales |
| Actualización del reporte semanal | **Claude Code Routine** | Produce texto + archivos + PR en GitHub |

### 4.2. Routine: `transito-lujan-analisis`

**Trigger:** Schedule · Lunes 8:00 AM  
**Repositorio:** `github.com/angelini75/transito-lujan`  
**Prompt del routine:**

```
Eres el sistema de análisis de tránsito urbano del proyecto Luján.

Cada semana ejecutás las siguientes tareas:

1. CONECTATE a PostgreSQL (host: postgres_iuser, db: iuser_db, schema: transito)
   - Variables en env: POSTGRES_HOST, POSTGRES_USER, POSTGRES_PASSWORD

2. CALCULÁ las métricas de la semana:
   - Friction index promedio por vector y por franja horaria
   - Top 5 vectores con mayor fricción (pico mañana y pico tarde)
   - Incidentes nuevos en tabla incidentes_noticias
   - Observaciones locales nuevas

3. ACTUALIZÁ los archivos GeoJSON en transito-app/data/:
   - friction_red.geojson: actualizar propiedades con valores de la semana
   - riesgo_intersecciones.geojson: recalcular scoring si hay nuevos incidentes
   - incidentes_semana.geojson: incidentes de los últimos 7 días

4. GENERÁ un archivo weekly_summary.md con:
   - Métricas clave de la semana
   - Cambios respecto a semana anterior
   - Alertas (vectores con friction > 1.5 sostenido)
   - Nuevos incidentes relevantes

5. ABRÍ un Pull Request con los archivos actualizados contra la rama main.
   Título: "Data update - semana del {FECHA}"
```

**Variables de entorno del Routine:**
```
POSTGRES_HOST=postgres_iuser (o la IP interna de la VM)
POSTGRES_USER=iuser_admin
POSTGRES_PASSWORD=[desde secrets]
```

---

## 5. Vectores OD Definitivos — Validación por Residente

**IMPORTANTE:** La lista definitiva de los 30 vectores debe ser validada por vos antes de activar el workflow. Los vectores de la v2 son una propuesta basada en topografía pública. Solo vos sabés:

- Si hay un vector que parece crítico en el mapa pero en realidad fluye bien
- Si hay un tramo no obvio en el mapa que sí es un cuello de botella real
- Los extremos exactos que evitan que Google reutee para calles alternativas

**Plantilla para definir vectores** (a completar por el residente):

```yaml
vectores_od:
  - id: v_pte_muniz
    nombre: "Puente Muñiz - Norte"
    tipologia: A  # Embudo físico
    origen_desc: "[TU DESCRIPCIÓN: 'esquina de X y Z, antes de la rotonda']"
    origen_lat: null  # a completar
    origen_lon: null  # a completar
    destino_desc: "[TU DESCRIPCIÓN]"
    destino_lat: null
    destino_lon: null
    horario_critico: "[cuándo es más intenso según tu experiencia]"
    notas: "[qué pasa específicamente ahí que vale la pena medir]"
    validado_residente: false

  - id: v_pte_brown
    # ... ídem para cada puente / barrera / corredor
```

El próximo paso, antes de activar n8n, es que me dictés (o escribas en un mensaje) la descripción de cada vector como vos lo conocés. Yo convierto eso en coordenadas precisas vía Nominatim.

---

## 6. Plan de Trabajo Revisado

```
ANTES DE EMPEZAR (vos, ~2 horas)
  ─ Validar / corregir la lista de vectores OD
  ─ Completar conocimiento_local.yaml con tu conocimiento de la ciudad
  ─ Decidir qué bot de Telegram usar para las observaciones de campo

Semana 1 — INFRAESTRUCTURA DE DATOS
  ─ Crear schema transito en PostgreSQL (via n8n o script directo)
  ─ Cargar tabla od_vectors con coordenadas validadas por vos
  ─ Importar conocimiento_local.yaml → tablas lomas_burro + conflictos + anomalias
  ─ Activar Workflow A (Google Routes) + Workflow C (Telegram)
  ─ Verificar primera corrida y primeros registros en DB

Semana 2 — RECOLECCIÓN + CALIBRACIÓN
  ─ Activar Workflow B (noticias)
  ─ Empezar recolección de GPS tracks (recorridos del día a día)
  ─ Usar el bot de Telegram para enviar primeras observaciones
  ─ Comparar friction_index Google vs GPS tracks → ajustar si hay sesgo

Semana 3 — ANÁLISIS
  ─ Calcular betweenness centrality y circuidad de red (geoenv_platform)
  ─ Modelo de riesgo: combinar incidentes + GPS + observaciones locales + API
  ─ Diseñar intervenciones con fichas (datos + costo + impacto + riesgo político)
  ─ Top 5 intervenciones de alto impacto / bajo costo / mínimo riesgo político

Semana 4 — FRONTEND
  ─ Agregar transito_server al docker-compose en VM
  ─ Construir mapa Leaflet: congestión + riesgo + intervenciones
  ─ Comparador de escenarios (toggle Base / A / B)
  ─ Módulos bloqueados "upsell" como se acordó en v2
  ─ Configurar Claude Code Routine semanal

Semana 5 — ENTREGABLES
  ─ Reporte técnico (Quarto)
  ─ Presentación ejecutiva (10 slides)
  ─ Publicación final en GitHub + web
```

---

## 7. Credenciales y Configuración Pendiente

| Item | Estado | Acción requerida |
|---|---|---|
| n8n corriendo | ✅ Listo | — |
| Postgres account en n8n | ✅ Configurado | Verificar que apunta a `postgres_iuser` + crear schema `transito` |
| Google Service Account | ✅ Configurado | Habilitar "Routes API" en Google Cloud Console para el proyecto `ee-angelini75` |
| Telegram bot para campo | ✅ Disponible (3 bots) | Elegir cuál usar; configurar trigger en n8n |
| Gemini API | ✅ Configurado | Verificar que el modelo sea `gemini-1.5-flash` (eficiente para clasificación) |
| API Key Google Routes | ⚠️ Por verificar | Confirmar que la Service Account tiene permisos de Routes API |
| Schema transito en DB | ❌ Pendiente | Ejecutar DDL de creación de tablas |
| od_vectors cargados | ❌ Pendiente | **Requiere validación del residente primero** |

---

## 8. Supuestos Revisados

1. **El conocimiento local es el ancla del proyecto.** Todos los datos automatizados se interpretan en el contexto de lo que vos describís. Un friction_index de 1.8 en un vector es dramático en Humberto I pero podría ser normal en un vector de Tipología B con barrera de tren.

2. **Los GPS tracks son la validación de campo.** No necesitamos instrumentación especial: el celular de una persona que ya hace esos recorridos todos los días es suficiente para calibrar el modelo.

3. **Las noticias son incompletas pero sistemáticas.** El scraping captura lo que se publica. El sesgo de publicación (solo accidentes graves o llamativos) es documentado y comunicado.

4. **n8n es más estable que Claude Code Routines para tareas de alta frecuencia.** Claude Code Routines tiene mínimo 1 hora de intervalo y corre en infraestructura de Anthropic — excelente para análisis, pero n8n en tu VM es más predecible para el polling continuo de datos.

5. **El frontend no es urgente para validar el enfoque.** Los primeros 10 días lo más valioso es recolectar datos reales y contrastarlos con tu conocimiento. La web viene después.

---

*Documento generado a partir de: `plan_implementacion_v2.md`, exploración de infraestructura n8n (API), documentación Claude Code Routines (`Claude_code_routines.md`), y directrices del residente-conductor local.*
