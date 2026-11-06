# Análisis de la señal ECG: Extracción de características y clasificación

**Curso:** Introducción a las Señales Biomédicas — Universidad Peruana Cayetano Heredia

**Equipo de adquisición:** BITalino (r)evolution, sensor ECG

**Derivaciones analizadas:** D1, D2 y D3 (triángulo de Einthoven)

## 1. Objetivos

1. Leer los registros de OpenSignals, convertirlos a milivoltios y verificar su calidad (paquetes perdidos, saturación, polaridad del QRS e intervalos RR anómalos).
2. Filtrar la señal (pasa-banda 0.5–40 Hz y notch de 60 Hz) y detectar los picos R con el algoritmo de Pan-Tompkins.
3. Segmentar la señal en latidos (−200 ms / +400 ms alrededor de R) y construir una tabla de 24 características por latido, agrupadas en cinco familias: ritmo, morfología, estadísticas básicas, estadísticas de orden superior y frecuencia (FFT y wavelet).
4. Comparar cinco clasificadores (Árbol de Decisión, Regresión Logística, Random Forest, KNN y SVM) con una validación que evite la fuga de datos.
5. Comparar el desempeño entre D1, D2 y D3 e identificar qué aporta más a la clasificación: el ritmo cardiaco o la forma del latido.

## 2. Procedimiento (Metodología)

El flujo completo del cuaderno es el siguiente:

```
Lectura → Control de calidad → Filtrado → Pan-Tompkins → Segmentación → Características (24) → Tabla → Clasificación (5 algoritmos)
```

### 2.1 Datos de entrada

Cada derivación tiene 4 archivos `.txt` de OpenSignals, uno por condición, para un total de 12 archivos. Cada archivo dura entre 30 y 37 s.

| Condición | Qué hizo el participante |
|---|---|
| Basal | Reposo con respiración normal |
| Hiperventilación | Ciclos repetidos de inhalar, mantener y exhalar |
| Hipoventilación | Retención de la respiración el mayor tiempo posible |
| Aeróbica | Registro inmediatamente después de unos 3 min de ejercicio |


### 2.2 Configuración y lectura de los registros

- Se usaron `numpy`, `pandas`, `matplotlib`, `scipy.signal`, `PyWavelets` y `scikit-learn`.
- La función `identificar()` deduce la condición y la derivación a partir del nombre del archivo, y `buscar_archivos()` arma el inventario de los 12 archivos.
- La función `leer_ecg()` lee la cabecera JSON del archivo (frecuencia de muestreo, resolución, canal ECG) y convierte las cuentas del ADC a milivoltios con la fórmula del sensor ECG de BITalino:

$$ECG_{mV} = \frac{\left(\frac{ADC}{2^{n}} - \frac{1}{2}\right)\cdot V_{CC}}{G_{ECG}} \times 1000,\quad V_{CC}=3.3\ \text{V},\ G_{ECG}=1100,\ n=10\ \text{bits}$$

- Durante la lectura se cuentan los **paquetes perdidos** (el contador `nSeq` debe avanzar de 1 en 1) y las **muestras saturadas** (ADC en 0 o ≥ 1022).
- La tabla de lectura de los tres notebooks indica una frecuencia de muestreo de **100 Hz** (ver la observación en la sección 4.6).

### 2.3 Filtrado (`filtrar_ecg`)

1. Se resta la media de la señal.
2. Filtro **pasa-banda Butterworth de orden 3, 0.5–40 Hz**, en formato `sos` y aplicado con `sosfiltfilt`. Elimina la deriva de línea base (respiración) y el ruido de alta frecuencia.
3. Filtro **notch de 60 Hz** (`iirnotch`, Q = 30) aplicado con `filtfilt` para la interferencia de la red eléctrica.

Se usa filtrado de fase cero ("ida y vuelta") para no desplazar las ondas en el tiempo y así no alterar las medidas de tiempo.

### 2.4 Detección de picos R (Pan-Tompkins simplificado, `detectar_picos_R`)

| Paso | Operación | Propósito |
|---|---|---|
| 1 | Pasa-banda 5–15 Hz (Butterworth orden 2) | Conservar la energía del QRS y atenuar P y T |
| 2 | Derivada | Resaltar las pendientes empinadas del QRS |
| 3 | Elevar al cuadrado | Hacer todo positivo y agrandar las pendientes grandes |
| 4 | Integración en ventana móvil de 150 ms | Convertir cada QRS en una sola "joroba" |
| 5 | Umbral (0.3 × percentil 99) y periodo refractario de 250 ms | Decidir qué jorobas son latidos |
| 6 | Máximo del ECG filtrado a ±100 ms de cada joroba | Ubicar el pico R |

### 2.5 Control de calidad (`preparar_registro`)

- **Polaridad:** si la asimetría (*skewness*) de la señal filtrada es negativa, el QRS apunta hacia abajo y la señal se invierte (`INVERTIR = "auto"`).
- **Intervalos RR anómalos** (`rr_anomalos`): un RR se marca como anómalo si se aleja más de 35 % de la mediana móvil de 9 valores.
- **Latidos válidos** (`indices_validos`): se descartan el primer y el último latido, los que no tienen ventana completa y los que tienen un RR anómalo a cualquiera de sus lados.
- **Semáforo:** ✅ sin saturación y sin RR anómalos; ⚠️ menos de 10 % de RR anómalos; ❌ saturación o 10 % o más de RR anómalos.

### 2.6 Frecuencia cardiaca

Se calcula como FC = 60 / RR usando solo RR no anómalos: FC media, FC al inicio (primeros 5 latidos) y FC al final (últimos 5 latidos). También se grafica el tacograma.

### 2.7 Segmentación en latidos (`segmentar`)

Se recorta una ventana de **−200 ms a +400 ms** alrededor de cada pico R válido. Con la frecuencia de muestreo del archivo (100 Hz) cada latido tiene **60 muestras**. Cada latido es una fila de la tabla de datos.

### 2.8 Extracción de características (24 por latido)

| Familia | Características |
|---|---|
| **Ritmo (3)** | `RR_prev_ms`, `RR_next_ms`, `FC_lpm` |
| **Morfológicas (7)** | `amp_P`, `amp_Q`, `amp_R`, `amp_S`, `amp_T`, `dur_QS_ms`, `t_RT_ms` |
| **Estadísticas básicas (5)** | `media`, `mediana`, `desv_est`, `p25`, `p75` |
| **Orden superior, HOS (2)** | `asimetria` (skewness), `curtosis` |
| **Frecuencia (7)** | FFT: `f_dominante_Hz`, `potencia_0.5_40`, `frac_5_15Hz`. Wavelet db4 de 6 niveles (energía relativa): `wav_E_cA6`, `wav_E_cD6`, `wav_E_cD5`, `wav_E_cD4` |

Detalles de cada familia:

- **Morfología:** las amplitudes se miden respecto a la línea isoeléctrica, estimada como el promedio entre −100 y −60 ms respecto a R. Ventanas de búsqueda: P, máximo entre −200 y −80 ms; Q, mínimo entre −60 y 0 ms; R, valor en el pico; S, mínimo entre 0 y +80 ms; T, máximo entre +150 y +400 ms. `dur_QS_ms` es el tiempo entre Q y S, y `t_RT_ms` el tiempo entre R y T.
- **FFT:** se aplica ventana de Hann y relleno a `nfft = 1024`. La frecuencia dominante y la potencia se calculan en la banda 0.5–40 Hz, y `frac_5_15Hz` es la fracción de esa potencia en 5–15 Hz (banda del QRS).
- **Wavelet:** se descompone cada latido con la wavelet Daubechies 4 (db4) a 6 niveles y se calcula el porcentaje de energía en cA6, cD6, cD5 y cD4.

### 2.9 Tabla de características (`construir_tabla`)

Se juntan las cinco familias para todos los latidos. El resultado es una tabla de **N latidos × 24 características**, más las columnas `clase` (condición), `archivo` y `t_R`. También se genera el archivo `caracteristicas_ecg.csv`.

### 2.10 Clasificación y validación

**Clasificadores** (cada uno dentro de un *pipeline* con `StandardScaler`):

| Algoritmo | Configuración |
|---|---|
| Árbol de Decisión (DT) | `max_depth = 5` |
| Regresión Logística (LR) | `max_iter = 3000` |
| Random Forest (RF) | 200 árboles |
| KNN | k = 5 |
| SVM | kernel RBF |

**Validación:** como los latidos consecutivos son casi idénticos, mezclar latidos al azar produce **fuga de datos**. Por eso se agrupan los latidos en **bloques de 10 s** y se usa `StratifiedGroupKFold`, de modo que cada bloque va completo a entrenamiento o a prueba. Si hay muy pocos bloques por condición, el cuaderno cae a `StratifiedKFold` simple, lo que ocurrió en D1. El número de pliegues es el mínimo entre 5 y el menor número de bloques por clase.

**Métricas:** exactitud (*accuracy*) y F1 macro, ambas con su desviación estándar entre pliegues. El azar equivale a 0.25 (4 clases).

**Experimentos adicionales:**

1. Comparación con **todas** las características, con **solo ritmo** (3 características) y **sin ritmo** (21 características de forma).
2. Comparación entre validación por bloques y validación con latidos mezclados al azar (con fuga), usando Random Forest solo con forma.
3. Matriz de confusión del mejor modelo e importancia de características de Random Forest.


---

## 3. Resultados

### 3.1 Archivos Python

* [Derivada 1 (DI)](https://github.com/Itzmiyeko/GRUPO9-ISB-2026-II/blob/main/Laboratorios/Lab_10/Extracci%C3%B3n%20de%20caracter%C3%ADsticas%20-%20Python/LAB11_ECG_Extraccion_Caracteristicas_D1.ipynb)
* [Derivada 2 (DII)](https://github.com/Itzmiyeko/GRUPO9-ISB-2026-II/blob/main/Laboratorios/Lab_10/Extracci%C3%B3n%20de%20caracter%C3%ADsticas%20-%20Python/LAB11_ECG_Extraccion_Caracteristicas_D2.ipynb)
* [Derivada 3 (DIII)](https://github.com/Itzmiyeko/GRUPO9-ISB-2026-II/blob/main/Laboratorios/Lab_10/Extracci%C3%B3n%20de%20caracter%C3%ADsticas%20-%20Python/LAB11_ECG_Extraccion_Caracteristicas_D3.ipynb) 

[Las respuestas del cuestionario del Colab del Laboratorio se realizo en otro archivo aparte:] (https://github.com/Itzmiyeko/GRUPO9-ISB-2026-II/blob/main/Laboratorios/Lab_10/Extracci%C3%B3n%20de%20caracter%C3%ADsticas%20-%20Python/LAB11_ECG_Extraccion_Caracteristicas_Preguntas_respondidas%20(1).ipynb)

### 3.2 Calidad de los registros (comunes a los tres notebooks)

| Condición | Derivación | QRS original | Latidos detectados | Latidos válidos | % RR anómalos | Muestras saturadas | Estado |
|---|---|---|---:|---:|---:|---:|:---:|
| Basal | D1 | positivo | 59 | 57 | 0.0 | 0 | ✅ |
| Hiperventilación | D1 | positivo | 54 | 35 | 18.9 | 0 | ❌ |
| Hipoventilación | D1 | positivo | **6** | **2** | 40.0 | 0 | ❌ |
| Aeróbica | D1 | negativo → invertido | 70 | 68 | 0.0 | 0 | ✅ |
| Basal | D2 | positivo | 56 | 54 | 0.0 | 0 | ✅ |
| Hiperventilación | D2 | positivo | 79 | 76 | 1.3 | 0 | ⚠️ |
| Hipoventilación | D2 | positivo | 64 | 62 | 0.0 | 0 | ✅ |
| Aeróbica | D2 | positivo | 75 | 73 | 0.0 | 1 | ❌ |
| Basal | D3 | positivo | 56 | 54 | 0.0 | 0 | ✅ |
| Hiperventilación | D3 | positivo | 66 | 64 | 0.0 | 0 | ✅ |
| Hipoventilación | D3 | positivo | 53 | 51 | 0.0 | 0 | ✅ |
| Aeróbica | D3 | positivo | 71 | 69 | 0.0 | 2 | ❌ |

No hubo paquetes perdidos en ningún archivo. Las duraciones fueron de 30.4 s a 31.1 s, salvo D2 hiperventilación (36.6 s) y D2 hipoventilación (35.2 s).

### 3.3 Frecuencia cardiaca media (lpm)

| Condición | D1 | D2 | D3 |
|---|---:|---:|---:|
| Basal | 116.1 | 109.2 | 110.4 |
| Hiperventilación | 125.2 | 130.0 | 131.2 |
| Hipoventilación | 209.3 (artefacto) | 108.8 | 104.0 |
| Aeróbica | 138.4 | 147.5 | 140.5 |

**Figura 1.** *FC media por condición y derivación, y tacograma de D1.*

<img width="1487" height="406" alt="Fig1_10" src="https://github.com/user-attachments/assets/0b8491a0-54e0-4bab-917e-3a291ab262f1" />

### 3.4 Latidos segmentados

| Derivación | Matriz de latidos | Basal | Hiperventilación | Hipoventilación | Aeróbica |
|---|---|---:|---:|---:|---:|
| D1 | 162 × 60 | 57 | 35 | **2** | 68 |
| D2 | 265 × 60 | 54 | 76 | 62 | 73 |
| D3 | 238 × 60 | 54 | 64 | 51 | 69 |

**Figura 2.** *Latidos segmentados y latido promedio por condición (D1, D2 y D3).*

<img width="1949" height="1215" alt="Fig2_l10" src="https://github.com/user-attachments/assets/2e10dade-f43b-4fb6-85f5-172547beda0b" />

**Figura 3.** *Latido promedio de cada condición superpuesto.*

<img width="1044" height="1215" alt="Fig3_l10" src="https://github.com/user-attachments/assets/b0e96c96-0965-4d4e-97fe-12358a59c95b" />

### 3.5 Resultados de la extracción de características

Cada derivación produjo una tabla de **24 características por latido** (162, 265 y 238 latidos en D1, D2 y D3). Promedio de algunas características representativas por condición:


**Figura 4.** *Características de ECG de D1*

<img width="455" height="667" alt="Fig4_l10" src="https://github.com/user-attachments/assets/5902ac3e-1720-40e8-9b8f-ff772aded65e" />

**Figura 5.** *Características de ECG de D2*

<img width="460" height="672" alt="Fig5_l10" src="https://github.com/user-attachments/assets/86f8e194-bd1a-4743-af14-adf5d32e82f0" />

**Figura 6.** *Características de ECG de D3*

<img width="452" height="672" alt="Fig6_l10" src="https://github.com/user-attachments/assets/6fa56aef-f9a2-4799-84b3-60833ba8ff74" />

### 3.6 Clasificación: comparación de 5 algoritmos

Los valores son media ± desviación estándar entre pliegues.

#### D1 — 162 latidos, 2 pliegues, validación estratificada simple (StratifiedKFold)

| Modelo | Exactitud | F1 macro |
|---|---:|---:|
| Árbol de Decisión | 0.969 ± 0.006 | **0.972 ± 0.005** |
| Regresión Logística | **0.988 ± 0.012** | 0.866 ± 0.134 |
| Random Forest | 0.981 ± 0.006 | 0.737 ± 0.006 |
| KNN (k = 5) | 0.951 ± 0.012 | 0.707 ± 0.013 |
| SVM (RBF) | 0.981 ± 0.006 | 0.741 ± 0.006 |

#### D2 — 265 latidos, 14 bloques de 10 s, 3 pliegues (StratifiedGroupKFold)

| Modelo | Exactitud | F1 macro |
|---|---:|---:|
| Árbol de Decisión | 0.673 ± 0.029 | 0.607 ± 0.079 |
| Regresión Logística | **0.720 ± 0.041** | **0.661 ± 0.088** |
| Random Forest | 0.630 ± 0.097 | 0.563 ± 0.135 |
| KNN (k = 5) | 0.666 ± 0.045 | 0.620 ± 0.069 |
| SVM (RBF) | 0.667 ± 0.063 | 0.607 ± 0.083 |

#### D3 — 238 latidos, 12 bloques de 10 s, 3 pliegues (StratifiedGroupKFold)

| Modelo | Exactitud | F1 macro |
|---|---:|---:|
| Árbol de Decisión | 0.596 ± 0.183 | 0.466 ± 0.122 |
| Regresión Logística | 0.748 ± 0.199 | 0.580 ± 0.061 |
| Random Forest | 0.740 ± 0.172 | 0.631 ± 0.210 |
| KNN (k = 5) | 0.705 ± 0.189 | 0.486 ± 0.055 |
| SVM (RBF) | **0.762 ± 0.178** | **0.699 ± 0.197** |

#### Resumen del mejor modelo según F1 macro

| Derivación | Mejor modelo | F1 macro | Observación |
|---|---|---:|---|
| D1 | Árbol de Decisión | 0.972 | Valor no confiable (ver sección 4.2) |
| D2 | Regresión Logística | 0.661 | Todos los modelos quedan dentro de una desviación estándar |
| D3 | SVM (RBF) | 0.699 | Variación entre pliegues muy alta (± 0.197) |


### 3.7 Matrices de confusión del mejor modelo (filas: clase real; columnas: clase predicha)

**D1 — Árbol de Decisión**

| Real \ Predicha | Basal | Hiperv. | Hipov. | Aeróbica |
|---|---:|---:|---:|---:|
| Basal | 55 | 2 | 0 | 0 |
| Hiperventilación | 1 | 33 | 0 | 1 |
| Hipoventilación | 0 | 0 | 2 | 0 |
| Aeróbica | 0 | 1 | 0 | 67 |

**D2 — Regresión Logística**

| Real \ Predicha | Basal | Hiperv. | Hipov. | Aeróbica |
|---|---:|---:|---:|---:|
| Basal | 34 | 0 | 20 | 0 |
| Hiperventilación | 7 | 46 | 1 | 22 |
| Hipoventilación | 10 | 6 | 46 | 0 |
| Aeróbica | 0 | 8 | 0 | 65 |

**D3 — SVM (RBF)**

| Real \ Predicha | Basal | Hiperv. | Hipov. | Aeróbica |
|---|---:|---:|---:|---:|
| Basal | 47 | 1 | 6 | 0 |
| Hiperventilación | 1 | 61 | 2 | 0 |
| Hipoventilación | **36** | 4 | 11 | 0 |
| Aeróbica | 0 | 2 | 0 | 67 |

### 3.8 Importancia de características (Random Forest, 5 más importantes)

| Derivación | Top 5 |
|---|---|
| D1 | `curtosis`, `amp_P`, `amp_S`, `amp_R`, `asimetria` |
| D2 | `desv_est`, `FC_lpm`, `RR_prev_ms`, `RR_next_ms`, `amp_T` |
| D3 | `desv_est`, `RR_prev_ms`, `FC_lpm`, `p25`, `potencia_0.5_40` |

**Figura 7.** *Matriz de confusión del mejor modelo e importancia de características — D1*

<img width="1519" height="537" alt="Fig7_l10" src="https://github.com/user-attachments/assets/83f68736-547a-45fa-9696-3d5afdb7510f" />

**Figura 8.** *Matriz de confusión del mejor modelo e importancia de características — D2*

<img width="1519" height="537" alt="fig8" src="https://github.com/user-attachments/assets/8923a8f0-9ebd-4559-9f29-46e3b72da857" />

**Figura 9.** *Matriz de confusión del mejor modelo e importancia de características — D3.*

<img width="1523" height="537" alt="fig9" src="https://github.com/user-attachments/assets/5a8c5186-6fa3-4f34-9cc4-8ba2df2fb333" />

### 3.9 ¿De dónde viene la información? (F1 macro)

| Modelo | D1 todas | D1 solo ritmo | D1 sin ritmo | D2 todas | D2 solo ritmo | D2 sin ritmo | D3 todas | D3 solo ritmo | D3 sin ritmo |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Árbol de Decisión | 0.972 | 0.896 | 0.712 | 0.607 | 0.366 | 0.719 | 0.466 | 0.553 | 0.380 |
| Regresión Logística | 0.866 | 0.644 | 0.743 | 0.661 | 0.341 | 0.660 | 0.580 | 0.570 | 0.451 |
| Random Forest | 0.737 | 0.889 | 0.738 | 0.563 | 0.413 | 0.692 | 0.631 | 0.528 | 0.591 |
| KNN (k = 5) | 0.707 | 0.604 | 0.701 | 0.620 | 0.472 | 0.601 | 0.486 | 0.491 | 0.456 |
| SVM (RBF) | 0.741 | 0.548 | 0.739 | 0.607 | 0.346 | 0.648 | 0.699 | 0.543 | 0.639 |


### 3.10 Efecto de la fuga de datos (Random Forest, solo forma, F1 macro)

| Derivación | Latidos mezclados al azar (con fuga) | Validación por bloques (correcta) |
|---|---:|---:|
| D1 | 0.738 | 0.738 (mismo esquema, ver sección 4.2) |
| D2 | 0.897 | 0.692 |
| D3 | 0.890 | 0.591 |

---

## 4. Explicación de los resultados

### 4.1 Calidad de los datos y frecuencia cardiaca

- **D2 y D3 son las derivaciones más limpias.** Tienen las cuatro condiciones con entre 51 y 76 latidos válidos y casi sin RR anómalos. Las marcas ❌ en la condición aeróbica de D2 (1 muestra saturada) y de D3 (2 muestras saturadas) se deben solo a que el criterio del semáforo penaliza cualquier saturación; el porcentaje de RR anómalos es 0 %.
- **D1 tiene dos registros problemáticos.** En hiperventilación, 18.9 % de los RR son anómalos y quedan solo 35 latidos válidos (RR entre 0.26 s y 1.55 s). En hipoventilación solo se detectaron 6 picos R (RR máximo de 3.74 s), de los cuales únicamente 2 latidos resultaron válidos. La FC de 209.3 lpm de esa condición no es fisiológica en reposo: es un artefacto de la detección, no un dato del participante.
- **Polaridad en D1.** Solo el archivo aeróbico de D1 tiene QRS negativo y fue invertido, mientras que los otros tres no. Con esto, las medidas nombradas "P, Q, R, S, T" en ese archivo no necesariamente miden la misma onda que en los demás. Esto puede explicar por qué en D1 `amp_R` baja a 0.125 mV y `amp_P` a 0.009 mV en la condición aeróbica, frente a 0.25–0.27 mV y 0.07–0.14 mV en las otras tres.
- **La FC sigue el orden fisiológico esperado en D2 y D3:** aeróbica (140–148 lpm) > hiperventilación (130–131 lpm) > basal (109–110 lpm) ≈ hipoventilación (104–109 lpm). El ejercicio aumenta la demanda de oxígeno y la actividad simpática, y la hiperventilación eleva la FC por la arritmia sinusal respiratoria y la activación simpática. La retención de la respiración no cambia mucho la FC en los 30 s registrados.
- **La FC basal es alta (104–116 lpm)** para un registro en reposo. Esto sugiere que el participante estaba activado o ansioso durante la toma, o que el reposo previo fue corto. Además, las derivaciones se grabaron en momentos distintos, por lo que no son comparables 1 a 1.
- **En la condición aeróbica la FC no es igual entre derivaciones** (138.4, 147.5 y 140.5 lpm). Como el ejercicio se hizo antes de cada registro y las horas de grabación son distintas (15:52 a 15:56), la FC iba recuperándose entre una derivación y otra. En D2 la FC pasó de 153.8 lpm al inicio a 142.2 lpm al final, lo que confirma esa recuperación durante el propio registro.

### 4.2 Resultados de clasificación en D1: por qué no son confiables

En D1 todos los modelos parecen excelentes por exactitud (0.95–0.99), pero esto se debe a las siguientes razones:

1. La condición hipoventilación tiene **solo 2 latidos** de un total de 162. Basta con acertar las otras tres clases para obtener una exactitud de 0.98.
2. Esa misma clase explica la brecha entre exactitud y F1 macro: Random Forest, KNN y SVM tienen exactitud 0.95–0.98 pero F1 macro de 0.71–0.74, lo que es compatible con que **no aciertan casi nunca la clase de 2 latidos**. El F1 de la clase pequeña pesa un cuarto del F1 macro.
3. Con tan pocos latidos el cuaderno no pudo usar `StratifiedGroupKFold` y cayó a `StratifiedKFold` simple con 2 pliegues. En ese esquema los latidos vecinos pueden quedar en entrenamiento y en prueba a la vez (fuga de datos). Por eso, en D1 el valor "con fuga" y el valor "por bloques" del experimento de la sección 3.9 son idénticos (0.738): ambos usan el mismo esquema.
4. El F1 de 0.972 del Árbol de Decisión sale de una matriz de confusión casi perfecta, incluida la clase de 2 latidos, que puede deberse a que el árbol memoriza esos pocos latidos.

Por todo esto, **el ranking de D1 no debe interpretarse como evidencia de que D1 sea la mejor derivación**. Es la derivación con peor calidad de datos.

### 4.3 Resultados de clasificación en D2 y D3

- **D2 y D3 usan validación por bloques, que es la correcta.** Con ella, el rendimiento cae a un F1 macro de 0.47–0.70, claramente por encima del azar (0.25) pero lejos del 98 % que reporta el artículo base.
- **Ningún modelo se impone con claridad.** En D2 el mejor (Regresión Logística, 0.661) y el peor (Random Forest, 0.563) difieren en 0.098, lo cual es similar a la desviación estándar entre pliegues (0.07–0.14). En D3 el SVM (0.699 ± 0.197) supera al Random Forest (0.631 ± 0.210) por menos que esa variación. Con 3 pliegues y 12–14 bloques de 10 s, las diferencias entre algoritmos **no son estadísticamente concluyentes**.
- **El mejor modelo cambia entre derivaciones** (Regresión Logística en D2, SVM en D3). Esto indica que el resultado depende más de los datos de cada derivación que de una superioridad intrínseca de un algoritmo. **No se reproduce la superioridad de Random Forest del artículo base.**
- **D3 tiene mayor variabilidad** (± 0.12 a ± 0.21 en F1), así que su puntaje depende mucho de qué bloques caen en prueba.
- **En D2**, la Regresión Logística confunde basal con hipoventilación (20 de 54 basales clasificados como hipoventilación y 10 de 62 hipoventilaciones como basal). También confunde hiperventilación con aeróbica (22 de 76). Esto tiene sentido fisiológico: **basal e hipoventilación tienen casi la misma FC** (109.4 y 108.9 lpm) y las demás medidas son similares, y **hiperventilación y aeróbica** son las dos condiciones con FC elevada.
- **En D3**, el SVM clasifica 36 de 51 latidos de hipoventilación como basal (solo acierta 11). Es la confusión más fuerte de todas las matrices, y coincide con que basal e hipoventilación son casi idénticas en D3 (FC de 110.3 y 104.2 lpm, `amp_R`, `amp_T`, `desv_est` y `frac_5_15Hz` prácticamente iguales). La clase aeróbica es la mejor reconocida (67 de 69).
- **Conclusión de las matrices:** las condiciones con un cambio fisiológico claro (aeróbica) se clasifican bien; las que se parecen al reposo (hipoventilación) son las más difíciles.

### 4.4 Ritmo o forma del latido

- **En D1** el resultado es poco informativo por los problemas mencionados en la sección 4.2 (por ejemplo, el Árbol de Decisión llega a 0.896 solo con ritmo).
- **En D2**, quitar el ritmo no empeora el desempeño (F1 sin ritmo de 0.66–0.72 para DT, LR y RF, similar o mejor que con todas las características) mientras que **con solo ritmo cae a 0.34–0.47**, apenas por encima del azar. Esto indica que la **forma** del latido aporta más información que el ritmo en D2, algo coherente con que basal e hipoventilación tienen el mismo ritmo.
- **En D3**, el SVM mantiene 0.639 sin ritmo contra 0.543 con solo ritmo, y el Random Forest 0.591 contra 0.528. La forma aporta algo más que el ritmo, pero la diferencia es pequeña frente a la variabilidad.
- Aun así, la importancia de características de Random Forest en D2 y D3 incluye `FC_lpm`, `RR_prev_ms` y `RR_next_ms` entre las más altas, junto con `desv_est`. Esto indica que el ritmo y la energía global del latido (`desv_est`) son las variables de las que más depende el modelo.

---

## 5. Conclusiones

1. Se implementó el flujo completo de procesamiento del ECG (lectura, filtrado, Pan-Tompkins, segmentación, 24 características por latido y clasificación) en las tres derivaciones. Se obtuvieron 162 latidos en D1, 265 en D2 y 238 en D3.
2. **D2 y D3 son las derivaciones con mejor calidad de datos**; D1 presenta dos registros problemáticos (hiperventilación con 18.9 % de RR anómalos e hipoventilación con solo 2 latidos válidos) y un archivo con QRS invertido. Por eso los resultados de D1 (F1 de hasta 0.972) **no son confiables** y no deben compararse directamente con D2 y D3.
3. Con la validación por bloques de 10 s, el desempeño en D2 y D3 es **moderado** (F1 macro de 0.47–0.70), por encima del azar (0.25) pero lejos del 98 % del artículo base. En D2 el mejor modelo fue Regresión Logística (F1 = 0.661) y en D3, SVM con kernel RBF (F1 = 0.699), aunque las diferencias entre algoritmos son menores que la variación entre pliegues. **No se reproduce la superioridad de Random Forest.**
4. La **FC** siguió el patrón fisiológico esperado (aeróbica > hiperventilación > basal ≈ hipoventilación) y resultó ser una de las variables más importantes. La clase aeróbica fue la mejor clasificada, y la mayor confusión fue entre **basal e hipoventilación**, que son fisiológicamente similares en 30 s de registro.
5. **La fuga de datos infla el desempeño** (F1 de 0.89–0.90 con latidos mezclados frente a 0.59–0.69 con bloques), así que la validación por bloques es indispensable para no sobreestimar el modelo.
6. Quitar el ritmo no degradó el desempeño en D2 y lo redujo poco en D3, por lo que la **forma del latido** aporta información útil además de la FC, aunque con estos datos esa conclusión es tentativa.
7. **Limitaciones y mejoras propuestas:**
   - Verificar la frecuencia de muestreo real de los archivos (100 Hz frente a los 1000 Hz que asume el cuaderno), ya que afecta al notch de 60 Hz, a las bandas wavelet y a la resolución de las medidas morfológicas.
   - Repetir el registro de hipoventilación y de hiperventilación en D1.
   - Registrar más participantes y más tiempo por condición, y grabar las derivaciones de forma comparable.
   - Considerar la evaluación por participante o por sesión para medir la generalización real del modelo.

---

## Referencias

1. T. Subba y T. Chingtham, "Comparative Analysis of Machine Learning Algorithms With Advanced Feature Extraction for ECG Signal Classification", *IEEE Access*, 2024. doi: 10.1109/ACCESS.2024.3387041
2. J. Pan y W. J. Tompkins, "A Real-Time QRS Detection Algorithm", *IEEE Trans. Biomed. Eng.*, vol. BME-32, no. 3, pp. 230–236, 1985.
3. PLUX Wireless Biosignals, *BITalino (r)evolution — ECG Sensor Datasheet*.
4. L. Sörnmo y P. Laguna, *Bioelectrical Signal Processing in Cardiac and Neurological Applications*, Elsevier, 2005.
