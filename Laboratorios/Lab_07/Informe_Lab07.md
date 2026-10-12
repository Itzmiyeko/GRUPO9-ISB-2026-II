# Resumen — Lab 7: Filtrado adaptativo y Transformada Wavelet en señales biomédicas

**Señales analizadas:** EMG brazo (oposición) · ECG DIII (aeróbico) · ECG DII (hipoventilación) · EEG (abre-cierra)



## 1. Metodología común

En los cuatro laboratorios se siguió el mismo flujo: **señal real → contaminación artificial controlada → filtrado → evaluación contra la señal original**.

| Elemento | Detalle |
|---|---|
| Contaminación | Interferencia de 60 Hz (amplitud 1.0·σ), armónico de 120 Hz (0.6·σ) y ruido gaussiano (0.3·σ), donde σ es la desviación estándar de la señal original |
| **A** — Pasa-banda | Butterworth orden 4 (0.5–40 Hz en ECG y EEG; 20–450 Hz en EMG) |
| **B1 / B2** — Notch | 60 Hz solo (B1) y 60 + 120 Hz en cascada (B2), Q = 30 |
| **LMS** | Filtro adaptativo con referencias seno/coseno de 60 y 120 Hz, μ = 0.005 |
| **Wavelet** | db4, 8 niveles, umbral universal (soft thresholding) |
| **Híbrido** | Notch 60 + 120 Hz seguido de Wavelet |
| Métricas | RMSE (menor es mejor), SNR en dB (mayor es mejor) y correlación r con el original |

---

## 2. Datos de cada registro

| Señal | Archivo | Canal | Fs | Muestras | Duración | Unidad | Nyquist |
|---|---|---|---|---|---|---|---|
| EMG brazo (oposición) | `OPOSICION 02.txt` | A6 | 1000 Hz | 33 900 | 33.9 s | ADC | 500 Hz |
| ECG DIII (aeróbico) | `act DIII.txt` | A2 | **100 Hz** | 3 045 | 30.45 s | mV | **50 Hz** |
| ECG DII (hipoventilación) | `Hipoventilacion DII.txt` | A2 | **100 Hz** | 3 525 | 35.25 s | mV | **50 Hz** |
| EEG (abre-cierra) | `abre_cierra_vivi.txt` | A2 | 1000 Hz | 46 800 | 46.8 s | mV* | 500 Hz |

\* Ver nota sobre la conversión de unidades del EEG en la sección 7.

En los cuatro registros el contador `nSeq` fue continuo: **0 paquetes perdidos**.

---

## 3. EMG brazo — oposición

**Registro:** canal A6, trabajado en unidades ADC centradas en cero (no se aplica la ganancia del sensor ECG). Rango de -510.6 a 512.4 ADC y σ = 72.7 ADC.

**Contaminación simulada:** 72.7 ADC a 60 Hz, 43.6 ADC a 120 Hz y ruido gaussiano de σ = 21.8 ADC.

| Método | RMSE (ADC) | SNR (dB) | r | Mejora SNR (dB) |
|---|---:|---:|---:|---:|
| Contaminado (sin filtrar) | 63.62 | 1.70 | 0.7715 | — |
| A — Pasa-banda 20–450 Hz | 63.34 | 1.74 | 0.7689 | +0.04 |
| B1 — Notch 60 Hz | 39.86 | 5.76 | 0.8795 | +4.06 |
| **B2 — Notch 60 + 120 Hz** | **26.17** | **9.41** | **0.9421** | **+7.72** |
| LMS adaptativo | 27.80 | 8.89 | 0.9373 | +7.19 |
| Wavelet (denoising) | 43.18 | 5.06 | 0.8297 | +3.37 |
| Híbrido notch + Wavelet | 39.30 | 5.88 | 0.8940 | +4.19 |

GRAFICA:

<img width="1387" height="1277" alt="emg1_lab7" src="https://github.com/user-attachments/assets/6490b45b-c5b7-4565-a045-78b8884a75fa" />


**Lectura:**
- El pasa-banda **casi no mejoró nada**: 60 y 120 Hz caen dentro de su banda de paso (20–450 Hz), así que no los puede quitar.
- Los métodos que atacan directamente las interferencias (notch en cascada y LMS) fueron los mejores y quedaron muy cerca entre sí.
- El Wavelet con umbral universal quedó por debajo del notch. Probable causa: el EMG es de banda ancha y el nivel D1 (250–500 Hz), usado para estimar el ruido, contiene señal real; eso infla el umbral (109.2 ADC frente a σ = 72.7 ADC) y se elimina también parte del EMG.
- Efecto sobre la señal sin contaminar: el pasa-banda la distorsiona poco (SNR 21.4 dB, r = 0.9964), mientras que el notch 60 + 120 Hz sí la altera (RMSE 14.7 ADC, SNR 14.4 dB, r = 0.9823), porque el EMG tiene energía real cerca de esas frecuencias.

---

## 4. ECG DIII — aeróbico

**Registro:** canal A2, 100 Hz de muestreo. Rango de -1.000 a 1.499 mV y σ = 0.326 mV.

**Contaminación simulada:** 0.326 mV a 60 Hz, 0.196 mV a 120 Hz y ruido gaussiano de σ = 0.098 mV.

**Punto crítico:** con Fs = 100 Hz el límite de Nyquist es 50 Hz, por lo que 60 y 120 Hz están por encima. El notebook **no pudo diseñar los notch** (quedan idénticos a la señal contaminada). Matemáticamente, esas senoides muestreadas a 100 Hz se pliegan (aliasing) a 40 Hz y 20 Hz, es decir, quedan dentro de la banda del ECG.
Grafica:

<img width="1401" height="1277" alt="ECG_aero_lab7" src="https://github.com/user-attachments/assets/6385053a-4e9f-4a58-9786-7cc834db0a1e" />


| Método | RMSE (mV) | SNR (dB) | r | Mejora SNR (dB) |
|---|---:|---:|---:|---:|
| Contaminado (sin filtrar) | 0.2878 | 1.10 | 0.7191 | — |
| A — Pasa-banda 0.5–40 Hz | 0.2120 | 3.75 | 0.8207 | +2.65 |
| B1 — Notch 60 Hz | 0.2878 | 1.10 | 0.7191 | 0.00 (no aplicado) |
| B2 — Notch 60 + 120 Hz | 0.2878 | 1.10 | 0.7191 | 0.00 (no aplicado) |
| **LMS adaptativo** | **0.1133** | **9.19** | **0.9418** | **+8.09** |
| Wavelet (denoising) | 0.3223 | 0.11 | 0.2435 | -0.98 |
| Híbrido notch + Wavelet | 0.3223 | 0.11 | 0.2435 | -0.98 |

**Lectura:**
- El **LMS fue claramente el mejor**: sus referencias se generan con la misma frecuencia de muestreo, así que reproducen exactamente la interferencia plegada y la cancelan. El notch, que depende de una frecuencia válida por debajo de Nyquist, no pudo actuar.
- El pasa-banda mejoró algo (+2.65 dB), pero no puede quitar una interferencia que cae dentro de su banda.
- El Wavelet **empeoró** la señal (SNR 0.11 dB, r = 0.24). Probable causa: el umbral universal (1.4874 mV frente a σ = 0.326 mV) resultó demasiado alto, porque el nivel D1 (25–50 Hz) contiene la interferencia plegada y sobreestima el ruido; así se eliminó también el QRS. El híbrido da el mismo resultado porque el notch no se aplicó.
- Sobre la señal sin contaminar, el pasa-banda 0.5–40 Hz apenas la altera (RMSE 0.030 mV, SNR 20.8 dB, r = 0.9958).

---

## 5. ECG DII — hipoventilación

**Registro:** canal A2, 100 Hz de muestreo. Rango de -0.748 a 1.153 mV y σ = 0.216 mV.

**Contaminación simulada:** 0.216 mV a 60 Hz, 0.129 mV a 120 Hz y ruido gaussiano de σ = 0.065 mV.

Aquí también 60 y 120 Hz superan el límite de Nyquist (50 Hz), por lo que **no se aplicaron los notch**.

| Método | RMSE (mV) | SNR (dB) | r |
|---|---:|---:|---:|
| Contaminado (sin filtrar) | 0.1907 | 1.09 | 0.7536 |
| A — Pasa-banda 0.5–40 Hz | 0.1351 | 4.09 | 0.8467 |
| B1 — Notch 60 Hz | 0.1907 | 1.09 | 0.7536 |
| B2 — Notch 60 + 120 Hz | 0.1907 | 1.09 | 0.7536 |

**Lectura:**
- El comportamiento es el mismo que en el ECG DIII: el pasa-banda mejora (+3.0 dB) y los notch no tienen efecto por la limitación de Nyquist.
- **Este notebook solo está ejecutado hasta la comparación de los filtros convencionales.** Faltan los resultados de LMS, Wavelet y método híbrido, y no se calcularon indicadores fisiológicos (por ejemplo, frecuencia cardiaca), por lo que no se puede concluir nada sobre el efecto de la hipoventilación en el ECG a partir de estos archivos.
- Dado que comparte Fs y estructura con el ECG DIII, es esperable que el LMS sea el método más adecuado, pero eso queda por comprobar ejecutando el resto del notebook.

---

## 6. EEG — abre-cierra

**Registro:** canal A2, 1000 Hz de muestreo, 46.8 s. Rango de -1.497 a 0.984 mV y σ = 0.262 mV. El archivo incluye además los canales A5 y A6, que no se analizan en el notebook.

**Contaminación simulada:** 0.262 mV a 60 Hz, 0.157 mV a 120 Hz y ruido gaussiano de σ = 0.079 mV. Con Fs = 1000 Hz, 60 y 120 Hz están por debajo de Nyquist, así que **todos los métodos pudieron aplicarse**.

GRAFICA: 

<img width="1387" height="1277" alt="eeg_LAB7" src="https://github.com/user-attachments/assets/8c0261c9-cc9c-4ab0-9ad8-d03a26a205a4" />


| Método | RMSE (mV) | SNR (dB) | r | Mejora SNR (dB) |
|---|---:|---:|---:|---:|
| Contaminado (sin filtrar) | 0.2293 | 1.23 | 0.7410 | — |
| **A — Pasa-banda 0.5–40 Hz** | **0.1082** | **7.76** | 0.9127 | **+6.53** |
| B1 — Notch 60 Hz | 0.1552 | 4.62 | 0.8447 | +3.39 |
| B2 — Notch 60 + 120 Hz | 0.1084 | 7.74 | **0.9160** | +6.51 |
| LMS adaptativo | 0.1096 | 7.64 | 0.9152 | +6.41 |
| Wavelet (denoising) | 0.1268 | 6.37 | 0.8781 | +5.14 |
| Híbrido notch + Wavelet | 0.1121 | 7.45 | 0.9083 | +6.22 |

**Lectura:**
- Es el caso donde **los métodos rinden de forma más pareja**: pasa-banda, notch 60 + 120 Hz y LMS quedan a menos de 0.15 dB entre sí.
- El notch solo a 60 Hz es claramente insuficiente (+3.39 dB): hay que atacar también el armónico de 120 Hz.
- El Wavelet es el más débil de los métodos que sí funcionaron, pero aun así mejora la señal en 5.1 dB.
- **Sobre la señal sin contaminar**, tanto el pasa-banda (RMSE 0.105 mV, SNR 8.0 dB) como el notch (RMSE 0.076 mV, SNR 10.9 dB) producen cambios apreciables. Esto sugiere que el registro original ya contiene energía fuera de 0.5–40 Hz y cerca de 60/120 Hz (probablemente interferencia real de red), por lo que parte del "error" medido es distorsión del filtro y no ruido residual.
- El notebook no separa las condiciones "abre" y "cierra", así que no se evaluó el contenido espectral propio del EEG (por ejemplo, el ritmo alfa) en cada condición.

---

## 7. Conclusiones

1. **No existe un método universalmente mejor.** El mejor filtro cambió con la señal: LMS en los ECG de 100 Hz, notch 60 + 120 Hz en el EMG, y pasa-banda / notch / LMS prácticamente empatados en el EEG.

2. **La frecuencia de muestreo condiciona todo el diseño del filtrado.** Con Fs = 100 Hz, 60 y 120 Hz superan Nyquist (50 Hz): los notch no se pudieron diseñar y la interferencia simulada queda plegada dentro de la banda del ECG. Para trabajar con interferencia de red real conviene muestrear a una Fs claramente mayor que 120 Hz.

3. **El filtro adaptativo (LMS) fue el más robusto cuando el filtro fijo no podía actuar.** En los ECG a 100 Hz, el LMS mejoró la SNR en 8.1 dB (ECG DIII) mientras el notch no tenía efecto. En EMG y EEG se comportó casi igual que el notch en cascada. Su limitación: necesita una señal de referencia correlacionada con la interferencia y no puede cancelar el ruido gaussiano.

4. **El pasa-banda solo sirve si la interferencia cae fuera de su banda.** En el EMG (20–450 Hz) fue prácticamente inútil contra 60/120 Hz (+0.04 dB); en el EEG sí rindió bien (+6.53 dB) porque 60 y 120 Hz quedan fuera de 0.5–40 Hz.

5. **El Wavelet con umbral universal no es una solución automática.** En el EMG y en el ECG DIII quedó por debajo de los métodos dirigidos, e incluso empeoró el ECG DIII (-0.98 dB). Su rendimiento depende de cómo se estima el ruido (nivel D1) y de si la señal es de banda ancha o contiene interferencias dentro de esa banda.

