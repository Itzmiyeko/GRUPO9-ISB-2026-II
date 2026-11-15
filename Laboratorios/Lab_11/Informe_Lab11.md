 Informe de Laboratorio 11: Tratamiento de la señal EEG, ritmos cerebrales, filtro ICA y extracción de características

**Universidad Peruana Cayetano Heredia — Introducción a las Señales Biomédicas**

## 1. Objetivos

### 1.1 Objetivo general

Limpiar y analizar señales EEG registradas con BITalino, estudiar sus ritmos cerebrales y extraer características que permitan distinguir la condición del participante mediante algoritmos de clasificación.

### 1.2 Objetivos específicos

1. Revisar la calidad de los registros: conversión a µV, saturación del conversor, paquetes perdidos, interferencias espectrales y curtosis.
2. Adaptar la lectura de registros de 1000 Hz y varios canales a la frecuencia de referencia del laboratorio (100 Hz), eliminando antes la red eléctrica de 60 Hz.
3. Aplicar un filtrado pasa-banda y calcular la potencia relativa de las bandas delta, theta, alfa, beta y gamma.
4. Aplicar el filtro ICA a un solo canal mediante Wavelet-ICA y evaluar qué componentes corresponden a artefactos.
5. Segmentar la señal en épocas de 2 s y extraer 22 características por época (potencia por banda, cocientes, espectrales, Hjorth y estadísticas).
6. Comparar cinco algoritmos de clasificación (Árbol de Decisión, Regresión Logística, Random Forest, KNN y SVM) con una validación que evite la fuga de información entre entrenamiento y prueba.

---

## 2. Metodología

El cuaderno sigue el esquema: señal fisiológica → filtrado → limpieza (ICA) → preprocesamiento (normalización y épocas) → extracción de características → ingeniería de características → clasificación. Las grabaciones se adquirieron a **1000 Hz con 3 canales**, por lo que se incluyó una etapa de selección de canal y remuestreo a 100 Hz antes del análisis.

### 2.1 Configuración

- Librerías: `numpy`, `pandas`, `matplotlib`, `scipy.signal` (filtros y método de Welch), `pywt` (PyWavelets 1.8.0) y `scikit-learn` (FastICA, clasificadores y validación).
- Bandas definidas: delta (1–4 Hz), theta (4–8 Hz), alfa (8–13 Hz), beta (13–30 Hz) y gamma (30–45 Hz).
- Parámetros de lectura: `CANAL_EEG = "A2"` (se usa el canal A2, porque el canal A6 presentaba saturación) y `FS_OBJETIVO = 100` Hz.

### 2.2 Carga e inventario de archivos

1. Se buscan los `.txt` de OpenSignals y se lee la cabecera JSON de cada uno (sensor, columnas, resolución, frecuencia de muestreo, dispositivo y hora).
2. Se conservan los archivos cuyo sensor es EEG y se asigna la condición según el nombre. Se actualizaron las condiciones: `musica_tranquila` pasa a `musica_relajante`, `musica_hard` a `musica_estresante`, y se añaden `abre_cierra` y `preguntas_susurradas`.
3. Si hubiera menos de dos condiciones, el cuaderno entraría en modo demostración con datos simulados. En este caso se trabajó con **datos reales** del grupo.

### 2.3 Lectura, conversión a µV, remuestreo y saturación

La función `leer_eeg` convierte las cuentas del ADC a microvoltios:

$$EEG_{\mu V} = \frac{\left(\frac{ADC}{2^{n}} - \frac{1}{2}\right) \cdot V_{CC}}{G_{EEG}} \times 10^{6}, \quad V_{CC}=3.3\ V,\ G_{EEG}=41\,782,\ n=10$$

Con esta ganancia el rango medible es de ±39.5 µV y la resolución de 0.077 µV por cuenta. Además:

1. Se marcan como **saturadas** las muestras con ADC ≤ 0 o ≥ 2ⁿ − 6.
2. Se cuentan los **paquetes perdidos** a partir del contador `nSeq` (módulo 16).
3. **Remuestreo de 1000 Hz a 100 Hz:** primero se aplica un notch de 60 Hz (Q = 30) a 1000 Hz, donde la red eléctrica es real; luego un filtro Butterworth pasa-bajas de orden 8 y 45 Hz como anti-aliasing; y finalmente se diezma por un factor de 10. Un bloque de 10 muestras se considera saturado si lo está alguna de ellas.

### 2.4 Inspección en el tiempo y en la frecuencia

1. Se grafican 10 s de cada registro sin filtrar (desde el segundo 10, restando la media), con líneas punteadas en ±39.5 µV.
2. Se estima la PSD con el **método de Welch** (`nperseg = 400`).
3. Se mide cuánto sobresale el pico de **40 Hz** respecto de sus vecinos.
4. Se detectan **líneas angostas** de interferencia: picos más de 100 veces sobre su entorno que concentren al menos 4 % de la potencia de 1–45 Hz (se ignoran los primeros 4 s).

### 2.5 Control de calidad

Cada registro recibe un estado: ✅ bueno (sin saturación, sin pico de 40 Hz ≥ 10 veces ni líneas angostas), ⚠️ revisar (saturación menor a 1 %, pico de 40 Hz o líneas angostas) o ❌ problemático (saturación ≥ 1 %). Se calculan además la desviación estándar y la curtosis.

### 2.6 Decisiones de procesamiento

| Parámetro | Valor | Motivo |
|---|---|---|
| Recorte inicial | 4 s | El amplificador tarda unos segundos en estabilizarse |
| Notch | `None` | Los 60 Hz ya se quitaron a 1000 Hz, antes de diezmar, así que no hay alias en 40 Hz |
| Quitar líneas angostas | Sí | Notch angosto en cada línea detectada, solo en su propio archivo |
| Rechazo de épocas | Toda época con al menos una muestra saturada | La saturación destruye información que ningún filtro recupera |

### 2.7 Filtrado

1. **Pasa-banda Butterworth** de 1 a 45 Hz, orden 4 (`sosfiltfilt`).
2. **Notch angosto** (Q = 30) en cada línea de interferencia detectada.
3. Todos los filtros se aplican "ida y vuelta" (fase cero) para no desfasar la señal.

### 2.8 Análisis de ritmos cerebrales

1. Se filtra el registro basal en cada banda y se grafica en el tiempo.
2. Se integra la PSD de Welch dentro de cada banda para obtener la **potencia relativa** (% del total de 1 a 45 Hz), usando solo las muestras no saturadas.

### 2.9 Filtro ICA

1. **Demostración simulada:** se mezclan 3 fuentes conocidas (alfa de 10 Hz, parpadeos y ráfagas musculares) en 3 "electrodos" y se recuperan con `FastICA`. Las componentes se identifican por la curtosis y por la fracción de potencia sobre 20 Hz.
2. **Wavelet-ICA para un canal:**
   - Se descompone la señal con una transformada wavelet estacionaria (`sym4`, 5 niveles) en **6 sub-bandas** que suman la señal original; se tratan como 6 "canales virtuales".
   - Se aplica `FastICA` con 6 componentes.
   - Si más del 40 % de la potencia de una componente está en 38–42 Hz, se **elimina** (interferencia de red).
   - Si su curtosis es mayor que 6, se **corrige**: se anulan solo los coeficientes wavelet (`db4`, 4 niveles) mayores que 4 desviaciones robustas.
   - Las demás se **conservan**. Se reconstruye la señal y se resta el artefacto estimado dentro de la banda de 1 a 45 Hz.
3. El paso a paso se muestra en `abre_cierra`, el registro con artefactos de este grupo, y luego se aplica a todos los registros.

### 2.10 Normalización y épocas

- La normalización z-score es opcional (`NORMALIZAR = False`, no se aplicó en la tabla principal).
- Cada registro se corta en **épocas de 2 s sin solapamiento** (200 muestras, resolución de 0.5 Hz) y se rechaza toda época con muestras saturadas.

### 2.11 Extracción de características (22 por época)

| Familia | Características |
|---|---|
| Potencia por banda | `abs_*` (log₁₀ µV²) y `rel_*` para delta, theta, alfa, beta y gamma (10) |
| Cocientes (ingeniería de características) | `theta_beta`, `alfa_beta` (2) |
| Espectrales | `f_pico`, `sef90`, `entropia` (3) |
| Hjorth | `hj_actividad`, `hj_movilidad`, `hj_complejidad` (3) |
| Estadísticas | `desv_est`, `asimetria`, `curtosis`, `pico_a_pico` (4) |

### 2.12 Clasificación y experimentos

1. Cada época es una fila de la tabla; las columnas `clase`, `archivo`, `dispositivo` y `t` solo identifican la época. La tabla se exporta a `caracteristicas_eeg.csv` (178 filas × 26 columnas).
2. Se comparan cinco modelos dentro de un *pipeline* con `StandardScaler`: Árbol de Decisión (profundidad máxima 5), Regresión Logística, Random Forest (200 árboles), KNN (k = 5) y SVM (kernel RBF).
3. Validación con `StratifiedGroupKFold` (hasta 5 pliegues): las épocas se agrupan en **bloques de 10 s** y cada bloque va completo a entrenamiento o a prueba, para evitar que épocas vecinas queden en ambos lados.
4. Métricas: exactitud y F1 macro (promedio y desviación entre pliegues).
5. Para el mejor modelo se obtienen la matriz de confusión y su desglose por archivo; con Random Forest se obtienen las 10 características más importantes.
6. Prueba de la **variable de confusión por dispositivo**: solo se ejecuta si hay más de un equipo.
7. La función `experimento()` repite todo el flujo con otros parámetros y se usó para el cuestionario final: música relajante vs. estresante, con y sin ICA, notch de 40 Hz, 60 Hz o ninguno, sin quitar líneas angostas, con normalización y con épocas de 1, 2 y 4 s.

---

## 3. Resultados

### 3.1 Inventario y calidad de los registros

Se analizaron 5 registros de EEG de una participante, todos con el dispositivo `0C:26`, canal A2, llevados a **100 Hz** (Nyquist = 50 Hz), sin paquetes perdidos.

**Tabla 1. Inventario de los registros**

| Archivo | Condición | Hora | Duración (s) | Fs (Hz) | Paquetes perdidos | Muestras saturadas | % saturado | Segundo con saturación |
|---|---|---|---:|---:|---:|---:|---:|---|
| basal.txt | basal | 15:33:59 | 61.0 | 100 | 0 | 0 | 0.00 | – |
| abre_cierra.txt | abre_cierra | 15:36:38 | 46.8 | 100 | 0 | 2 | 0.04 | 41 |
| preguntas_susurradas.txt | preguntas_susurradas | 15:40:14 | 157.7 | 100 | 0 | 1 | 0.01 | 146 |
| musica_tranquila.txt | musica_relajante | 15:44:09 | 61.2 | 100 | 0 | 0 | 0.00 | – |
| musica_hard.txt | musica_estresante | 15:45:59 | 61.2 | 100 | 0 | 2 | 0.03 | 51 |

<img width="1387" height="1037" alt="l11_1" src="https://github.com/user-attachments/assets/3c09bba2-aa11-4ae3-9701-1830b1c2b637" />

*Figura 1. Señales EEG crudas (10 s a partir del segundo 10) de los cinco registros, con el rango del sensor (±39.5 µV) marcado con líneas punteadas.*


El pico de 40 Hz sobresale entre 0.9 y 1.5 veces respecto de sus vecinos (basal 1.0; abre_cierra 0.9; preguntas_susurradas 0.9; musica_tranquila 1.5; musica_hard 1.2), es decir, no hay interferencia en 40 Hz. El detector de líneas angostas solo encontró una: **1.9 Hz (8.3 % de la potencia) en `basal.txt`**.

**Tabla 2. Control de calidad automático**

| Archivo | Condición | % saturado | Pico 40 Hz (veces) | Otras líneas (Hz) | Desv. estándar (µV) | Curtosis | Estado |
|---|---|---:|---:|---|---:|---:|:---:|
| basal.txt | basal | 0.00 | 1.0 | 1.9 | 5.96 | 1.7 | ⚠️ |
| abre_cierra.txt | abre_cierra | 0.04 | 0.9 | – | 6.53 | 1.1 | ⚠️ |
| preguntas_susurradas.txt | preguntas_susurradas | 0.01 | 0.9 | – | 6.27 | 1.3 | ⚠️ |
| musica_tranquila.txt | musica_relajante | 0.00 | 1.5 | – | 5.34 | 2.7 | ✅ |
| musica_hard.txt | musica_estresante | 0.03 | 1.2 | – | 5.15 | 3.5 | ⚠️ |

### 3.2 Filtrado

<img width="1487" height="387" alt="l11_2" src="https://github.com/user-attachments/assets/2f494402-5f4b-4d16-b56f-31f7983738ca" />

*Figura 2. Efecto del filtrado pasa-banda (1–45 Hz) sobre `musica_tranquila.txt`, el registro con mayor pico en 40 Hz (1.5 veces). Izquierda: 2 s de señal cruda y filtrada. Derecha: PSD antes y después.* 

### 3.3 Ritmos cerebrales

<img width="1387" height="986" alt="l11_3" src="https://github.com/user-attachments/assets/c9c70264-e3fc-492c-9a5e-f3ea62585b2a" />

*Figura 3. EEG filtrado del registro `basal.txt` (segundos 10 a 15) y su descomposición en las bandas delta, theta, alfa, beta y gamma.* (Salida de la celda B.3)

**Tabla 3. Potencia relativa por banda (%), señal filtrada sin muestras saturadas**

| Registro | Delta | Theta | Alfa | Beta | Gamma |
|---|---:|---:|---:|---:|---:|
| basal.txt | 59.3 | 20.2 | 9.6 | 8.6 | 2.2 |
| abre_cierra.txt | 59.6 | 18.9 | 8.1 | 10.9 | 2.6 |
| preguntas_susurradas.txt | 54.9 | 16.7 | 8.2 | 14.4 | 5.9 |
| musica_tranquila.txt (relajante) | 57.2 | 16.7 | 9.0 | 13.1 | 4.0 |
| musica_hard.txt (estresante) | 49.2 | 20.4 | 8.4 | 14.8 | 7.3 |

<img width="1186" height="431" alt="l11_4" src="https://github.com/user-attachments/assets/72138b29-55b7-470f-b1ca-0d8a1af6a283" />

*Figura 4. Potencia relativa por banda en cada registro (barras apiladas).* (Salida de la celda B.3)

### 3.4 Filtro ICA

<img width="1487" height="653" alt="l11_5" src="https://github.com/user-attachments/assets/3a001ad2-c473-4019-a358-2cbb25b11ceb" />

*Figura 5. Demostración de ICA con señales simuladas: fuentes reales (alfa, parpadeo y músculo), mezcla en tres electrodos y componentes recuperadas por FastICA, con su curtosis y la fracción de potencia sobre 20 Hz.* 

**Tabla 4. Componentes de Wavelet-ICA en `abre_cierra.txt`**

| IC | Curtosis | Fracción > 20 Hz | Fracción 38–42 Hz | % varianza | Sub-banda dominante | Acción |
|---:|---:|---:|---:|---:|---|---|
| 0 | 1.087 | 0.225 | 0.028 | 8.5 | D2 | Conservar |
| 1 | 0.268 | 0.097 | 0.019 | 14.1 | D3 | Conservar |
| 2 | 0.646 | 0.889 | 0.111 | 4.2 | D1 | Conservar |
| 3 | 0.356 | 0.001 | 0.000 | 18.9 | D5 | Conservar |
| 4 | 1.089 | 0.000 | 0.000 | 25.4 | A5 | Conservar |
| 5 | 4.860 | 0.000 | 0.000 | 28.9 | D4 | Conservar |

Varianza eliminada por el ICA en `abre_cierra.txt`: **0.0 %**.


**Tabla 5. Resultado del ICA en todos los registros**

| Registro | ICs tratadas | % varianza eliminada |
|---|---|---:|
| basal.txt | ninguna | 0.0 |
| abre_cierra.txt | ninguna | 0.0 |
| preguntas_susurradas.txt | ninguna | 0.0 |
| musica_tranquila.txt | ninguna | 0.0 |
| musica_hard.txt | ninguna | 0.0 |

### 3.5 Épocas de 2 s

**Tabla 6. Épocas por registro**

| Registro | Épocas totales | Épocas válidas | Rechazadas por saturación |
|---|---:|---:|---:|
| basal.txt | 28 | 28 | 0 |
| abre_cierra.txt | 21 | 20 | 1 |
| preguntas_susurradas.txt | 76 | 75 | 1 |
| musica_tranquila.txt | 28 | 28 | 0 |
| musica_hard.txt | 28 | 27 | 1 |

### 3.6 Extracción de características

Se obtuvo una tabla de **178 épocas × 22 características**: 28 de `basal`, 20 de `abre_cierra`, 75 de `preguntas_susurradas`, 27 de `musica_estresante` y 28 de `musica_relajante`. Como ejemplo, una época de `basal.txt` tuvo `rel_delta` = 0.671, `rel_theta` = 0.162, `f_pico` = 1.5 Hz, `sef90` = 10 Hz, `entropia` = 0.638, `hj_complejidad` = 2.709 y `pico_a_pico` = 25.3 µV.

**Tabla 7. Promedio de cada característica por condición**

| Característica | basal | abre_cierra | preguntas_susurradas | musica_estresante | musica_relajante |
|---|---:|---:|---:|---:|---:|
| abs_delta | 1.373 | 1.355 | 1.402 | 1.061 | 1.209 |
| rel_delta | 0.534 | 0.544 | 0.506 | 0.378 | 0.487 |
| abs_theta | 0.969 | 0.845 | 0.923 | 0.830 | 0.742 |
| rel_theta | 0.223 | 0.183 | 0.184 | 0.229 | 0.173 |
| abs_alfa | 0.662 | 0.596 | 0.616 | 0.529 | 0.508 |
| rel_alfa | 0.107 | 0.098 | 0.089 | 0.115 | 0.103 |
| abs_beta | 0.637 | 0.738 | 0.880 | 0.751 | 0.755 |
| rel_beta | 0.108 | 0.140 | 0.155 | 0.192 | 0.178 |
| abs_gamma | 0.049 | 0.137 | 0.481 | 0.387 | 0.262 |
| rel_gamma | 0.028 | 0.035 | 0.067 | 0.085 | 0.059 |
| theta_beta | 2.607 | 1.551 | 1.395 | 1.498 | 1.156 |
| alfa_beta | 1.181 | 0.800 | 0.625 | 0.700 | 0.644 |
| f_pico (Hz) | 2.125 | 2.225 | 2.393 | 2.778 | 2.357 |
| sef90 (Hz) | 15.839 | 18.725 | 23.613 | 26.926 | 23.821 |
| entropia | 0.666 | 0.673 | 0.705 | 0.763 | 0.729 |
| hj_actividad | 1.389 | 1.400 | 1.424 | 1.255 | 1.312 |
| hj_movilidad | 0.529 | 0.573 | 0.701 | 0.763 | 0.677 |
| hj_complejidad | 2.486 | 2.390 | 2.106 | 1.963 | 2.147 |
| desv_est (µV) | 5.011 | 5.095 | 5.243 | 4.316 | 4.642 |
| asimetria | 0.271 | 0.069 | 0.109 | 0.052 | 0.260 |
| curtosis | 0.867 | 0.217 | 0.326 | 0.430 | 0.847 |
| pico_a_pico (µV) | 28.457 | 27.643 | 29.341 | 25.025 | 26.753 |

<img width="1586" height="460" alt="l11_6" src="https://github.com/user-attachments/assets/1d9a3a8e-fd5e-4a04-99fa-896aa234d060" />

*Figura 6. (Izquierda) Mapa de calor de la potencia relativa media por banda y condición. (Derecha) Distribución estandarizada de `rel_delta`, `rel_theta`, `rel_alfa`, `rel_beta`, `entropia`, `hj_movilidad` y `curtosis` por condición.*

### 3.7 Clasificación y comparación de algoritmos

Validación: 178 épocas, 39 bloques de 10 s, 5 pliegues, 5 clases (**azar = 0.20**).

**Tabla 8. Comparación de los cinco algoritmos (5 condiciones)**

| Modelo | Exactitud | F1 macro | ± F1 |
|---|---:|---:|---:|
| **Regresión Logística** | 0.440 | **0.275** | 0.107 |
| Árbol de Decisión | 0.349 | 0.260 | 0.077 |
| Random Forest | 0.424 | 0.239 | 0.037 |
| KNN (k = 5) | 0.353 | 0.231 | 0.037 |
| SVM (RBF) | 0.441 | 0.212 | 0.053 |

**Tabla 9. Predicciones del mejor modelo (Regresión Logística) por archivo**

| Archivo real | Pred. basal | Pred. abre_cierra | Pred. preguntas_susurradas | Pred. musica_estresante | Pred. musica_relajante |
|---|---:|---:|---:|---:|---:|
| basal.txt | **12** | 8 | 1 | 2 | 5 |
| abre_cierra.txt | 5 | **3** | 7 | 3 | 2 |
| preguntas_susurradas.txt | 5 | 1 | **57** | 7 | 5 |
| musica_hard.txt (estresante) | 1 | 0 | 14 | **4** | 8 |
| musica_tranquila.txt (relajante) | 4 | 3 | 12 | 6 | **3** |

<img width="1593" height="536" alt="l11_7" src="https://github.com/user-attachments/assets/4ca7dd2e-fff6-453f-b40f-98d71505ad58" />

*Figura 7. (Izquierda) Matriz de confusión de la Regresión Logística con validación por bloques. (Derecha) Las 10 características más importantes según Random Forest.* 

### 3.8 Experimentos del cuestionario final

**Música relajante vs. estresante (mismo dispositivo, solo 2 clases).** El ICA no eliminó ninguna componente en ambos registros (0 de red y 0 de transitorios).

| Modelo | F1 macro | ± F1 |
|---|---:|---:|
| Árbol de Decisión | 0.588 | 0.222 |
| Regresión Logística | 0.524 | 0.229 |
| Random Forest | 0.516 | 0.198 |
| KNN (k = 5) | 0.513 | 0.197 |
| SVM (RBF) | 0.403 | 0.153 |

| Característica | musica_estresante | musica_relajante |
|---|---:|---:|
| rel_delta (%) | 37.820 | 48.658 |
| rel_theta (%) | 22.947 | 17.261 |
| rel_alfa (%) | 11.534 | 10.312 |
| rel_beta (%) | 19.155 | 17.833 |
| rel_gamma (%) | 8.533 | 5.923 |
| entropia | 0.763 | 0.729 |
| f_pico (Hz) | 2.778 | 2.357 |
| theta_beta | 1.498 | 1.156 |
| alfa_beta | 0.700 | 0.644 |

**¿Ayuda el ICA?** El F1 de los cinco modelos es **idéntico con y sin ICA** (por ejemplo, Regresión Logística 0.275 rechazando épocas saturadas y 0.265 sin rechazarlas, en ambos casos). En `abre_cierra` los promedios de `rel_delta` (0.544) y `rel_gamma` (0.035) tampoco cambian.

| Configuración (`abre_cierra`) | Épocas | Curtosis media | Pico a pico medio (µV) |
|---|---:|---:|---:|
| Con o sin ICA, rechazando saturadas | 20 | 0.217 | 27.643 |
| Con o sin ICA, **sin** rechazar saturadas | 21 | 0.233 | 28.564 |

**Notch de 40 Hz, 60 Hz o ninguno** (sin quitar líneas angostas):

| Configuración | Pico 40 Hz tras procesar | ICs de red eliminadas | Mejor F1 (Árbol) |
|---|---|---:|---:|
| Notch 40 Hz | 0.0 | 0 | 0.293 |
| Notch 60 Hz | 0.9–1.4 (no se aplicó: 60 Hz > Nyquist) | 0 | 0.330 |
| Sin notch | 0.9–1.4 | 0 | 0.330 |

| `rel_gamma` (%) | notch 40 Hz | notch 60 Hz | sin notch |
|---|---:|---:|---:|
| basal | 2.2 | 2.7 | 2.7 |
| abre_cierra | 2.7 | 3.5 | 3.5 |
| preguntas_susurradas | 5.5 | 6.7 | 6.7 |
| musica_estresante | 6.6 | 8.5 | 8.5 |
| musica_relajante | 4.7 | 5.9 | 5.9 |

**Sin quitar la línea angosta de 1.9 Hz:** el mejor F1 fue 0.330 (frente a 0.275 quitándola). En `basal.txt` la `rel_theta` pasó de 22.3 % a 22.0 % y la mediana de `f_pico` de 1.5 a 2.0 Hz; los demás registros no cambiaron.

**Normalización (z-score por registro):**

| Modelo | F1 sin normalizar | F1 normalizada |
|---|---:|---:|
| Árbol de Decisión | 0.260 | 0.247 |
| Regresión Logística | 0.275 | 0.252 |
| Random Forest | 0.239 | 0.256 |
| KNN (k = 5) | 0.231 | 0.208 |
| SVM (RBF) | 0.212 | 0.194 |

Al normalizar, `desv_est` queda entre 0.93 y 0.99, `hj_actividad` entre −0.07 y −0.02 y `pico_a_pico` entre 5.15 y 5.63; antes iban de 4.32 a 5.24, de 1.26 a 1.42 y de 25.0 a 29.3 µV, respectivamente.

**Duración de la época:**

| Época | Resolución 1/T | Épocas válidas | Mejor modelo | Mejor F1 | F1 promedio de los 5 modelos |
|---|---:|---:|---|---:|---:|
| 1 s | 1.00 Hz | 363 | Random Forest | 0.270 | 0.244 |
| 2 s | 0.50 Hz | 178 | Regresión Logística | 0.275 | 0.243 |
| 4 s | 0.25 Hz | 87 | Random Forest | **0.468** | **0.402** |

---

## 4. Explicación de los resultados

### 4.1 Calidad de los registros

- **Datos mucho más limpios que con el canal saturado.** Con el canal A2 la saturación es de 0.00 % a 0.04 % (como máximo 2 muestras por registro) y no hay paquetes perdidos. Por eso ningún registro quedó en ❌: cuatro quedaron en ⚠️ por saturaciones mínimas o por la línea de 1.9 Hz, y `musica_tranquila` quedó en ✅.
- **Sin interferencia de red.** El pico de 40 Hz es de 0.9 a 1.5 veces el de sus vecinos. Esto confirma que quitar los 60 Hz con un notch a 1000 Hz, antes de diezmar a 100 Hz, evitó el alias que, de muestrear directamente a 100 Hz, habría aparecido en 40 Hz. Por eso `NOTCH_HZ = None` es la decisión correcta.
- **Línea de 1.9 Hz en `basal.txt`.** Concentra 8.3 % de la potencia de 1 a 45 Hz y cae en la banda delta. Una línea tan angosta no es un ritmo cerebral sino una interferencia o deriva, de modo que se quitó con un notch angosto.
- **Amplitud y forma.** La desviación estándar es de 5.15 a 6.53 µV, menor que los 20–100 µV típicos del EEG de cuero cabelludo, y la curtosis es baja (1.1 a 3.5): no hay saltos grandes ni eventos extremos.

### 4.2 Ritmos cerebrales

- **Predomina la delta en todos los registros** (49.2 % a 59.6 %), seguida de theta (16.7 % a 20.4 %). La alfa es baja (8.1 % a 9.6 %) y no se observa una alfa dominante en el registro basal. Parte de la delta puede deberse a deriva lenta y a movimientos oculares, que ocupan la misma banda (< 4 Hz), y a que la derivación no sea occipital, que es donde la alfa de reposo es más fuerte.
- **Los registros de mayor actividad muestran más beta y gamma.** Del basal (beta 8.6 %, gamma 2.2 %) se pasa a `preguntas_susurradas` (14.4 % y 5.9 %), `musica_tranquila` (13.1 % y 4.0 %) y `musica_hard` (14.8 % y 7.3 %), que además es el único con delta claramente menor (49.2 %). Más beta y gamma es compatible con mayor activación, aunque en `preguntas_susurradas` también puede reflejar actividad muscular al hablar, ya que la gamma de superficie suele estar contaminada por EMG.
- Con una sola grabación por condición no se puede atribuir estas diferencias a la música: podrían deberse al momento de la sesión, al nivel de relajación o al movimiento.

### 4.3 Filtro ICA

- **Ninguna componente fue marcada como artefacto** en ninguno de los registros, por lo que la varianza eliminada fue de 0.0 %. En `abre_cierra` la componente de mayor curtosis (IC 5) llegó a 4.86, por debajo del umbral de 6, y la mayor fracción de potencia en 38–42 Hz fue 0.111 (IC 2), muy por debajo del umbral de 0.4.
- Esto es coherente con el análisis previo: no hay red eléctrica residual ni transitorios bruscos que corregir. El método no "falló": simplemente no había artefactos que cumplieran sus criterios.
- Un límite del método es que, con un solo canal, Wavelet-ICA solo detecta transitorios (curtosis alta) y red eléctrica. No puede separar artefactos continuos, como parpadeos rítmicos o actividad muscular sostenida, que se superponen en frecuencia con el EEG. Por eso el alto porcentaje de delta podría incluir contaminación ocular que el ICA de un canal no remueve.
- Los rótulos de las sub-bandas (D1 de 25–50 Hz, …, A5 de 0–1.6 Hz) son correctos aquí porque la señal ya está a 100 Hz.

### 4.4 Características

- **Las características que más varían entre condiciones** son las espectrales y las de forma de onda: de `basal` a `musica_estresante`, `sef90` sube de 15.8 a 26.9 Hz, la entropía de 0.666 a 0.763, la movilidad de Hjorth de 0.529 a 0.763 y la `rel_gamma` de 0.028 a 0.085, mientras la complejidad de Hjorth baja de 2.486 a 1.963. Todas indican que la energía se desplaza de las frecuencias lentas a las rápidas.
- **`basal` tiene el cociente `theta_beta` más alto** (2.607 frente a 1.156–1.551 en las demás), porque es el registro con menos beta.
- **`basal` y `abre_cierra` son muy parecidas** (por ejemplo, `rel_delta` 0.534 y 0.544; `f_pico` 2.1 y 2.2 Hz), lo que anticipa que será difícil separarlas.
- **Las características de amplitud** (`desv_est`, `pico_a_pico`, `hj_actividad`) varían poco entre condiciones (`desv_est` de 4.3 a 5.2 µV), y `f_pico` queda entre 2.1 y 2.8 Hz en todas, es decir, dentro de la delta.

### 4.5 Clasificación

- **El rendimiento es apenas superior al azar.** El mejor F1 macro es 0.275 (Regresión Logística) frente a un azar de 0.20, y el peor es 0.212 (SVM). Las diferencias entre modelos (0.21–0.28) son menores que la variación entre pliegues (hasta ± 0.107), por lo que no se puede afirmar que un algoritmo sea mejor que otro.
- **La exactitud (0.35–0.44) engaña por el desbalance de clases.** `preguntas_susurradas` aporta 75 de las 178 épocas (42 %): una exactitud de 0.44 equivale a predecir casi siempre esa clase. La matriz de confusión lo confirma: el modelo acierta 57 de 75 épocas de `preguntas_susurradas` (76 %), pero solo 12 de 28 de `basal`, 3 de 20 de `abre_cierra`, 4 de 27 de `musica_estresante` y 3 de 28 de `musica_relajante`. Además, 91 de las 178 predicciones caen en `preguntas_susurradas`.
- **Las condiciones se parecen demasiado.** Los espectros están dominados por la delta en todas las condiciones, hay una sola participante y una grabación por condición, y el EEG es de un solo canal. Dos pares son especialmente difíciles: `basal` con `abre_cierra`, y música relajante con estresante (14 y 12 épocas de música son predichas como `preguntas_susurradas`).
- **El resultado es honesto, no inflado.** La validación por bloques de 10 s impide que épocas consecutivas estén en entrenamiento y prueba, y el único dispositivo elimina la confusión por equipo. Un F1 bajo con esta validación es más creíble que un F1 alto obtenido con fuga de información.
- **Música relajante vs. estresante.** Con dos clases el mejor F1 fue 0.588 (Árbol de Decisión), solo algo mejor que el azar de 0.5 y con una desviación de ± 0.222, por lo que es inestable. La diferencia fisiológica más marcada fue una delta mayor en la relajante (48.7 % frente a 37.8 %) y más theta y gamma en la estresante (22.9 % y 8.5 % frente a 17.3 % y 5.9 %).


---

## 5. Conclusiones

1. **La calidad de los datos fue buena tras elegir el canal A2:** saturación de 0.00 % a 0.04 %, sin paquetes perdidos y sin interferencia de red. Solo se detectó una línea angosta de 1.9 Hz en `basal.txt`, que se quitó con un notch angosto.
2. La adaptación de 1000 Hz a 100 Hz fue correcta, ya que quitar los 60 Hz antes de diezmar evitó el alias de 40 Hz, por lo que el notch de 40 Hz no es necesario y solo elimina actividad real (menos `rel_gamma`).
3. Domina la delta en todos los registros (49 % a 60 %) y la alfa es baja (8 % a 10 %). Las condiciones de más actividad, sobre todo `musica_hard` y `preguntas_susurradas`, tienen más beta y gamma, aunque en esta última puede haber actividad muscular.
4. Wavelet-ICA no marcó ningún artefacto y no modificó la señal (0.0 % de varianza eliminada), porque no hay transitorios ni red residual. No es capaz de separar artefactos continuos que se superponen con el EEG.
5. Las 22 características (bandas, cocientes, espectrales, Hjorth y estadísticas) muestran un desplazamiento hacia frecuencias rápidas en `preguntas_susurradas` y en las músicas respecto del `basal`, con `sef90`, entropía, movilidad de Hjorth y `rel_gamma` como las más sensibles.
6. **Clasificación:** con 5 condiciones los cinco algoritmos rinden apenas por encima del azar (F1 macro de 0.212 a 0.275 frente a 0.20), con la Regresión Logística como la mejor, y el resultado está sesgado hacia la clase mayoritaria `preguntas_susurradas`. Las diferencias entre algoritmos no son significativas frente a la variación entre pliegues.

