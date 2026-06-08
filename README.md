# 🌫️ Machine Learning para la Calidad del Aire — ZMG

Proyecto interdisciplinario entre **Ingeniería en Ciencias de Datos** e **Ingeniería Química** para modelar y predecir regímenes atmosféricos en la Zona Metropolitana de Guadalajara usando Hidden Markov Models.

---

## 📋 Descripción

Los sensores de calidad del aire en Zapopan registran concentraciones de contaminantes cada minuto (CO, NO₂, PM2.5, O₃, SO₂, TVOC). El reto no es solo predecir el AQI puntual, sino **entender qué estado químico tiene la atmósfera** y cuánto tiempo va a durar.

Este proyecto implementa un **Hidden Markov Model (HMM)** que infiere cuatro regímenes atmosféricos latentes a partir de las señales de los sensores, y proyecta la distribución de estados hacia adelante en horizontes de **t+30 y t+60 minutos**.

---

## 🗂️ Estructura del repositorio

```
├── analisis_exploratorio.ipynb       # Fase 2: EDA — distribuciones, correlaciones, patrones temporales
├── etl_calidad_aire.ipynb            # Fase 3: Pipeline ETL completo
├── modelacion.ipynb                  # Fase 4: Modelos supervisados (Ridge, LightGBM, Random Forest)
├── hmm_calidad_aire.ipynb            # Fase 4-5: Modelo final HMM con K=4 estados
├── hmm_calidad_aire_backup.ipynb     # Backup del notebook HMM
├── Guion_Video_Final.md              # Guion completo del video de presentación ejecutiva
├── contexto_HMM_calidad_aire_zapopan.md  # Contexto del problema y decisiones de diseño
├── Fase 1_ Comprensión del Problema.pdf  # Entregable Fase 1
├── Proyecto - Machine learning para la calidad del aire.html  # Especificación del proyecto
├── modelacion.html                   # Notebook de modelación exportado a HTML
└── Data/
    ├── Datos_maestro_sensores.xlsx   # Dataset original con todos los sensores
    ├── Nomenclatura_Maestro.xlsx     # Diccionario de variables y nomenclatura
    ├── dataset_etl_completo.csv      # Dataset procesado por el ETL
    ├── dataset_train.csv             # Partición de entrenamiento
    ├── dataset_val.csv               # Partición de validación
    ├── dataset_test.csv              # Partición de prueba
    ├── diccionario_variables.csv     # Descripción de todas las variables
    ├── scaler_params.csv             # Parámetros del StandardScaler (para reproducibilidad)
    ├── 01_*.png — 07_*.png           # Gráficas del análisis exploratorio
    ├── ETL_01_*.png — ETL_04_*.png   # Gráficas del proceso ETL
    ├── MOD_01_*.png — MOD_09_*.png   # Gráficas de los modelos supervisados
    └── HMM_01_*.png — HMM_07_*.png  # Gráficas del modelo HMM final
```

---

## 🔬 Fases del proyecto

| Fase | Descripción | Notebook |
|------|-------------|----------|
| 1 | Comprensión del problema | `Fase 1_ Comprensión del Problema.pdf` |
| 2 | Análisis exploratorio de datos (EDA) | `analisis_exploratorio.ipynb` |
| 3 | Proceso ETL | `etl_calidad_aire.ipynb` |
| 4 | Modelado y evaluación | `modelacion.ipynb` + `hmm_calidad_aire.ipynb` |
| 5 | Evaluación y comunicación de resultados | `Guion_Video_Final.md` |

---

## 🤖 Modelo final — HMM con K=4 estados

El modelo aprende cuatro regímenes atmosféricos **sin supervisión**:

| Estado | Nombre | Firma química destacada | AQI modal | % en test |
|--------|--------|------------------------|-----------|-----------|
| E1 | Industrial / Diesel | SO₂ máximo (0.039 ppm), sin NO₂ elevado | 2 | 18.3% |
| E2 | Línea base | Todos los contaminantes moderados-bajos | 2 | 52.7% |
| E3 | Combustión intensa | CO=2.48 ppm (5× promedio), PM2.5=17.1 µg/m³ | 3 | 28.7% |
| E4 | Fotoquímico | O₃=0.171 ppm, NO₂=0.078 ppm — transitorio | 2 | 0.3% |

### Predicción multi-step desde episodio de combustión (E3)

| Horizonte | P(E2 baseline) | P(E3 combustión) | CO esperado | AQI predicho |
|-----------|----------------|-----------------|-------------|--------------|
| t = 0 | 0% | **100%** | 2.48 ppm | — |
| t + 30 min | 47.7% | 47.7% | 1.38 ppm | **3 (elevado)** |
| t + 60 min | **65.0%** | 28.3% | 0.98 ppm | **2 (moderado)** |

### Métricas de evaluación

| Métrica | Valor |
|---------|-------|
| Log-likelihood en test | -14,035 |
| Silhouette score (test) | 0.229 |
| Sobreajuste train-test | Sin sobreajuste (diff = -9,086) |
| Estados presentes en test | 4/4 |

---

## 📊 Modelos supervisados evaluados (Fase 4)

Antes de seleccionar el HMM se evaluaron modelos de regresión supervisada para predecir el AQI:

| Modelo | RMSE (val) | R² (val) |
|--------|-----------|---------|
| Regresión Lineal (OLS) | 0.3481 | 0.8030 |
| Ridge (α=1.0) | 0.3482 | 0.8029 |
| LightGBM | 0.2099 | 0.9284 |
| **Random Forest** | **0.2040** | **0.9323** |

El HMM fue seleccionado sobre Random Forest porque ofrece **interpretación de regímenes**, **cuantificación de incertidumbre** y **predicción de la duración de los episodios** — capacidades que ningún modelo supervisado puede entregar.

---

## ⚙️ Requisitos

```bash
pip install pandas numpy matplotlib scikit-learn hmmlearn scipy openpyxl lightgbm
```

| Librería | Uso |
|----------|-----|
| `hmmlearn` | Entrenamiento del GaussianHMM |
| `scikit-learn` | Preprocesamiento, métricas (Silhouette) |
| `lightgbm` | Modelo de boosting en Fase 4 |
| `scipy` | Estadísticas (moda para mapeo AQI) |
| `openpyxl` | Lectura del Excel maestro |

---

## 🚀 Cómo ejecutar

1. Clonar el repositorio:
```bash
git clone https://github.com/Emilio-Mtz-bit/Calidad-Aire-HMM.git
cd Calidad-Aire-HMM
```

2. Instalar dependencias:
```bash
pip install pandas numpy matplotlib scikit-learn hmmlearn scipy openpyxl lightgbm
```

3. Ejecutar los notebooks **en orden**:
   - `analisis_exploratorio.ipynb`
   - `etl_calidad_aire.ipynb`
   - `modelacion.ipynb`
   - `hmm_calidad_aire.ipynb`

> Los notebooks asumen que `Data/Datos_maestro_sensores.xlsx` existe en la ruta relativa `Data/`.

---

## 📍 Datos

- **Fuente:** Sensores D_P1 y D_P2 en Zapopan, ZMG
- **Periodo:** 6 – 25 de noviembre de 2025
- **Registros:** 13,758 observaciones
- **Frecuencia:** ~1 minuto por registro (D_P1: 1.02 min | D_P2: 1.57 min)
- **Variables:** CO, NO₂, PM2.5, O₃, SO₂, TVOC, AQI, temperatura, humedad

---

## 🤝 Colaboración interdisciplinaria

Proyecto desarrollado en colaboración con estudiantes de **Ingeniería Química**, quienes validaron la coherencia química de los cuatro regímenes aprendidos e interpretaron las firmas de contaminantes (e.g., SO₂ alto sin NO₂ como indicador de combustóleo industrial; la naturaleza transitoria del ozono troposférico en zona urbana).

---

## 🏫 Contexto académico

**Materia:** Optimización No Lineal — Semestre 6  
**Institución:** Tecnológico de Monterrey  
**Entrega:** Fase 5 — Evaluación y comunicación de resultados  
**Fecha:** 7 de junio de 2026
