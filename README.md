# Caracterización y Mitigación de Ruido en Banda ISM 2.4 GHz (SDR · GNU Radio)

Banco de pruebas en GNU Radio que caracteriza empíricamente la degradación de un enlace en la banda ISM de 2.4 GHz (AWGN) e implementa mitigación de ruido mediante filtrado FIR digital en banda base. Se logra una mejora medida de la relación señal-ruido (SNR) de **+12.44 dB**.

> **Front-end RF objetivo:** HackRF One. La validación empírica de este repositorio se realiza sobre un **banco de pruebas simulado** en GNU Radio (bloque *Channel Model*, AWGN), de modo que el modelo de canal, la métrica de SNR y el resultado son reproducibles sin depender del espectro real ni del hardware. La cadena de procesado (ajuste de ganancias TX/RX en banda base + FIR) es directamente transferible a una fuente HackRF.

---

## Objetivos

- Caracterizar la degradación de un enlace en banda ISM 2.4 GHz bajo ruido aditivo gaussiano.
- Diseñar e implementar un filtro FIR paso-bajo para mitigar el ruido fuera de banda.
- Medir empíricamente la mejora de SNR y contrastarla con la predicción teórica.

## Especificaciones

| Parámetro | Valor |
|---|---|
| Frecuencia de muestreo (fs) | 20 MS/s |
| Ancho de banda de análisis | 20 MHz |
| Tipo de canal | AWGN (Channel Model, `noise_voltage=0.3`, seed fijo) |
| Filtro | FIR paso-bajo, ventana Hamming |
| Frecuencia de corte | 600 kHz |
| Ancho de transición | 200 kHz |
| Mejora de SNR medida | +12.44 dB |

## Stack

GNU Radio Companion 3.10.9.2 · Python 3 · (HackRF One como front-end objetivo)

---

## Arquitectura del sistema

![Diagrama de bloques](media/diagrama_bloques.svg)

La señal de interés se genera en banda base y se degrada con AWGN en el bloque *Channel Model* (equivalente a la contaminación del canal ISM 2.4 GHz). Tras el *Throttle*, la señal se divide en dos ramas paralelas:

- **Rama baseline:** mide la potencia total sin procesar (referencia "antes").
- **Rama mitigada:** aplica el FIR paso-bajo y mide la potencia residual (resultado "después").

Cada rama dispone de su propio visualizador de espectro (*QT GUI Frequency Sink*) y su medidor de potencia (*Probe Avg Mag² → Number Sink*).

---

## Metodología

### 1. Modelo de canal

El ruido se inyecta con `Channel Model` (AWGN, `noise_voltage = 0.3`). Se fija `noise_seed` para reproducibilidad. La potencia de ruido queda repartida uniformemente sobre los 20 MHz de la banda.

### 2. Diseño del filtro FIR

El filtro se calcula con `firdes.low_pass(gain, fs, cutoff, transition, window)`:

```python
firdes.low_pass(1, 20e6, 600e3, 200e3, window.WIN_HAMMING, 6.76)
```

**Fundamento del dimensionado.** La mejora de SNR por filtrado proviene de reducir el ancho de banda de ruido que llega al detector, conservando la señal:

```
Mejora_SNR (dB) = 10 · log10( BW_ruido / BW_filtro )
```

Para el objetivo de ~12 dB:

```
BW_filtro = BW_ruido / 10^(12/10) = 20 MHz / 15.85 ≈ 1.26 MHz
```

Con corte a 600 kHz (ancho de paso bilateral ≈ 1.2 MHz) se obtiene la mejora buscada.

**Longitud del filtro (nº de coeficientes).** Para el método de ventaneo, la longitud es inversamente proporcional al ancho de transición normalizado. Con ventana Hamming (factor ≈ 3.3):

```
N ≈ 3.3 · fs / Δf_transición = 3.3 · (20e6 / 200e3) ≈ 330 taps
```

Un ancho de transición más estrecho aumenta la selectividad a costa de más coeficientes (mayor carga computacional y retardo de grupo).

### 3. Medición de SNR (método ON/OFF)

El *Probe Avg Mag²* mide potencia **total** (señal + ruido). Para separar ambas componentes se ejecuta el flowgraph dos veces:

- **Señal ON** (`Amplitude = 1`): potencia total.
- **Señal OFF** (`Amplitude = 0`): potencia de ruido aislada.

La potencia de señal se obtiene por resta (`ON − OFF`), y de ahí la SNR de cada rama.

---

## Resultados

### Mediciones

| Medición | Baseline | Filtrada |
|---|---|---|
| Amplitude = 1 (señal ON) | 1.088968 | 1.005122 |
| Amplitude = 0 (solo ruido) | 0.088417 | 0.005031 |

### Cálculos

```
P_señal_baseline = 1.088968 − 0.088417 = 1.000551
P_señal_filtrada = 1.005122 − 0.005031 = 1.000091   (pérdida de inserción ≈ 0.002 dB)

SNR_antes   = 10·log10(1.000551 / 0.088417) = 10.54 dB
SNR_después = 10·log10(1.000091 / 0.005031) = 22.98 dB

MEJORA = 22.98 − 10.54 = 12.44 dB
```

### Verificación cruzada

La reducción de ruido medida, `10·log10(0.088417 / 0.005031) = 12.45 dB`, coincide con la predicción teórica por reducción de ancho de banda (~12.2 dB). El filtro conserva la señal casi intacta (pérdida de inserción ≈ 0.002 dB), confirmando que la mejora procede exclusivamente de la mitigación de ruido fuera de banda.

### Espectros

| Antes (baseline) | Después (filtrado) |
|---|---|
| Suelo de ruido ≈ −50 dB en toda la banda | Ruido fuera de banda desplomado a ≈ −125 dB |

Ver `media/espectro_on.png` y `media/espectro_off.png`.

---

## Discusión: límite físico del filtrado

La mejora por estrechamiento del filtro **no es ilimitada**. El ancho de paso solo puede reducirse hasta el ancho de banda de la propia señal:

- Señal de banda estrecha (tono, como en este banco de pruebas) → margen de mejora amplio.
- Señal de banda ancha (Wi-Fi real ocupa ~20 MHz) → filtrar por debajo de su ancho **corta la señal** (distorsión, ISI, pérdida de datos).

El óptimo teórico es el **filtro adaptado** (*matched filter*), ajustado exactamente al ancho de banda de la señal. Por ello, en sistemas reales el filtrado se combina con otras técnicas: selección de canal, espectro ensanchado, codificación de canal y antenas directivas.

---

## Troubleshooting

**`ValueError: port number 0 exceeds max of (none)` al conectar el Probe.**
El bloque `Probe Avg Mag²` en variante *Complex* (`probe_avg_mag_sqrd_c`) es un sumidero puro: no tiene puerto de salida streaming (se consulta por código con `.level()`). Solución: usar la variante **Complex → Float** (`probe_avg_mag_sqrd_cf`), que expone una salida float conectable al *Number Sink*.

---

## Estructura del repositorio

```
.
├── README.md
├── src/
│   └── Proyecto1.py          # Flowgraph generado (Python)
├── grc/
│   └── Proyecto1.grc         # Flowgraph editable (GNU Radio Companion)
└── media/
    ├── diagrama_bloques.svg  # Arquitectura del sistema
    ├── flowgraph_grc.png     # Captura del flowgraph en GRC
    ├── espectro_on.png       # Espectros con señal (Amplitude=1)
    └── espectro_off.png      # Espectros solo ruido (Amplitude=0)
```

## Ejecución

```bash
# Desde GNU Radio Companion: abrir grc/Proyecto1.grc → Generate (F5) → Run (F6)
# O directamente:
python3 src/Proyecto1.py
```

Para reproducir la medición de SNR: ejecutar con `Amplitude = 1`, anotar potencias; ejecutar con `Amplitude = 0`, anotar potencias; aplicar las fórmulas de la sección Resultados.
