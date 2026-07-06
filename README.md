# SniesManager — Pipelines SNIES + Indicador de Oportunidad de Portafolio

Aplicación de escritorio (Tkinter) y pipelines ETL para trabajar con información del **SNIES** y construir el **Indicador de oportunidad de portafolio** (estudio de mercado por categoría) de la Universidad EAFIT.

El sistema tiene dos flujos principales:

| Flujo | Qué hace | Salida principal |
|-------|----------|------------------|
| **Pipeline SNIES** | Descarga/normaliza `Programas.xlsx`, detecta programas nuevos, clasifica referentes EAFIT con ML y permite ajustes manuales | `outputs/Programas.xlsx` |
| **Indicador de portafolio** | Fases 1–5: categorías → insumos históricos → sábana → scoring → Excel nacional y segmentos; Fase 7: valorización por programa × región | `outputs/estudio_de_mercado/*.xlsx` |

---

## Características principales

### Interfaz gráfica (`app/main.py`)

- Menú principal con módulos por funcionalidad, **tooltips** y **bloqueo de UI** durante procesos largos.
- Tarjeta **«Próximos pasos»** cuando falta algo clave (carpeta del proyecto, `Programas.xlsx`, Excel Fase 1).
- **Verificar / Reparar** estado del sistema (Internet, `ref/`, modelos ML, permisos en `outputs/`).
- **Configuración del Sistema** (⚙️ en Utilidades): año activo, semáforo de archivos, SMLMV y carga de insumos.

### Pipeline SNIES

- Descarga automática de `Programas.xlsx` vía Selenium + Chrome.
- Normalización, detección de programas nuevos e histórico.
- Clasificación de referentes EAFIT con ML (embeddings + Random Forest).
- **Auto-catálogo EAFIT** (institución 1712): programas propios nuevos se registran en `catalogoOfertasEAFIT.csv` y se excluyen del clasificador ML.
- Ajuste manual de emparejamientos e imputación de `ÁREA_DE_CONOCIMIENTO`.

### Indicador de oportunidad de portafolio

| Fase | Descripción | Artefacto clave |
|------|-------------|-----------------|
| **1** | Base maestra: cada programa SNIES → `CATEGORIA_FINAL` (ML + reglas) | `outputs/temp/base_maestra.parquet` + export `Base_Programas_Categoria_F1_*.xlsx` |
| **2** | Insumos históricos: matriculados, inscritos, primer curso, graduados, OLE | CSV en `outputs/historico/raw/` |
| **3** | Sábana consolidada única | `outputs/temp/sabana_consolidada.parquet` |
| **4** | Agregado por categoría, métricas (AAGR, CAGR, participación, señales) y **scoring ponderado** | `outputs/temp/agregado_categorias.parquet` |
| **5** | Excel nacional con hojas `total`, `total_esp`, `total_mae`, `total_pregrado`, `resumen_ejecutivo`, `cambios_vs_anterior`, etc. | `Estudio_Mercado_Colombia.xlsx` |
| **Segmentos** | Bogotá, Antioquia, Eje Cafetero, Virtual (métricas recalculadas por subconjunto) | `Estudio_Mercado_<Segmento>.xlsx` |
| **7 — Valorización** | Métricas mercado (M) y referentes (R) por programa EAFIT × región; `CAL_INTEGRADA = 0.4·M + 0.6·R` | `Programas_para_valorizacion_output.xlsx` |

**Métricas destacadas (Fase 4–5):**

- **AAGR robusto** con umbrales fijos de calificación (alineados al proceso manual).
- **Señal de tendencia** (último año) y **señal post-pandemia** (base 2021 en adelante).
- **Proyección de primer curso**: modelo híbrido lineal/logarítmico (mejor R²); horizonte de **6 años** desde `AÑO_FIN_DATOS` (ej. con 2024 → columnas 2025–2030); etiqueta `TIPO_PROYECCION` (EXPANSIÓN, CRECIENTE, ESTABLE, etc.).
- Encabezados del Excel **dinámicos** respecto al año activo (CAGR, proyección, «vs AÑO_FIN_DATOS»).

### Editor de resultados

Página para abrir, filtrar, editar y guardar el Excel final del estudio de mercado.

---

## Estructura del proyecto

```
proyectoMejora/
├── app/
│   └── main.py                    # GUI (menú, páginas, ConfiguracionDialog)
├── etl/
│   ├── descargaSNIES.py           # Descarga Programas.xlsx (Selenium)
│   ├── procesamientoSNIES.py      # Programas nuevos + histórico
│   ├── clasificacionProgramas.py  # ML referentes EAFIT + auto-catálogo
│   ├── imputacionAreas.py         # Imputación ÁREA_DE_CONOCIMIENTO
│   ├── mercado_pipeline.py        # Estudio de mercado (Fases 1–5 + segmentos)
│   ├── valorizacion_pipeline.py   # Fase 7 — Valorización
│   ├── scoring.py                 # Scoring ponderado (Fase 4)
│   ├── scraper_matriculas.py      # Lectura local matriculados/inscritos/primer curso/graduados
│   ├── scraper_ole.py             # Indicadores OLE
│   ├── merge_incremental.py       # Merge incremental + snapshots
│   ├── normalizacion*.py          # Normalización y limpieza SNIES
│   ├── pipeline_logger.py         # Logging centralizado
│   ├── exceptions_helpers.py      # I/O robusto (Excel/CSV con reintentos)
│   └── config.py                  # Rutas, años, SMLMV, benchmarks
├── models/                        # Modelos ML (.pkl)
├── outputs/
│   ├── Programas.xlsx
│   ├── historico/raw/             # CSV intermedios (Fase 2)
│   ├── temp/                      # Parquets y cachés
│   └── estudio_de_mercado/        # Excels finales + histórico
├── ref/
│   ├── backup/                    # Insumos SNIES locales (ver abajo)
│   ├── referentesUnificados.csv
│   └── catalogoOfertasEAFIT.csv
├── logs/
│   └── pipeline.log
├── config.json                    # Configuración de sesión (opcional)
├── config.json.example
├── requirements.txt
└── build_exe.py                   # Empaquetado PyInstaller
```

---

## Requisitos

- **Python 3.10+** (recomendado 3.12)
- **Google Chrome** instalado (descarga automatizada SNIES)
- Entorno virtual recomendado

---

## Instalación

```bash
python -m venv .venv

# Windows (PowerShell):
.venv\Scripts\Activate.ps1

# Windows (cmd):
.venv\Scripts\activate.bat

# Linux / macOS:
# source .venv/bin/activate

pip install -r requirements.txt
```

---

## Configuración

### `config.json` (raíz del proyecto)

`etl/config.py` lee este archivo al arrancar. Claves habituales:

| Clave | Descripción |
|-------|-------------|
| `base_dir` | Carpeta raíz del proyecto (útil si mueves datos fuera del repo) |
| `outputs_dir`, `ref_dir`, `models_dir`, `logs_dir` | Rutas absolutas alternativas |
| `AÑO_FIN_DATOS` | Último año con datos completos (default en código: **2024**) |
| `AÑO_INICIO_HISTORICO` | Primer año de matrícula/inscritos (default: **2019**) |
| `AÑO_INICIO_PRIMER_CURSO` | Primer año en serie de primer curso (default: **2014**) |
| `SMLMV_POR_ANO` | Diccionario `{ "2024": 1300000, ... }` para normalizar salarios OLE |
| `SMLMV` | SMLMV de sesión (override puntual) |
| `BENCHMARK_COSTO_*` | Benchmarks de costo por nivel (Pregrado, Esp, Maestría, etc.) |
| `umbral_referente` | Probabilidad mínima (0–1) para marcar referente; default **0.70** |
| `headless` | `true` = Chrome sin ventana en descarga SNIES |
| `max_wait_download_sec` | Timeout de descarga (default 180 s) |

Copia `config.json.example` como punto de partida.

> **Importante — cambio de año:** `AÑO_FIN_DATOS` se importa una sola vez al arrancar en `mercado_pipeline.py`, `scoring.py` y `valorizacion_pipeline.py`. Si cambias el año en **Configuración del Sistema** y guardas, debes **cerrar y reabrir SniesManager** para que los cálculos usen el año nuevo. Si ejecutas el pipeline sin reiniciar, `validar_archivos_entrada()` lo detecta y **detiene la ejecución** con un mensaje explícito (no calcula en silencio con el año viejo).

### Archivos de referencia ML (`ref/`)

- `referentesUnificados.csv` — entrenamiento del clasificador.
- `catalogoOfertasEAFIT.csv` — catálogo de programas EAFIT.
- `Referente_Categorias.xlsx` (o `.csv`) — matriz de categorías para Fase 1 (hoja según `HOJA_REFERENTE_CATEGORIAS` en `config.py`).

### Insumos locales — Estudio de mercado (`ref/backup/`)

Si falta un archivo/año, el pipeline registra advertencia y continúa con ceros/NaN donde corresponda.

| Tipo | Ubicación / patrón | Cómo agregar desde la GUI |
|------|-------------------|---------------------------|
| **Matrícula** | `ref/backup/matriculas/matriculados_YYYY.xlsx` | Configuración → 📁 Agregar archivo |
| **Inscritos** | `ref/backup/inscritos/inscritos_YYYY.xlsx` | Configuración → 📁 Agregar archivo |
| **Graduados** | `ref/backup/graduados/graduados_YYYY.xlsx` | Configuración → 📁 Agregar archivo |
| **Primer curso** | `ref/backup/matriculas primer curso/matriculas_primercurso_ESTANDARIZADO.xlsx` | Configuración → 📥 Agregar año nuevo al consolidado (normaliza y fusiona) |
| **IES** | `ref/backup/ies/Instituciones.xlsx` | Manual |
| **OLE** (opcional) | `ref/backup/ole_indicadores.csv` o `.xlsx` | Manual |

**Primer curso consolidado:** a diferencia de los otros tres tipos, primer curso usa un **único Excel estandarizado** con todos los años (columna `AÑO`). El pipeline lee la serie 2014–`AÑO_FIN_DATOS` desde ese archivo. Los archivos anuales `primer_curso_YYYY.xlsx` en la misma carpeta son insumos para **incorporar años nuevos** al consolidado.

### Rutas útiles (`etl/config.py`)

- `ARCHIVO_PROGRAMAS` → `outputs/Programas.xlsx`
- `ARCHIVO_ESTUDIO_MERCADO` → `outputs/estudio_de_mercado/Estudio_Mercado_Colombia.xlsx`
- `CHECKPOINT_BASE_MAESTRA` → `outputs/temp/base_maestra.parquet`
- `TEMP_DIR` → `outputs/temp/`

---

## Configuración del Sistema (diálogo ⚙️)

Acceso: menú principal → **Utilidades** → **⚙️ Configuración**.

| Sección | Función |
|---------|---------|
| **Año de datos activo** | Spinner `AÑO_FIN_DATOS` (2019–2035). Semáforo de archivos se actualiza con debounce (sin bloquear la UI). |
| **Archivos en ref/backup/** | ✅ / ❌ / ⏳ por tipo; resumen `N/4 archivos disponibles`. |
| **Primer curso** | Botón para fusionar un archivo SNIES crudo en el consolidado (por año). |
| **Matrícula / Inscritos / Graduados** | Botón para copiar un `.xlsx` con el nombre estándar a la carpeta correcta (sin normalización). |
| **SMLMV por año** | Editor de valores; botón **➕ Agregar año nuevo** para años futuros. |

La ventana tiene **scroll vertical**; los botones Cancelar / Guardar y aplicar quedan fijos al fondo.

---

## Uso

### Ejecutar la aplicación

```bash
python app/main.py
```

O, si usas el ejecutable empaquetado: `SniesManager.exe` (ver `GUIA_EMPAQUETADO.md`).

### Flujo recomendado

```
Pipeline SNIES → Revisión de Áreas (IA) → Ajuste manual → Indicador de portafolio
```

1. **▶️ Ejecutar análisis SNIES** — descarga y clasifica programas.
2. **Revisión de Áreas** — imputa `ÁREA_DE_CONOCIMIENTO` faltante.
3. **Ajuste manual** — corrige referentes EAFIT si hace falta.
4. **Indicador de oportunidad de portafolio** — Fase 1, luego Fases 2–5, segmentos y Valorización.
5. **Resultados Estudio de Mercado** — revisar/editar el Excel exportado.

### Desde el menú también puedes

- **Utilidades:** logs, desbloquear `.pipeline.lock`, abrir `outputs/`, revisar `Base_Programas_Categoria_F1_*.xlsx`, gestionar programas candidatos (Valorización), reentrenar modelo ML, **Configuración del Sistema**.
- **Consolidar archivos (Merge):** oculto en la GUI; la lógica permanece en código.

### Comandos por terminal (opcional)

```bash
# Descargar Programas.xlsx
python etl/descargaSNIES.py

# Procesar programas nuevos
python etl/procesamientoSNIES.py

# Entrenar / clasificar referentes
python etl/clasificacionProgramas.py entrenar
python etl/clasificacionProgramas.py
```

El indicador de portafolio está pensado para ejecutarse desde la GUI (Fases 1–5, segmentos, Fase 7).

---

## Salidas clave

| Artefacto | Ruta |
|-----------|------|
| Programas SNIES | `outputs/Programas.xlsx` |
| Base Fase 1 | `outputs/estudio_de_mercado/Base_Programas_Categoria_F1_<fecha>.xlsx` |
| Estudio nacional | `outputs/estudio_de_mercado/Estudio_Mercado_Colombia.xlsx` |
| Segmentos | `outputs/estudio_de_mercado/Estudio_Mercado_{Bogota,Antioquia,Eje_Cafetero,Virtual}.xlsx` |
| Valorización | `outputs/estudio_de_mercado/Programas_para_valorizacion_output.xlsx` |
| Parquets / cachés | `outputs/temp/*.parquet` |
| Histórico estudio | `outputs/estudio_de_mercado/historico_estudio_de_mercado/` |
| Logs | `logs/pipeline.log` |

---

## Estudio de mercado — notas operativas

- **Validación al inicio:** `validar_archivos_entrada()` comprueba sincronización de año, `Programas.xlsx`, carpetas y años de matrícula antes de Fase 2. Errores bloquean; advertencias permiten continuar.
- **Fase 3:** tras consolidar, limpia CSV intermedios en `outputs/historico/raw/`.
- **Fase 5:** la hoja `cambios_vs_anterior` compara con `agregado_categorias_anterior.parquet` (primera corrida solo crea el snapshot).
- **Segmentos:** reutilizan caché `agregado_<segmento>.parquet` si la sábana no cambió; opción de forzar recálculo en la GUI.
- **Proyección:** `HORIZONTE_PROYECCION_ANOS = 6` en `mercado_pipeline.py`; el bloque Excel «PROYECCIÓN {año+1}-{año+6}» se genera dinámicamente.
- **Valorización (Fase 7):** requiere segmentos previos; `AÑOS_PROY = [2027…2031]` en `valorizacion_pipeline.py` es horizonte de planeación institucional fijo (independiente de `AÑO_FIN_DATOS`).

---

## Modelo de referentes EAFIT

Combina **sentence-transformers** (embeddings multilingües) con **Random Forest** sobre variables derivadas. Entrenamiento con `referentesUnificados.csv`. Umbral configurable: `umbral_referente` en `config.json`.

### Auto-catálogo EAFIT (institución 1712)

Para programas nuevos del SNIES con `CÓDIGO_INSTITUCIÓN` (o padre) **1712**:

1. Si no están en `catalogoOfertasEAFIT.csv`, se añaden automáticamente.
2. Se excluyen del clasificador ML.
3. En `Programas.xlsx`: `ES_REFERENTE='Sí'`, `PROBABILIDAD=1.0`, apuntando al programa propio.

Idempotente: una segunda corrida no duplica filas.

---

## Empaquetado (PyInstaller)

- `build_exe.py`, `GUIA_EMPAQUETADO.md`, `INSTRUCCIONES_EMPAQUETADO.md`
- Con `.exe` en `dist/`, `config.py` intenta usar la carpeta **padre** de `dist/` como raíz para `outputs/`, `ref/` y `models/`.
- Si el proyecto está en OneDrive/SharePoint, marca `ref/`, `models/`, `outputs/` y `config.json` como **«Mantener siempre en este dispositivo»** (ver `INSTRUCCIONES PARA LA EJECUCION.txt`).

---

## Solución de problemas

| Síntoma | Qué revisar |
|---------|-------------|
| Faltan insumos | `ref/backup/`, semáforo en Configuración, `logs/pipeline.log` |
| Año desincronizado | Reiniciar la app tras cambiar `AÑO_FIN_DATOS` en Configuración |
| Excel en uso | Cerrar el archivo y reintentar |
| Columnas `_x/_y` en sábana | Borrar parquet en `outputs/temp/` y re-ejecutar la fase |
| Pipeline bloqueado | Utilidades → Desbloquear (`.pipeline.lock`) |
| Modelo ML ausente | Utilidades → Reentrenar modelo |

---

## Dependencias principales

`pandas`, `numpy`, `scikit-learn`, `sentence-transformers`, `joblib`, `selenium`, `webdriver-manager`, `openpyxl`, `rapidfuzz`, `unidecode`

Listado completo en `requirements.txt`.

---

## Documentación adicional

| Archivo | Contenido |
|---------|-----------|
| `ARCHIVOS_PROYECTO.md` | Inventario detallado de entradas/salidas |
| `docs/ANALISIS_FLUJO_Y_EXCEPCIONES.md` | Flujos y manejo de errores |
| `docs/MEJORAS_SISTEMA.md` | Backlog técnico |
| `GUIA_EMPAQUETADO.md` / `INSTRUCCIONES_EMPAQUETADO.md` | Build del `.exe` |
| `DIAGNOSTICO_SISTEMA.md` | Informe de diagnóstico puntual |
| `INSTRUCCIONES PARA LA EJECUCION.txt` | Sincronización local (OneDrive) |

---

## Desarrollo

```bash
# Verificar sintaxis
python -m py_compile app/main.py etl/mercado_pipeline.py

# Tests puntuales (si existen)
pytest
```

---

*Universidad EAFIT — Prácticas / proyecto de mejora del pipeline SNIES y estudio de mercado.*
