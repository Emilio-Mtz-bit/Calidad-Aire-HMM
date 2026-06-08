# Contexto: Modelación de Calidad del Aire con HMM — Zapopan ZMG

## Objetivo del proyecto

Predecir la calidad del aire a **30 y 60 minutos** en la zona metropolitana de Zapopan usando un **Hidden Markov Model (HMM)**. La "calidad del aire" se trata como una **variable latente** — un estado oculto de la atmósfera que nunca se observa directamente, sino que se infiere de los contaminantes medidos por los sensores.

---

## Dataset

| Campo | Detalle |
|---|---|
| Archivo | `Datos_maestro_sensores.xlsx` |
| Registros | 13,758 filas (filtrado a sensores D) |
| Sensores | `D_P1`, `D_P2` — únicos a usar |
| Período | Nov 2025 (cada sensor tiene su propio rango) |
| Frecuencia | ~1.02 min (D_P1) / ~1.57 min (D_P2) |

### Columnas disponibles

```
sensor_periodo   # ID del sensor — usar solo D_P1 y D_P2
datetime         # timestamp, dtype datetime64
CO2_ppm          # CO2 equivalente — proxy de combustión, NO es contaminante criterio
TVOC_ppb         # VOCs totales — proxy de combustión, sensor MOS no selectivo
AQI              # Índice interno del sensor CCS811 (1–5). NO es IMECA ni EPA AQI
CO_ppm           # Monóxido de carbono — contaminante criterio ✓
SO2_ppm          # Dióxido de azufre — contaminante criterio ✓
O3_ppm           # Ozono troposférico — contaminante criterio ✓
NO2_ppm          # Dióxido de nitrógeno — contaminante criterio ✓
PM2_5_ugm3       # Partícula fina — contaminante más relevante para salud ✓
PM10_ugm3        # Partícula gruesa — contaminante criterio ✓
temp_C           # Temperatura ambiente
hum_pct          # Humedad relativa
Bat_pct          # Batería del sensor — no usar como feature
```

### Gaps en los datos (crítico para HMM)

| Sensor | Gaps > 5 min | Gap máximo |
|---|---|---|
| D_P1 | 0 | 1.5 min — **continuo** |
| D_P2 | 4 | 3,005 min (~50 h) — **crítico** |

> **Importante:** El HMM asume que los datos son una secuencia temporal continua. D_P2 tiene 4 gaps, el mayor de ~50 horas. Hay que segmentar por gaps antes de entrenar. D_P1 es completamente continuo.

### Distribución de AQI en sensores D (sensor interno)

| AQI | Registros aprox. | Nota |
|---|---|---|
| 1 — Bueno | ~2,900 | |
| 2 — Moderado | ~6,700 | Clase dominante |
| 3 — Elevado | ~2,900 | |
| 4 — Alto | ~900 | Clase minoritaria |
| 5 — Crítico | ~280 | Solo en D_P1 |

> D_P1 y D_P2 tienen rangos temporales distintos (Nov 6–11 y Nov 13–25) — no se solapan. Al entrenar el HMM con ambos concatenados, declarar sus segmentos por separado en `lengths`.

---

## Decisiones de diseño químico

### Variables de observación para el HMM

Usar **exclusivamente los contaminantes criterio reales**. Descartar CO2 y el AQI del sensor como features de observación (son índices derivados, no mediciones independientes).

```python
OBSERVATION_FEATURES = [
    'CO_ppm',
    'NO2_ppm',
    'PM2_5_ugm3',
    'O3_ppm',
    'SO2_ppm',
    'TVOC_ppb',   # útil como proxy de combustión aunque no sea criterio
]
```

Opcionalmente agregar `temp_C` y `hum_pct` como covariables que afectan la dispersión.

### Por qué estos y no CO2/AQI

- **CO2** a 400–1500 ppm es concentración atmosférica normal, no un contaminante del aire en sentido regulatorio.
- **AQI** del dataset es el índice interno del sensor CCS811, calculado a partir de CO2 y TVOC. Usarlo como feature de observación en el HMM introduce redundancia con TVOC.
- **PM2.5** es el contaminante más relevante para salud (OMS): el 28.1% de los registros supera el límite OMS de 15 µg/m³.
- **NO2** tiene la mejor autocorrelación entre los gases reales (ACF = 0.43 a lag 20), lo que aporta señal temporal real al HMM.

---

## Cómo funciona el HMM en este contexto

### Estructura del modelo

```
Estado oculto Z(t) ∈ {1, 2, ..., K}     ← régimen de calidad del aire (latente)
Observación X(t) = [CO, NO2, PM2.5, O3, SO2, TVOC](t)  ← mediciones del sensor

El modelo aprende:
  π   = distribución de estado inicial
  A   = matriz de transición K×K  (P(Z_t+1 | Z_t))
  μ_k = media de observaciones en estado k
  Σ_k = covarianza de observaciones en estado k
```

### Interpretación química esperada de los estados

El modelo aprenderá los estados de forma no supervisada. Con K=4 o K=5, es esperable que encuentre:

| Estado (esperado) | Firma química | Horario típico |
|---|---|---|
| Aire limpio | CO bajo, PM2.5 bajo, todo bajo | Madrugada, noches ventosas |
| Tráfico matutino | CO↑↑, NO2↑↑, TVOC↑↑, PM2.5↑ | 6–9 AM |
| Fotoquímico | O3↑↑, NO2 moderado, CO bajo | 12–3 PM |
| Episodio mixto | PM2.5↑↑, CO↑, NO2↑, TVOC↑ | Variable, eventos críticos |

> Esto es una hipótesis química. Los estados reales los define el modelo a partir de los datos — pueden ser diferentes o más matizados.

---

## Preprocesamiento — pasos obligatorios

### 0. Filtrar solo sensores D

```python
SENSORES_D = ['D_P1', 'D_P2']
df = df[df['sensor_periodo'].isin(SENSORES_D)].copy()
# Resultado: 13,758 registros
```

### 1. Segmentar por gaps

```python
# Identificar inicio de cada segmento continuo
# Un gap > 5 min rompe la continuidad de la serie

def segmentar_por_gaps(df, sensor, umbral_min=5):
    sub = df[df['sensor_periodo'] == sensor].copy().sort_values('datetime')
    sub['diff_min'] = sub['datetime'].diff().dt.total_seconds() / 60
    sub['gap'] = sub['diff_min'] > umbral_min
    sub['segment_id'] = sub['gap'].cumsum()
    return sub
```

Cada `segment_id` es una secuencia temporal independiente. Entrenar el HMM concatenando segmentos pero declarando sus longitudes por separado (`lengths` en hmmlearn).

### 2. Calcular horizontes de predicción

```python
# Pasos necesarios según frecuencia de cada sensor D
FREQ = {
    'D_P1': 1.02,   # min por registro — Nov 6–11 2025, 6,208 registros
    'D_P2': 1.57,   # min por registro — Nov 13–25 2025, 7,550 registros
}

H_30 = {s: int(np.ceil(30 / f)) for s, f in FREQ.items()}  # D_P1: 30 pasos | D_P2: 20 pasos
H_60 = {s: int(np.ceil(60 / f)) for s, f in FREQ.items()}  # D_P1: 59 pasos | D_P2: 39 pasos
```

### 3. Normalización

```python
from sklearn.preprocessing import StandardScaler

# Ajustar SOLO sobre el conjunto de train, aplicar sobre train y test
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled  = scaler.transform(X_test)
```

### 4. Split temporal — nunca aleatorio

```python
# Cortar por fecha, no por índice aleatorio
# Usar aproximadamente 80% del rango temporal como train

cutoff = df['datetime'].quantile(0.8)  # ajustar por sensor
train = df[df['datetime'] <= cutoff]
test  = df[df['datetime'] >  cutoff]
```

---

## Implementación con hmmlearn

### Instalación

```bash
pip install hmmlearn
```

### Modelo base

```python
from hmmlearn import hmm
import numpy as np

# GaussianHMM: observaciones continuas con distribución Gaussiana por estado
model = hmm.GaussianHMM(
    n_components=K,          # número de estados ocultos (seleccionar con BIC)
    covariance_type='full',  # covarianza completa — captura correlaciones entre contaminantes
    n_iter=100,              # iteraciones EM
    random_state=42,
    verbose=True
)

# X_train: array (n_samples, n_features)
# lengths: lista con el largo de cada segmento continuo
model.fit(X_train_scaled, lengths=lengths_train)
```

### Selección del número de estados K con BIC

```python
from hmmlearn import hmm

def calcular_bic(X, lengths, k):
    model = hmm.GaussianHMM(n_components=k, covariance_type='full',
                             n_iter=100, random_state=42)
    model.fit(X, lengths=lengths)
    log_likelihood = model.score(X, lengths=lengths)
    
    # Número de parámetros libres
    n_features = X.shape[1]
    n_params = (k * k - k +           # matriz de transición
                k * n_features +       # medias
                k * n_features**2)     # covarianzas (full)
    
    bic = -2 * log_likelihood + n_params * np.log(len(X))
    return bic, model

# Probar K de 2 a 8
resultados = {}
for k in range(2, 9):
    bic, modelo = calcular_bic(X_train_scaled, lengths_train, k)
    resultados[k] = {'bic': bic, 'modelo': modelo}
    print(f"K={k} → BIC={bic:.2f}")

K_optimo = min(resultados, key=lambda k: resultados[k]['bic'])
```

### Decodificar estados y hacer predicciones

```python
# 1. Decodificar secuencia de estados (algoritmo de Viterbi)
estados_ocultos = model.predict(X_test_scaled, lengths=lengths_test)

# 2. Probabilidades de estado en cada paso (algoritmo forward)
log_probs, posteriors = model.score_samples(X_test_scaled, lengths=lengths_test)
# posteriors: array (n_samples, K) — P(Z_t = k | X_0:t)

# 3. Predecir estado en t+H
def predecir_estado_futuro(model, posterior_actual, H):
    """
    Proyecta la distribución de estado H pasos hacia adelante
    multiplicando por la matriz de transición H veces.
    
    posterior_actual: array (K,) — P(Z_t | datos observados hasta t)
    H: número de pasos hacia adelante
    """
    A = model.transmat_
    posterior_futuro = posterior_actual
    for _ in range(H):
        posterior_futuro = posterior_futuro @ A
    return posterior_futuro   # P(Z_{t+H} | datos hasta t)

# Ejemplo: predecir estado en t+30min para el último punto del test
ultimo_posterior = posteriors[-1]   # P(Z_t | todos los datos)
pred_30min = predecir_estado_futuro(model, ultimo_posterior, H=20)

print("Probabilidad por régimen en t+30 min:")
for k in range(model.n_components):
    print(f"  Estado {k+1}: {pred_30min[k]*100:.1f}%")
```

### Obtener concentraciones predichas

```python
# La concentración esperada en t+H es la media ponderada de las emisiones de cada estado
def predecir_concentracion(model, posterior_futuro, scaler):
    """
    Calcula la concentración esperada como expectativa sobre los estados.
    """
    means = model.means_          # (K, n_features) — medias en espacio normalizado
    
    # Expectativa: suma ponderada de medias de cada estado
    concentracion_norm = np.dot(posterior_futuro, means)  # (n_features,)
    
    # Desnormalizar
    concentracion = scaler.inverse_transform(concentracion_norm.reshape(1, -1))
    return concentracion.flatten()

conc_predicha = predecir_concentracion(model, pred_30min, scaler)
for nombre, valor in zip(OBSERVATION_FEATURES, conc_predicha):
    print(f"  {nombre}: {valor:.4f}")
```

---

## Evaluación del modelo

### Métricas para los estados ocultos

```python
# Como los estados no tienen etiqueta previa, evaluar con métricas internas
from sklearn.metrics import silhouette_score

# Silhouette sobre el espacio de observaciones
sil = silhouette_score(X_test_scaled, estados_ocultos)
print(f"Silhouette score: {sil:.3f}")  # más alto = estados más compactos y separados

# Log-likelihood en test (mayor = mejor)
ll_test = model.score(X_test_scaled, lengths=lengths_test)
print(f"Log-likelihood test: {ll_test:.2f}")
```

### Comparar predicción de AQI vs AQI real

```python
# Mapear estado predicho → AQI usando la moda de AQI observado en cada estado
from scipy import stats

estado_a_aqi = {}
for k in range(model.n_components):
    mask = estados_ocultos == k
    aqi_en_estado = y_aqi_test[mask]   # AQI real cuando el modelo dice "estado k"
    estado_a_aqi[k] = stats.mode(aqi_en_estado).mode[0]

# AQI predicho como el estado más probable en t+H
aqi_predicho = estado_a_aqi[np.argmax(pred_30min)]
```

### Interpretación química de los estados aprendidos

```python
# Ver la media de cada contaminante por estado (en escala original)
means_original = scaler.inverse_transform(model.means_)
df_estados = pd.DataFrame(
    means_original,
    columns=OBSERVATION_FEATURES,
    index=[f'Estado {k+1}' for k in range(model.n_components)]
)
print(df_estados.round(4))
# Analizar cuál estado corresponde a "tráfico", "fotoquímico", etc.
# basándose en las concentraciones relativas de cada contaminante
```

---

## Estructura recomendada del notebook

```
1. Carga y exploración del dataset
2. Preprocesamiento
   2.1 Identificar y segmentar gaps
   2.2 Seleccionar features de observación
   2.3 Calcular lengths por segmento
   2.4 Split temporal train/test
   2.5 StandardScaler (fit solo en train)
3. Selección de K (número de estados)
   3.1 Loop BIC para K = 2..8
   3.2 Graficar BIC vs K
   3.3 Seleccionar K óptimo
4. Entrenamiento del GaussianHMM final
5. Decodificación de estados (Viterbi)
   5.1 Visualizar secuencia de estados en el tiempo
   5.2 Interpretar cada estado con medias de contaminantes
6. Predicción a t+30 min y t+60 min
   6.1 Función de proyección con matriz de transición
   6.2 Distribución de probabilidades por estado
   6.3 Concentración esperada por contaminante
7. Evaluación
   7.1 Log-likelihood en test
   7.2 Silhouette score
   7.3 Comparación AQI predicho vs AQI real
   7.4 Matriz de confusión estado predicho → AQI
8. Visualizaciones
   8.1 Serie temporal con estados coloreados
   8.2 Heatmap de la matriz de transición A
   8.3 Distribuciones de emisión por estado (violinplots)
   8.4 Probabilidades de estado predichas a t+H
```

---

## Notas importantes para Claude Code

- Usar `hmmlearn.hmm.GaussianHMM` con `covariance_type='full'`
- El argumento `lengths` en `.fit()` y `.score()` es obligatorio cuando hay múltiples segmentos — sin él, hmmlearn asume que toda la secuencia es continua
- Los estados del HMM no tienen orden intrínseco — el estado "1" no es necesariamente "limpio". Hay que interpretarlos post-hoc con las medias de emisión
- Si el modelo no converge, reducir `n_iter` o cambiar a `covariance_type='diag'` como punto de partida
- Para predicción multi-paso, la incertidumbre crece con H — a 60 min la distribución de estado tenderá a la distribución estacionaria del modelo
- El AQI del sensor (`AQI` en el dataset) puede usarse como **etiqueta de validación** para evaluar qué tan bien los estados aprendidos se corresponden con los niveles de calidad del aire, pero NO como feature de entrada al HMM
