# Guion — Video de Presentación Ejecutiva
## Machine Learning para la Calidad del Aire · Zona Metropolitana de Guadalajara
### Entrega Final — Fase 5 | Fecha: 7 de junio de 2026

---

**Duración total:** 7 minutos exactos  
**Formato:** Diapositivas (PowerPoint/Canva) + cámara  
**Participantes:** 4 personas (~1 min 45 seg cada una)  
**Modelo final seleccionado:** Hidden Markov Model (HMM) con K=4 estados

---

## ESTRUCTURA GENERAL

| Sección | Persona | Tiempo | Contenido |
|---------|---------|--------|-----------|
| 1 | Persona 1 | 0:00 – 1:45 | Contexto del problema + pipeline del proyecto |
| 2 | Persona 2 | 1:45 – 3:30 | ¿Qué es el HMM? + Los 4 regímenes atmosféricos |
| 3 | Persona 3 | 3:30 – 5:15 | Resultados: predicción y métricas |
| 4 | Persona 4 | 5:15 – 7:00 | Análisis crítico + impacto + cierre |

---

---

## SECCIÓN 1 — CONTEXTO Y PIPELINE DEL PROYECTO
**Persona 1 · Tiempo: 0:00 – 1:45**

> **[SLIDE 1 — Portada]**  
> Título: *Machine Learning para la Calidad del Aire · ZMG*  
> Imagen de fondo: ciudad con smog o mapa de Guadalajara  
> Subtítulo: *Predicción de regímenes atmosféricos con Hidden Markov Model*

---

**[GUION — Persona 1]**

*[Comenzar en cámara, luego pasar a slide]*

"¿Cuándo fue la última vez que revisaste la calidad del aire antes de salir? Probablemente nunca, porque esa información casi nunca llega a tiempo.

En la Zona Metropolitana de Guadalajara, los sensores registran contaminantes cada minuto — CO, NO2, PM2.5, ozono, SO2 y compuestos orgánicos volátiles. Pero tener datos no es lo mismo que entender qué le está pasando a la atmósfera.

Nuestro proyecto busca responder dos preguntas concretas: *¿en qué estado químico se encuentra el aire ahora mismo?* y *¿cuál será su estado en los próximos 30 o 60 minutos?*"

> **[SLIDE 2 — Pipeline del proyecto]**  
> Diagrama de flujo horizontal con 5 fases:  
> ETL → Análisis Exploratorio → Modelado Supervisado → HMM → Predicción

"Para llegar a eso, pasamos por cinco fases. Primero, un proceso ETL donde limpiamos y estandarizamos 13,758 registros de dos sensores en Zapopan — cubriendo del 6 al 25 de noviembre de 2025. Después, un análisis exploratorio que nos mostró que las variables no siguen distribuciones normales, tienen picos extremos, y que la calidad del aire no es un número continuo: tiene *episodios*, tiene *saltos*.

Eso nos llevó a probar modelos supervisados: Regresión Ridge, LightGBM y Random Forest. El mejor de ellos, Random Forest, alcanzó un R² de 0.93 prediciendo el AQI. Pero había un problema: predecir un número no nos dice *por qué* la atmósfera está así, ni cuánto va a durar el episodio.

Por eso elegimos el Hidden Markov Model como modelo final. Y eso es lo que les van a explicar mis compañeros."

> **[Ceder la palabra a Persona 2]**

---

### Notas de producción — Persona 1
- Hablar a cámara en el arranque (primeras 2 oraciones). Rompe con la dinámica de "leer slides".
- En el slide del pipeline, señalar cada fase mientras se menciona.
- Ritmo: pausado al inicio, más fluido al describir el pipeline.
- Si hay tiempo de sobra, agregar: *"Trabajamos con los sensores D_P1 y D_P2, que son los únicos con mediciones completas de todos los contaminantes criterio."*

---

---

## SECCIÓN 2 — EL HMM Y LOS 4 REGÍMENES ATMOSFÉRICOS
**Persona 2 · Tiempo: 1:45 – 3:30**

> **[SLIDE 3 — ¿Qué es el HMM?]**  
> Diagrama simple:  
> — Fila superior: "Estado oculto Z(t)" → burbujas: E1, E2, E3, E4 con flechas entre ellas  
> — Fila inferior: "Observaciones X(t)" → cajitas: CO, NO2, PM2.5, O3, SO2, TVOC  
> — Flechas verticales de cada estado a sus observaciones  
> Leyenda: *"Lo que vemos ≠ lo que está pasando"*

---

**[GUION — Persona 2]**

"La idea central del HMM es que la atmósfera tiene estados que *no podemos ver directamente*. No existe un sensor que diga 'estamos en un episodio de combustión'. Lo que sí medimos son las concentraciones de contaminantes. El HMM infiere el estado oculto a partir de esas señales químicas.

A diferencia del clustering tradicional, que asigna cada instante a un grupo sin importar el tiempo, el HMM incorpora la dinámica: aprende la probabilidad de que el sistema *cambie* de un estado a otro. Eso es lo que nos permite predecir hacia adelante."

> **[SLIDE 4 — Los 4 estados aprendidos]**  
> Tabla visual con los 4 estados:  
> | Estado | Nombre | Firma química | AQI modal |  
> |--------|--------|--------------|-----------|  
> | E1 | Industrial / Diesel | SO2 máximo (0.039 ppm), TVOC=526 ppb | 2 |  
> | E2 | Línea base | Todo moderado-bajo | 2 |  
> | E3 | Combustión intensa | CO=2.48 ppm (5× promedio), TVOC=2,327 ppb, PM2.5=17.1 µg/m³ | 3 |  
> | E4 | Fotoquímico | O3=0.171 ppm máximo, NO2=0.078 ppm | 2 (transitorio) |  
> Usar colores: azul (E1), verde (E2), naranja (E3), rojo (E4)

"El modelo encontró exactamente cuatro regímenes. Sin que nadie se los dijera.

El **Estado 2** es la línea base: aire con niveles moderados, presente el 52.7% del tiempo en nuestro periodo de prueba. El **Estado 3** es el más preocupante — combustión intensa, con CO cinco veces por encima del promedio y PM2.5 sobre el límite de la OMS. Estuvo presente el 28.7% del tiempo.

El **Estado 1** muestra una firma interesante: SO2 alto sin NO2 elevado. Aquí fue clave la colaboración con los estudiantes de Ingeniería Química — ellos nos explicaron que esa combinación específica apunta a combustión de combustóleo o fuente industrial fija, no a tráfico vehicular. El NO2 lo generan los motores a altas temperaturas; el SO2 proviene de combustibles con azufre. Esa distinción no la íbamos a sacar solos de los datos.

Y el **Estado 4** es el episodio fotoquímico — ozono y NO2 altos — pero completamente transitorio. Los compañeros de IQ nos confirmaron que esto es coherente con la química real: el ozono troposférico se forma por reacción entre NO2 y radiación solar, y se consume rápidamente al atardecer o en presencia de CO. Por eso A[4,4]=0.000 — el estado fotoquímico nunca se auto-sostiene. Cada evento dura menos de dos minutos antes de que el sistema regrese a línea base. El modelo capturó ese comportamiento sin que nadie se lo programara."

> **[SLIDE 5 — Matriz de transición]**  
> Heatmap de la matriz A (4×4)  
> Destacar con texto: "E1: persiste 143 min promedio · E3: persiste 25 min · E4: siempre transitoria"

"Y esto nos lleva a algo que ningún modelo de regresión puede dar: la *persistencia*. Un episodio de combustión intensa dura en promedio 25 minutos antes de disiparse. Uno industrial, 143 minutos. Saber eso es la diferencia entre emitir una alerta y emitir una alarma."

> **[Ceder la palabra a Persona 3]**

---

### Notas de producción — Persona 2
- En el slide de los 4 estados, no leer la tabla completa — destacar E3 y E4 con énfasis.
- La frase *"Sin que nadie se los dijera"* es el momento de impacto — pausa breve después.
- Si se acaba el tiempo: omitir el slide de la matriz de transición y solo mencionar la persistencia de E3.

---

---

## SECCIÓN 3 — RESULTADOS: PREDICCIÓN Y MÉTRICAS
**Persona 3 · Tiempo: 3:30 – 5:15**

> **[SLIDE 6 — Por qué K=4 y no K=8]**  
> Dos columnas comparativas:  
> | Métrica | K=8 | K=4 |  
> |---------|-----|-----|  
> | Sobreajuste train-test | SEVERO (diff=18,549) | Sin sobreajuste (diff=-9,086) |  
> | Silhouette score | 0.074 (muy débil) | **0.229** (3× mejor) |  
> | Estados presentes en test | 6/8 (2 fantasmas) | **4/4** |  
> Título del slide: *"El BIC eligió K=8. Nosotros elegimos K=4. Así lo justificamos."*

---

**[GUION — Persona 3]**

"El algoritmo BIC, que usamos para seleccionar el número de estados óptimo, señaló K=8. Nosotros lo analizamos y decidimos forzar K=4. ¿Por qué?

Porque con K=8 el modelo memorizó el conjunto de entrenamiento: dos de los ocho estados nunca aparecieron en los datos de prueba, y el Silhouette score — que mide qué tan bien separados están los clusters — cayó a 0.074, casi en el suelo. El BIC penaliza la complejidad, pero no penaliza la falta de generalización. En series temporales cortas con alta autocorrelación, sobreselecciona K.

Con K=4, en cambio, los cuatro estados aparecen en los datos de prueba, el Silhouette es tres veces mayor, y la diferencia de log-likelihood entre entrenamiento y prueba es *negativa*: el modelo generaliza mejor a datos nuevos que a los que vio. Eso, combinado con el respaldo de la química atmosférica — que establece 4 o 5 regímenes distintos en una zona urbana — hace que K=4 sea la elección correcta."

> **[SLIDE 7 — Predicción multi-step]**  
> Tres gráficas de barras lado a lado (del notebook HMM_05_pred_distribucion.png):  
> — t=0: 100% en E3  
> — t+30 min: 47.7% E2 | 47.7% E3 | 4.5% E1  
> — t+60 min: 65.0% E2 | 28.3% E3 | 6.5% E1  
> Debajo: tabla de concentraciones esperadas  
> | Contaminante | t+30 min | t+60 min |  
> |-------------|---------|---------|  
> | CO (ppm) | 1.38 | 0.98 |  
> | PM2.5 (µg/m³) | 13.6 | 12.3 |  
> | TVOC (ppb) | 1,218 | 809 |

"Ahora la parte más aplicable: la predicción. Arrancamos desde un episodio de combustión intensa — el sistema está 100% en E3, con CO a 2.48 ppm.

A los 30 minutos, la probabilidad se parte al 50-50: hay casi la misma chance de que el episodio persista a que se disipe. El CO esperado baja a 1.38 ppm, y el AQI predicho es 3 — nivel elevado. A los 60 minutos, el 65% del peso probabilístico ya está en línea base. El CO esperado cae por debajo de 1 ppm y el AQI predicho baja a 2 — moderado.

Esto no es un número puntual. Es una *distribución de probabilidad* sobre los posibles futuros del sistema. Esa incertidumbre cuantificada es exactamente lo que se necesita para diseñar alertas de calidad del aire accionables."

> **[SLIDE 8 — Métricas del modelo]**  
> Dos cajas resumen:  
> — Log-likelihood en test: -14,035 (sin sobreajuste)  
> — Silhouette score: 0.229 (separación razonable)  
> + Nota: *"Un modelo no supervisado no tiene 'respuestas correctas'. La validación se hace con métricas internas + consistencia con el AQI del sensor."*

"En términos de métricas formales: el log-likelihood en test es -14,035 y el Silhouette de 0.229. Para un modelo no supervisado con series temporales reales y cuatro estados, esto es una separación razonable. Además, al mapear cada estado aprendido al AQI modal del sensor — una etiqueta independiente que el modelo nunca usó — encontramos consistencia perfecta: E3 corresponde a AQI=3, todos los demás a AQI=2."

> **[Ceder la palabra a Persona 4]**

---

### Notas de producción — Persona 3
- En el slide de K=4 vs K=8, mencionar solo las tres métricas de la tabla. No entrar en detalles del BIC matemático.
- La frase sobre "distribución de probabilidad sobre los posibles futuros" es el punto de cierre de esta sección — decirla despacio.
- Si sobra tiempo: *"Comparado con Random Forest — nuestro mejor modelo supervisado con R²=0.93 — el HMM no predice mejor el AQI puntual, pero predice algo que Random Forest no puede: qué pasa después."*

---

---

## SECCIÓN 4 — ANÁLISIS CRÍTICO, LIMITACIONES E IMPACTO
**Persona 4 · Tiempo: 5:15 – 7:00**

> **[SLIDE 9 — HMM vs Modelos Supervisados]**  
> Tabla comparativa:  
> | Capacidad | Regresión / RF / LightGBM | HMM |  
> |-----------|--------------------------|-----|  
> | Predecir AQI puntual | ✅ (R²=0.93) | ≈ (indirecto) |  
> | Identificar régimen atmosférico | ❌ | ✅ |  
> | Cuantificar incertidumbre | ❌ | ✅ |  
> | Predecir duración del episodio | ❌ | ✅ |  
> | Predicción multi-horizonte | ❌ | ✅ |

---

**[GUION — Persona 4]**

"Pongamos en perspectiva qué ganamos con el HMM sobre los modelos supervisados que probamos primero.

Random Forest predice el AQI con R² de 0.93 — excelente para regresión. Pero nos da un número. No nos dice si ese AQI de 2.8 es porque estamos al final de un episodio de combustión que se va a disipar, o al inicio de uno que va a empeorar. El HMM sí puede hacer esa distinción.

Tres ventajas concretas: primero, identificación de regímenes sin etiquetas — el modelo encontró los cuatro estados químicamente coherentes sin supervisión. Segundo, cuantificación de incertidumbre — la salida es una distribución de probabilidades, no un número puntual. Y tercero, cuantificación de la persistencia de los episodios, que es fundamental para sistemas de alerta tempranas."

> **[SLIDE 10 — Limitaciones honestas]**  
> Lista con íconos:  
> ⚠️ Solo un mes de datos (noviembre 2025) — no captura variación estacional  
> ⚠️ CO y TVOC tienen distribuciones muy asimétricas — las colas extremas son difíciles para HMM Gaussiano  
> ⚠️ A t+60 min la distribución converge al estado estacionario — información limitada para horizontes mayores  
> ⚠️ BIC no confiable en series cortas — requiere combinar con validación temporal

"Y con la misma transparencia, las limitaciones. Tenemos datos de un solo mes y un solo sector de Zapopan. Un modelo de calidad del aire robusto necesitaría datos de por lo menos un año para capturar variaciones estacionales — las quemas agrícolas de primavera no están aquí.

Además, contaminantes como CO y TVOC tienen picos extremos que una distribución Gaussiana no modela bien. Una extensión natural sería usar una mezcla de Gaussianas por estado para capturar esos eventos.

Y a horizontes mayores de una hora, la información del estado actual se diluye. Para predicciones de varias horas se necesitaría incorporar variables meteorológicas externas."

> **[SLIDE 11 — Impacto y cierre]**  
> Imagen: mapa de Guadalajara o ícono de alerta  
> Texto central: *"¿Para qué sirve este modelo en el mundo real?"*  
> Tres bullets:  
> — Alertas automáticas cuando P(E3) > 40% durante más de 10 min  
> — Identificar fuentes industriales vs vehiculares por firma química  
> — Base para un sistema de monitoreo continuo con datos de toda la ZMG  

"Entonces, ¿para qué sirve esto en la práctica? Imaginen un dashboard de monitoreo que, cada minuto, actualiza la probabilidad de estar en un episodio de combustión intensa. Cuando esa probabilidad supera el 40% durante diez minutos consecutivos, se dispara una alerta automática — no cuando ya todos lo respiran, sino antes.

La firma química de cada estado también permite distinguir si el origen es industrial o vehicular sin necesidad de un equipo de monitoreo adicional. Solo con los sensores existentes.

Este proyecto comenzó como un problema de optimización con datos de calidad del aire. Terminó siendo un modelo que describe cómo *cambia* la atmósfera, cuánto duran los problemas, y cuándo es más probable que mejoren.

Pero quiero cerrar hablando de algo que no aparece en ninguna métrica: lo que significó trabajar con estudiantes de Ingeniería Química.

Nosotros llegamos con el modelo. Los estados E1, E2, E3, E4 eran etiquetas numéricas — el algoritmo los encontró, pero no sabía qué eran. Fueron los compañeros de IQ quienes pusieron nombre a esos números. Nos explicaron que SO2 alto sin NO2 es firma de combustóleo, no de gasolina. Que el ozono se forma fotoquímicamente y se consume rápido. Que PM2.5 sobre 15 µg/m³ sostenido es un umbral de alerta real para poblaciones vulnerables.

Sin eso, habríamos entregado cuatro clusters con letras. Con eso, entregamos cuatro regímenes atmosféricos con significado físico verificado. Esa es la diferencia entre hacer machine learning *sobre* un problema y hacerlo *dentro* de él.

La validación más importante no fue el Silhouette de 0.229. Fue que los estudiantes de química revisaron los estados y dijeron: 'sí, eso existe, eso tiene sentido'."

> **[SLIDE FINAL — Agradecimiento / Créditos]**  
> Nombres del equipo  
> Institución · Semestre · Fecha

"Gracias."

---

### Notas de producción — Persona 4
- La comparación de la tabla (slide 9) se puede decir de memoria — no leerla línea a línea.
- En limitaciones, el tono debe ser de *reflexión honesta*, no de disculpa. Un modelo que reconoce sus límites es más confiable que uno que no los menciona.
- El bloque interdisciplinario es el cierre emocional — hablar directo a cámara, sin mirar slides, ritmo más lento.
- La frase *"La validación más importante no fue el Silhouette de 0.229. Fue que los estudiantes de química dijeron: sí, eso existe"* es el remate final antes del "Gracias" — no apresurarse.
- Dejar ~3 segundos de pausa antes de "Gracias."

---

---

## RESUMEN DE SLIDES REQUERIDOS

| # | Título sugerido | Contenido clave | Persona |
|---|----------------|-----------------|---------|
| 1 | Portada | Título + imagen + subtítulo | 1 |
| 2 | Pipeline del proyecto | Diagrama de 5 fases | 1 |
| 3 | ¿Qué es el HMM? | Diagrama estados ocultos → observaciones | 2 |
| 4 | Los 4 estados aprendidos | Tabla con firma química y AQI | 2 |
| 5 | Matriz de transición | Heatmap + persistencia en texto | 2 |
| 6 | K=4 vs K=8 | Tabla comparativa de métricas | 3 |
| 7 | Predicción multi-step | Barras de probabilidad t=0/+30/+60 + concentraciones | 3 |
| 8 | Métricas del modelo | LL test + Silhouette + nota sobre validación | 3 |
| 9 | HMM vs Modelos Supervisados | Tabla comparativa de capacidades | 4 |
| 10 | Limitaciones honestas | Lista con íconos de advertencia | 4 |
| 11 | Impacto y cierre | Imagen + 3 bullets de uso real | 4 |
| 12 | Créditos | Nombres del equipo | 4 |

---

## CHECKLIST ANTES DE GRABAR

- [ ] Cada persona ensayó su sección al menos 2 veces con cronómetro
- [ ] Las transiciones entre personas son fluidas ("y eso es lo que les va a explicar mi compañero…")
- [ ] Las slides tienen las figuras guardadas del notebook (HMM_03, HMM_04, HMM_05, HMM_06, HMM_07 en la carpeta Data/)
- [ ] El video total no supera 7 minutos (grabar un ensayo completo antes de la versión final)
- [ ] Hablar a cámara en los momentos de impacto, no leer los slides

---

## DATOS DE REFERENCIA RÁPIDA (para no buscarlos durante el ensayo)

| Dato | Valor |
|------|-------|
| Total registros | 13,758 |
| Periodo de datos | 6 – 25 nov 2025 |
| Sensores | D_P1 (Zapopan) y D_P2 (Zapopan) |
| Contaminantes | CO, NO2, PM2.5, O3, SO2, TVOC |
| R² Random Forest (mejor supervisado) | 0.9323 |
| K seleccionado | 4 estados |
| Silhouette en test | 0.229 |
| E3 en test | 28.7% del tiempo |
| CO en E3 | 2.48 ppm (5× el promedio) |
| Predicción t+30 desde E3 | CO=1.38 ppm, AQI=3, P(E2)=47.7% |
| Predicción t+60 desde E3 | CO=0.98 ppm, AQI=2, P(E2)=65.0% |
| Persistencia E3 | ~25 minutos promedio |
| Persistencia E1 | ~143 minutos promedio |
