Aquí tienes las directrices técnicas detalladas para que el equipo de análisis ejecute la Prueba de Concepto (PoC). El enfoque es estrictamente operativo, priorizando herramientas open-source, automatización y generación de valor demostrable con costo cero en adquisición de datos.

### Memorándum Técnico: Directrices para PoC de Análisis de Tránsito

**Objetivo:** Desarrollar una herramienta de diagnóstico y simulación de escenarios viales para el casco urbano y accesos principales, operando exclusivamente con fuentes de datos gratuitas o capas sin costo (*free tiers*).

------

### 1. Ingestión de Datos y Orquestación (El "Hack" Operativo)

La recolección manual o mediante scripts aislados es frágil. Debemos establecer un pipeline automatizado desde el día uno.

- **Arquitectura de recolección:** Desplegar un contenedor Docker con una instancia de **n8n** para orquestar las peticiones a la API. Esto garantiza la persistencia del muestreo sin depender de ejecuciones locales.
- **Selección de API:** Utilizar la capa gratuita de *Google Distance Matrix API* o *HERE Routing API*. Ambas permiten un volumen de peticiones suficiente para una PoC si se restringe inteligentemente la muestra.
- **Diseño de la muestra O-D:** Definir un máximo de 30 pares Origen-Destino. Estos vectores deben cruzar obligatoriamente los embudos críticos de la topografía local (ej. puentes sobre el Río Luján, cruces a nivel del ferrocarril Sarmiento, y los principales ejes de acceso como Humberto I y Av. Nuestra Señora de Luján).
- **Cronograma de disparos (Triggers):** Programar el flujo en n8n para consultar los tiempos de viaje 6 veces al día (incluyendo picos de 07:30, 12:30 y 17:30).
- **Variable crítica local:** Es imperativo que el *cron job* funcione los sábados y domingos. El eje histórico-basilical altera completamente la dinámica del flujo vehicular en comparación con los días hábiles. El modelo fracasará si no captura esta estacionalidad de fin de semana.
- **Almacenamiento:** El propio flujo de n8n debe parsear el JSON de respuesta y anexarlo a una base de datos liviana (PostgreSQL con extensión PostGIS) o exportarlo iterativamente a archivos CSV estructurados.

### 2. Procesamiento Espacial y Topología de Red

El núcleo analítico requerirá un rigor absoluto en el tratamiento de la geometría vial.

- **Extracción de red base:** Utilizar flujos de trabajo en Python combinando `osmnx` y `geopandas` (o sus equivalentes espaciales en R, como `sf`) para descargar el polígono central de la ciudad desde OpenStreetMap.
- **Purga topológica (Crítico):** Ejecutar `osmnx.simplify_graph` inmediatamente después de la descarga. Las rotondas o cruces complejos no simplificados generarán falsos cuellos de botella en los algoritmos de enrutamiento.
- **Imputación de velocidades base:** Asignar un límite de velocidad teórico a cada vector del grafo según su jerarquía en OSM (ej. *primary* = 60 km/h, *residential* = 40 km/h).
- **Cálculo del Índice de Fricción:** Cruzar los tiempos reales obtenidos por la API con los tiempos teóricos a flujo libre. Para las calles no muestreadas por la API, aplicar algoritmos de interpolación espacial para estimar su nivel de congestión basándose en los vectores adyacentes.

### 3. Proxy de Demanda y Modelo de Riesgo Vial

A falta de una matriz Origen-Destino oficial y estadísticas de siniestralidad completas, generaremos aproximaciones sintéticas.

- **Modelo de Gravedad O-D:** Utilizar datos abiertos del INDEC (Censo 2022). Asignar pesos de origen basados en la densidad habitacional por radio censal. Los destinos deben mapearse manualmente: polos educativos (UNLu), el Hospital Zonal, áreas industriales y el microcentro.
- **Siniestralidad Alternativa:** La base de la ANSV presenta subregistros severos a nivel municipal. El equipo debe desarrollar un script de *web scraping* (utilizando `BeautifulSoup` o herramientas similares) enfocado en portales de noticias locales (ej. El Civismo). Extraer reportes filtrando por "accidente" + "moto" y geocodificar las intersecciones mencionadas para enriquecer el mapa de calor de riesgo.

### 4. Simulación de Escenarios

El alcance de la simulación debe ser delimitado para no requerir software de microsimulación pesado.

- **Enfoque Macroscópico:** Utilizar algoritmos de caminos mínimos (*shortest path*) sobre el grafo validado de `networkx`.
- **Metodología de alteración:** Para evaluar una intervención (ej. cambiar el sentido de una calle o restringir un giro), el equipo debe modificar el atributo direccional del nodo/vector en el grafo de Python y recalcular los tiempos de viaje de la matriz O-D sintética.
- **Comparativa Delta:** El sistema debe arrojar un porcentaje de variación. El objetivo no es predecir que "un auto tardará 4.2 minutos", sino establecer que "la intervención X reduce la fricción de la red en un 12% en el horario pico vespertino".

### 5. Desarrollo del Front-end y UX para Decisores

El producto final es la herramienta de persuasión para el municipio.

- **Arquitectura \*Serverless\*:** La aplicación web no debe requerir backend. El equipo procesará los tres escenarios (Actual, Intervención A, Intervención B) y exportará todo a formato GeoJSON.
- **Motor de Visualización:** Integrar estos archivos estáticos utilizando `Leaflet.js` o Mapbox GL JS, alojando el *dashboard* resultante en GitHub Pages.
- **Diseño de la Interfaz:** Incluir un *toggle* o interruptor que permita apagar y encender capas de riesgo y congestión.
- **Estrategia de Upselling (El "Cebo"):** Incluir en el menú lateral opciones sombreadas o bloqueadas con textos como *"Módulo de Datos en Tiempo Real (Requiere API Waze CCP/TomTom)"* o *"Integración de Red Semafórica (Requiere acceso a centro de monitoreo local)"*. Esto evidencia gráficamente las limitaciones de la PoC y marca el camino exacto para la siguiente fase de inversión municipal.