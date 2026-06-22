# Mapa Epidemiológico — Mosca de la Fruta

Aplicación **Streamlit** (`mapa_moscav4.py`) que muestra un mapa interactivo de capturas de mosca de la fruta sobre los fundos agrícolas **Arena Azul (AQ1)** y **AQ2** (Quri Allpa / Kawsay Allpa / Ayllu Allpa / Vivadis / Santa Teresa) en Paijan.

---

## Índice

1. [Arquitectura general](#arquitectura-general)
2. [Flujo de datos](#flujo-de-datos)
3. [Funciones clave](#funciones-clave)
4. [Modos de visualización](#modos-de-visualización)
5. [Filtros disponibles](#filtros-disponibles)
6. [Panel de KPIs](#panel-de-kpis)
7. [Exportar a GitHub Pages](#exportar-a-github-pages)
8. [Captura PNG](#captura-png)
9. [Instalación y dependencias](#instalación-y-dependencias)
10. [Configuración de secretos](#configuración-de-secretos)
11. [Estructura de archivos requerida](#estructura-de-archivos-requerida)
12. [Casuísticas y comportamiento esperado](#casuísticas-y-comportamiento-esperado)

---

## Arquitectura general

```
mapa_moscav4.py  (Streamlit backend Python)
│
├── Fuente de datos: Microsoft Fabric / SQL Server (tabla MOSQUITA)
│   └── Autenticación: MSAL Device Flow (Azure AD)
│
├── Geometría: archivo KMZ con polígonos de lotes
│   └── Descarga automática desde GitHub (fallback: archivo local)
│
├── Motor de mapa: mapa_streamlit_js.html  (Leaflet.js)
│   └── Los datos se inyectan en window.streamlitData (JSON)
│
└── Publicación: GitHub API → GitHub Pages (HTML + PNG históricos)
```

---

## Flujo de datos

```
1. Autenticación MSAL
        │
        ▼
2. Query SQL → tabla MOSQUITA (año 2026 filtrado en Python)
        │
        ▼
3. Carga KMZ desde GitHub (o local)
   └── Parseado con lxml → lista de polígonos con:
       fundo_aq | mod_n | tur_n | lote_n | coords[]
        │
        ▼
4. Filtros en sidebar (Semana, Fundo, Módulo, Lote, Turno, Trampa, Fechas)
        │
        ▼
5. Agrupación por (fundo, modulo_n, turno_n, lote_n, trampa) → sum capturas
        │
        ▼
6. calcular_lotes_con_centroide()
   ├── Busca clave AQ|MOD|TUR|LOTE en índice KMZ → centroide del polígono
   ├── Si es trampa PERIMETRAL → ubica el punto en el borde del polígono (Shapely)
   └── Fallback: coordenadas GPS del Excel si existen y son válidas
        │
        ▼
7. Modo visualización:
   ├── Normal      → marcadores de color semaforizado
   ├── Espectral   → contornos gaussianos + marcadores
   └── Curvas de Nivel → contornos gaussianos configurables + líneas + etiquetas
        │
        ▼
8. JSON → window.streamlitData → mapa_streamlit_js.html (Leaflet)
        │
        ▼
9. Publicar (botón) → GitHub Pages HTML + PNG opcional
```

---

## Funciones clave

### `norm_mod(val)` / `norm_tur(val)` / `norm_lote(val)`
Normalizan los valores de módulo, turno y lote desde cualquier formato textual al número entero canónico. Equivalentes a las funciones del mismo nombre en el lado JavaScript.

| Entrada | `norm_mod` | `norm_tur` | `norm_lote` |
|---------|-----------|-----------|------------|
| `"MOD 01"` | `1` | — | — |
| `"M03-T2"` | — | `2` | — |
| `"115B"` | — | — | `"115"` |
| `"1.0"` | — | — | `"1"` |

### `fundo_to_aq(fundo)`
Mapea el nombre textual del fundo al código AQ:

| Nombre fundo | Código AQ |
|---|---|
| Arena Azul | AQ1 |
| Quri Allpa / Vivadis / Kawsay Allpa / Santa Teresa / Ayllu Allpa / Ampliacion | AQ2 |

### `get_semaforo_category(val)`
Convierte capturas a categoría de semaforización:

| Capturas | Categoría | Color |
|---|---|---|
| 0 | 0 | Blanco |
| 1 | 1 | Verde |
| 2 | 2 | Amarillo |
| 3 | 3 | Naranja |
| > 3 | 4 | Rojo |

### `load_kmz_local(kmz_path)`
- Abre el KMZ (ZIP que contiene un KML).
- Busca **carpetas** con nombres `AQ1 - MODULO X` o `AQ2 - MODULO X` (ignora carpeta `Pozos_Prize`).
- Extrae por cada Placemark: nombre, descripción HTML, coordenadas, turno, lote.
- Retorna lista de dicts con: `name`, `coords`, `mod_n`, `tur_n`, `fundo_aq`, `lote`, `lote_name`.
- Cachado con `@st.cache_data`.

### `load_kmz_puntos(kmz_bytes_dict)`
Carga KMZ de **puntos GPS de trampas** (4 ficheros: AQ1, AA, VV, ST) y construye un índice `{fundo_aq|lote → lat/lon}`. Soporta nomenclaturas como `PT_ST_240`, `PT_AA_115B`, `PT_AQ1_JT_T10_Lt86`.

### `calcular_lotes_con_centroide(valid, kmz_polygons)`
Cruza cada fila del DataFrame con el índice KMZ por clave `AQ|MOD|TUR|LOTE`:
- **Match KMZ**: usa el centroide del polígono. Para trampas **perimetrales** proyecta el centroide al borde exterior del polígono (Shapely `nearest_points`).
- **Sin match**: usa coordenadas GPS del Excel si son válidas (no `NaN`, no `-9999`).
- Registra lotes sin match para mostrarlos en la tabla de diagnóstico.

### `generar_contornos_gauss(lotes, polygons_kmz, ...)`
Genera contornos de interpolación gaussiana sobre una grilla 200×200:
1. Calcula el bbox a partir de los polígonos KMZ.
2. Calcula sigma adaptativo basado en la distancia al vecino más cercano.
3. Promedios ponderados locales con kernel gaussiano por cada lote.
4. Suavizado gaussiano ligero (50 m equivalente).
5. Recorte con máscara Shapely (solo dentro de polígonos).
6. Reescala el pico de la grilla al máximo real de capturas.
7. Colormap semaforizado: verde tenue → verde → amarillo → naranja → rojo.
8. Serializa fills y lines de contour como listas de coordenadas `[[lat, lon], ...]`.

---

## Modos de visualización

| Modo | Descripción |
|---|---|
| **Normal** | Marcadores de color semaforizado sobre cada lote. |
| **Espectral** | Contornos gaussianos coloreados + marcadores encima. |
| **Curvas de Nivel** | Contornos + líneas configurables (número, grosor, opacidad, etiquetas). |

> Los modos Espectral y Curvas de Nivel se desactivan automáticamente cuando se selecciona "Ver solo Caseras Perimetrales".

---

## Filtros disponibles

Todos los filtros son **encadenados** (cada uno filtra sobre el resultado del anterior):

| Filtro | Descripción |
|---|---|
| Semana | Semana epidemiológica. |
| Fundo | Nombre del fundo. |
| Módulo | Código de módulo. |
| Lote | Número de lote. |
| Turno | Número de turno. |
| Trampa | Tipo de trampa (excluye perimetrales por defecto). |
| Caseras Perimetrales | Checkbox para ver solo trampas perimetrales (cambia modo a GPS, deshabilita vectores y curvas). |
| Rango de fechas | Filtra por fecha de captura. |
| Semaforización | Checkboxes para incluir/excluir categorías (⚪🟢🟡🟠🔴). |

---

## Panel de KPIs

Muestra 4 tarjetas en la parte superior del mapa:

| KPI | Descripción |
|---|---|
| Capturas totales | Suma de capturas filtradas. Muestra delta % vs. semana anterior. |
| Promedio / trampa | Capturas totales / trampas activas. Muestra delta % vs. semana anterior. |
| Trampas activas | Número de trampas únicas en el filtro actual. |
| Zonas en alerta | Lotes con categoría roja (> 3 capturas). Se muestra en naranja si > 0. |

---

## Exportar a GitHub Pages

Al presionar **"Publicar HTML"**:
1. Construye el HTML completo (`html_with_data` = HTML del mapa + JSON de datos embebido).
2. Lo sube vía GitHub API (PUT) a:
   - `mapa_mosca.html` (siempre sobreescribe el archivo fijo).
   - `historico/S{semana}/mapa_{sufijo}.html` (versión histórica).
3. Retorna la URL pública de GitHub Pages del histórico.

El sufijo del archivo se construye con: año, semana, fundo y trampa activos (ej: `A2026_S5_F-ARENA_AZUL_T-JACKSON`).

---

## Captura PNG

Al presionar **"PNG"**:

- **Windows (local):** usa **Selenium** con ChromeDriver (gestionado por `webdriver-manager`).
- **Linux/Cloud (Streamlit Cloud):** usa **Playwright** con Chromium headless.

En ambos casos:
1. Guarda el HTML en un archivo temporal.
2. Abre el navegador headless y espera carga del mapa.
3. Ejecuta `activarModoPNGGeneral()` (función JS en el HTML).
4. Detecta bounds de las capas Leaflet y hace `fitBounds`.
5. Captura screenshot del elemento `#mapContainer`.
6. Sube el PNG a GitHub (`historico_png/S{semana}/mapa_{sufijo}.png`).
7. Ofrece descarga directa con botón.

---

## Instalación y dependencias

### Python >= 3.10 requerido (uso de `int | None` en type hints)

```bash
pip install streamlit pandas numpy pyodbc msal lxml shapely scipy matplotlib pillow
```

### Para captura PNG en Windows (local):
```bash
pip install selenium webdriver-manager
```

### Para captura PNG en Linux / Streamlit Cloud:
```bash
pip install playwright
playwright install chromium
```

### Tabla completa de dependencias

| Paquete | Uso |
|---|---|
| `streamlit` | Framework de la aplicación web |
| `pandas` | Manipulación de datos tabulares |
| `numpy` | Cálculos numéricos y grillas |
| `pyodbc` | Conexión ODBC a SQL Server / Microsoft Fabric |
| `msal` | Autenticación Azure AD (Microsoft Authentication Library) |
| `lxml` | Parseo de KML dentro del KMZ |
| `shapely` | Geometría vectorial (máscaras, borde de polígono para perimetrales) |
| `scipy` | Filtro gaussiano y árbol KD de vecinos cercanos |
| `matplotlib` | Generación de contornos (contourf / contour) |
| `pillow` | Procesamiento de imagen para PNG |
| `selenium` | Captura PNG en Windows |
| `webdriver-manager` | Descarga automática de ChromeDriver |
| `playwright` | Captura PNG en entornos Linux/Cloud |

---

## Configuración de secretos

El archivo `.streamlit/secrets.toml` debe contener:

```toml
GITHUB_TOKEN        = "ghp_xxxxxxxxxxxx"   # Token con permisos repo write
GITHUB_TOKEN_KMZ    = "ghp_xxxxxxxxxxxx"   # Token para leer el KMZ privado
GITHUB_OWNER        = "controloperacionalprize-boss"
GITHUB_REPO         = "mapa_html"
GITHUB_BRANCH       = "main"

[database]
server   = "tu-servidor.database.windows.net"
database = "nombre-base-de-datos"
username = "usuario@dominio.com"
```

> `GITHUB_TOKEN_KMZ` puede ser el mismo token que `GITHUB_TOKEN` si el repositorio del KMZ y el de páginas son del mismo owner.

---

## Estructura de archivos requerida

```
PRODUCCION_MOSCAV2/
├── mapa_moscav4.py          ← Este script
├── mapa_streamlit_js.html   ← Componente Leaflet.js (OBLIGATORIO)
└── data/
    └── MODULOS_PRIZE_PAIJAN.kmz  ← Fallback local si GitHub no disponible
```

El KMZ principal se descarga automáticamente desde el repositorio GitHub privado `controloperacionalprize-boss/CAMPO_RENDIMIENTO`. Si la descarga falla, busca el archivo en `data/MODULOS_PRIZE_PAIJAN.kmz`.

---

## Casuísticas y comportamiento esperado

### 1. Sin conexión a SQL Server / Fabric

- La función `_get_access_token()` lanza el **Device Flow**: muestra en pantalla el código que el usuario debe introducir en `microsoft.com/devicelogin`.
- Si el token falla, retorna un DataFrame vacío con todas las columnas (evita errores de `KeyError` en el resto del script).
- El mapa se muestra vacío pero funcional.

### 2. Token MSAL expirado

- MSAL intenta renovar silenciosamente con `acquire_token_silent`. Si falla, relanza el Device Flow.
- El token y la query SQL están separados en dos funciones para que solo la query (sin widgets) se cachee con `@st.cache_data`.

### 3. KMZ no descargable desde GitHub

- Se muestra error en la barra lateral: `❌ KMZ HTTP 404` o `❌ KMZ Error`.
- Intenta usar el archivo local en `data/MODULOS_PRIZE_PAIJAN.kmz`.
- Si tampoco existe localmente, `kmz_polygons = []` y no se pueden trazar polígonos ni calcular centroides.

### 4. Lote sin match en el KMZ

- Se muestra un expander al pie del mapa: `⚠️ N lotes sin match KMZ`.
- La tabla muestra la clave Excel (`AQ|MOD|TUR|LOTE`) y las claves KMZ disponibles para ese módulo (hasta 4 sugerencias).
- Causas comunes:
  - Nombre de lote con formato distinto al KMZ (ej: `115B` en Excel vs `115` en KMZ).
  - Turno con formato SENASA compuesto (`M01-T3`) que se normaliza correctamente, pero el KMZ lo tiene como turno simple.
  - El fundo no está mapeado en `fundo_to_aq()`.

### 5. Trampa perimetral

- Al activar "Ver solo Caseras Perimetrales":
  - El punto se ubica en el **borde** del polígono del lote (no en el centroide).
  - El modo de interpolación se fuerza a "GPS (si existe)".
  - Se ocultan los controles de Vectores de Propagación y Curvas de Nivel.
  - Si el polígono no se encuentra en el KMZ, hace fallback al centroide.

### 6. Menos de 3 lotes con centroide KMZ

- Los modos Espectral y Curvas de Nivel no generan contornos (requieren mínimo 3 puntos).
- Se retorna `{"fills": [], "lines": []}` sin error.

### 7. Vectores de propagación

- Disponibles en modo Normal / Espectral.
- Permiten configurar densidad, longitud de flecha, tamaño de punta, magnitud mínima y color.
- Se calculan en el lado JavaScript con el gradiente del campo gaussiano.

### 8. Publicar HTML con filtros activos

- El sufijo del archivo histórico refleja exactamente los filtros aplicados (año, semana, fundo, trampa).
- Siempre sobreescribe `mapa_mosca.html` (URL fija para compartir).
- Crea adicionalmente `historico/S{N}/mapa_{sufijo}.html` para trazabilidad histórica.

### 9. Captura PNG en Streamlit Cloud

- Usa Playwright porque Selenium no funciona en entornos Linux sin display.
- Instala Chromium automáticamente con `playwright install chromium` en tiempo de ejecución.
- Si la captura del elemento `#mapContainer` falla, hace screenshot completo de la página.

### 10. Año de datos hardcodeado

- La query filtra `WHERE anio = 2026` directamente en Python después de cargar todos los datos.
- Para cambiar el año hay que modificar la línea: `df = df[df["anio"] == 2026].copy()` en `_query_mosquita()`.

---

## Ejecutar la aplicación

```bash
cd PRODUCCION_MOSCAV2
streamlit run mapa_moscav4.py
```

La aplicación queda disponible en `http://localhost:8501`.
