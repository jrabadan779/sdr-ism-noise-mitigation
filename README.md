# Mitigación de ruido en banda ISM 2.4 GHz con SDR y GNU Radio

Reconstrucción de un proyecto de laboratorio en el que caracterizo cómo se degrada una señal en la banda de 2.4 GHz (la del Wi-Fi, Bluetooth, microondas...) cuando aparece ruido, y aplico un filtro FIR digital para recuperar relación señal-ruido. En las medidas que tomé, el filtro me da una mejora de SNR de **12.44 dB**.

Un apunte sobre el montaje, para ser claro: el front-end pensado para esto es un HackRF One, pero las medidas de este repositorio las hice sobre un banco de pruebas en **simulación**, usando el bloque *Channel Model* de GNU Radio para inyectar ruido AWGN. Lo monté así para que el experimento sea reproducible sin depender del espectro real ni de tener el equipo delante; la cadena de procesado (ganancias en banda base + FIR) es la misma que aplicaría sobre una fuente HackRF real.

## Qué hace, en corto

- Genera una señal en banda base y le añade ruido, simulando un canal ISM 2.4 GHz contaminado.
- Le pasa un filtro FIR paso-bajo que deja pasar la señal y recorta el ruido de las zonas del espectro donde no hay nada útil.
- Mide la potencia antes y después de filtrar para calcular cuánto sube la SNR.

## Cómo lo monté

![Diagrama de bloques del sistema](media/diagrama_bloques.svg)

La lógica del flowgraph es sencilla: genero la señal, la ensucio con ruido, y a partir de ahí parto la cadena en dos ramas. Una la dejo intacta (mi referencia, el "antes") y la otra pasa por el filtro (el "después"). Cada rama tiene su propio visor de espectro y su medidor de potencia, así puedo comparar las dos en la misma ejecución sin desincronizarme.

Trabajo con muestreo a 20 MS/s y una ventana de análisis de 20 MHz.

## El filtro: por qué estos números

El filtro lo genero con `firdes.low_pass`:

```python
firdes.low_pass(1, 20e6, 600e3, 200e3, window.WIN_HAMMING, 6.76)
```

La clave que tardé en interiorizar es que la mejora de SNR no viene de "amplificar" la señal, sino de **reducir el ancho de banda por el que entra ruido**, dejando la señal intacta. La relación es:

```
Mejora_SNR (dB) = 10 · log10( BW_ruido / BW_filtro )
```

Como quería apuntar a unos 12 dB, despejé el ancho de banda de paso que necesitaba:

```
BW_filtro = 20 MHz / 10^(12/10) = 20 MHz / 15.85 ≈ 1.26 MHz
```

Por eso puse el corte en 600 kHz (ancho de paso bilateral de ~1.2 MHz). El ancho de transición (200 kHz) fija de paso la longitud del filtro: con ventana Hamming salen del orden de `3.3 · fs / Δf ≈ 330` coeficientes. Estrechar más la transición daría un filtro más selectivo pero con más carga de cálculo y más retardo, así que 200 kHz me pareció un compromiso razonable.

## Cómo medí la SNR

Aquí hay un detalle que se pasa por alto: el medidor de potencia (*Probe Avg Mag²*) mide potencia **total**, señal más ruido mezclados. Para separar las dos componentes ejecuté el flowgraph dos veces:

- Con la señal encendida (`Amplitude = 1`) mido la potencia total.
- Con la señal apagada (`Amplitude = 0`) mido solo el ruido.

Restando (ON − OFF) saco la potencia de señal limpia, y de ahí la SNR de cada rama.

## Resultados

Estas son las cuatro lecturas de potencia que anoté:

| Medición | Baseline | Filtrada |
|---|---|---|
| Señal ON (`Amplitude = 1`) | 1.088968 | 1.005122 |
| Solo ruido (`Amplitude = 0`) | 0.088417 | 0.005031 |

Y el cálculo:

```
P_señal_baseline = 1.088968 − 0.088417 = 1.000551
P_señal_filtrada = 1.005122 − 0.005031 = 1.000091

SNR_antes   = 10·log10(1.000551 / 0.088417) = 10.54 dB
SNR_después = 10·log10(1.000091 / 0.005031) = 22.98 dB

Mejora = 22.98 − 10.54 = 12.44 dB
```

Lo que más me convenció de que la medida era buena: la señal casi no se toca al filtrar (pasa de 1.000551 a 1.000091, una pérdida de inserción de ~0.002 dB), así que la mejora viene íntegra de haber quitado ruido, no de deformar la señal. Además, la reducción de ruido medida directamente, `10·log10(0.088417/0.005031) = 12.45 dB`, coincide con la predicción teórica de arriba.

En el espectro se ve claramente. Con la señal encendida, el suelo de ruido fuera de banda se desploma unos 70 dB tras el filtro:

![Espectros con señal encendida](media/espectro_on.png)

Y con la señal apagada se ve el ruido puro antes y después, que es de donde salen las lecturas OFF:

![Espectros solo con ruido](media/espectro_off.png)

## Un problema que me encontré

Al conectar el medidor de potencia me saltaba este error al ejecutar:

```
ValueError: port number 0 exceeds max of (none)
```

Me costó un rato darme cuenta: el bloque `Probe Avg Mag²` en su variante *Complex* (`probe_avg_mag_sqrd_c`) no tiene salida por streaming, es un sumidero que se consulta por código con `.level()`. Por eso GNU Radio se quejaba de que intentaba conectar un puerto que no existe. La solución fue usar la variante **Complex → Float** (`probe_avg_mag_sqrd_cf`), que sí expone una salida float conectable al display numérico.

## Hasta dónde llega el filtro

Una conclusión importante del experimento: no puedes estrechar el filtro indefinidamente para ganar más SNR. El ancho de paso solo se puede reducir hasta el ancho de banda de la propia señal. En mi banco de pruebas la señal es prácticamente un tono (muy estrecha), así que tengo mucho margen; pero un Wi-Fi real ocupa ~20 MHz, y si filtrara por debajo de eso empezaría a recortar la señal misma (distorsión, pérdida de datos). El óptimo teórico sería un filtro adaptado al ancho exacto de la señal. Por eso en la práctica el filtrado se combina con otras técnicas: cambiar de canal, espectro ensanchado, codificación, antenas directivas.

## Estructura del repositorio

```
.
├── README.md
├── src/Proyecto1.py          # Flowgraph generado (Python)
├── grc/Proyecto1.grc         # Flowgraph editable (GNU Radio Companion)
└── media/
    ├── diagrama_bloques.svg
    ├── espectro_on.png        # Espectros con señal (Amplitude=1)
    └── espectro_off.png       # Espectros solo ruido (Amplitude=0)
```

## Cómo ejecutarlo

Desde GNU Radio Companion: abrir `grc/Proyecto1.grc`, Generate (F5), Run (F6). O directamente:

```bash
python3 src/Proyecto1.py
```

Para reproducir las medidas: ejecutar con `Amplitude = 1` y anotar las dos potencias, luego con `Amplitude = 0` y anotar las otras dos, y aplicar las fórmulas de la sección de resultados.
