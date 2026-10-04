# Interferencia en modo común y topología de apantallado de los electrodos
## Análisis teórico, modelo comportamental y protocolo de validación
### Front-end ECG basado en ADS1292 + ESP32-S3 (TFM)

---

## Índice

1. Contexto y objetivo
2. Fundamentos: tensión diferencial y tensión en modo común
3. Origen físico de la interferencia en modo común
4. Mecanismos de conversión modo común → diferencial
5. El lazo RLD (Right-Leg Drive) como realimentación negativa del modo común
6. El problema del apantallado: la malla no es gratis
7. Modelo equivalente y ecuaciones de nodo
8. Taxonomía de topologías de apantallado
9. Resultados cuantitativos
10. Modos de fallo y estabilidad
11. Decisión de diseño e implementación en la PCB
12. Protocolo de validación experimental
13. Errores conceptuales frecuentes
14. Hipótesis, limitaciones y trabajo futuro
15. Glosario
16. Referencias

---

## 1. Contexto y objetivo

El sistema adquiere ECG con un AFE **ADS1292** (2 canales, ΔΣ de 24 bits, con amplificador RLD integrado) controlado por un **ESP32-S3**. Los electrodos (Ag/AgCl desechables con gel) se conectan mediante cables apantallados. Cada conector de electrodo dispone de un pin adicional para la malla.

La cuestión de diseño es: **¿a qué potencial debe conectarse la malla de los cables de electrodo?**

Opciones consideradas: masa analógica (GNDA), nodo del electrodo RLD (INRLD), salida del amplificador RLD (RLDOUT), un buffer activo del modo común, o dejarla flotante.

El objetivo del documento es justificar la elección con un modelo físico cuantitativo, y definir un experimento que lo valide sobre la PCB.

---

## 2. Fundamentos: tensión diferencial y tensión en modo común

Sean $V_P$ y $V_N$ las tensiones de las dos entradas de un canal, medidas respecto a la masa del circuito (GNDA). Se definen:

$$
V_{DM} = V_P - V_N \qquad\qquad V_{CM} = \frac{V_P + V_N}{2}
$$

y recíprocamente:

$$
V_P = V_{CM} + \tfrac{1}{2}V_{DM}, \qquad V_N = V_{CM} - \tfrac{1}{2}V_{DM}
$$

| Magnitud | Significado físico en ECG | Orden de magnitud |
|---|---|---|
| $V_{DM}$ | Proyección del dipolo cardíaco sobre la derivación: **la señal útil** | 0.1–3 mV |
| $V_{CM}$ | Potencial del cuerpo entero respecto a la masa del equipo | µV a V (sin RLD) |

Un amplificador diferencial ideal responde solo a $V_{DM}$. Uno real tiene una salida:

$$
V_{out} = A_{DM}\,V_{DM} + A_{CM}\,V_{CM}, \qquad \text{CMRR} = \frac{A_{DM}}{A_{CM}}
$$

**Nota terminológica.** En sentido estricto no es *ruido* (proceso aleatorio) sino **interferencia**: una señal determinista a 50 Hz y sus armónicos. En la memoria conviene usar "interferencia en modo común".

---

## 3. Origen físico de la interferencia en modo común

### 3.1 Acoplamiento capacitivo con la red eléctrica

```
   Red 230 V~ (cableado, lámparas, enchufes)
        ║
        ║  C_red ≈ 1–3 pF
        ▼  I_d = V_red · jω · C_red        (corriente de desplazamiento)
   ┌──────────────────┐
   │      CUERPO      │   resistencia interna ~ 100 Ω
   │   (conductor     │   ≪ |Z_C| ~ MΩ   ⇒  equipotencial
   │    volumétrico)  │
   └────────┬─────────┘
            ║  C_cuerpo ≈ 100–200 pF  (a tierra / a GNDA)
           ═╩═
```

La corriente de desplazamiento vale:

$$
I_d = V_{red}\,\omega\,C_{red} = 230\,\text{V}\cdot 2\pi\cdot 50\,\text{Hz}\cdot 2\,\text{pF} \approx 145\,\text{nA}_{rms}
$$

### 3.2 Por qué es "común"

La corriente entra en el cuerpo a través de impedancias capacitivas del orden de MΩ, mientras que la resistencia interna del cuerpo es de cientos de ohmios. La caída de tensión dentro del cuerpo es por tanto despreciable: **todo el cuerpo sube y baja a 50 Hz en bloque**, y todos los electrodos ven la misma tensión. Sin ninguna medida correctora:

$$
V_{CM} \approx \frac{I_d}{\omega\,C_{cuerpo}} = \frac{145\,\text{nA}}{2\pi\cdot 50\cdot 200\,\text{pF}} \approx 2.3\,\text{V}
$$

Es decir, el modo común puede ser **60–80 dB mayor que el ECG**.

### 3.3 Otras fuentes de modo común

- Deriva DC del potencial corporal y potenciales de semicelda de los electrodos.
- Interferencias de RF, que se demodulan en las no linealidades de la entrada (motivo del filtro RC de entrada de 22 kΩ + 470 pF).
- Descargas electrostáticas y movimiento de cables (efecto triboeléctrico).

---

## 4. Mecanismos de conversión modo común → diferencial

Si el amplificador solo mide la diferencia, ¿por qué molesta el modo común? Porque existen mecanismos que **convierten** parte de $V_{CM}$ en $V_{DM}$, y esa parte ya es indistinguible del ECG.

| # | Mecanismo | Descripción | Relevancia en este diseño |
|---|---|---|---|
| 1 | CMRR finito del AFE | Desajustes internos del PGA del ADS1292 | Baja: el ADS1292 especifica ~120 dB |
| 2 | **Divisor de potencial desequilibrado** | Cada entrada es un divisor entre la impedancia del electrodo $Z_e$ y la impedancia a un nodo de referencia (capacidad del cable, capacidad de entrada). Si $Z_{e1} \neq Z_{e2}$, las dos entradas se atenúan distinto | **Dominante** |
| 3 | Rango de modo común | $V_{CM}$ debe permanecer entre AVSS + 0.3 V y AVDD − 0.3 V; si no, el PGA satura | El RLD fija el cuerpo a medio rail en DC |

### 4.1 Expresión del mecanismo 2

#### 4.1.1 Dos elementos distintos: impedancia del electrodo y capacidad parásita

| | Impedancia del electrodo $Z_e$ | Capacidad parásita $C$ |
|---|---|---|
| Qué es | Interfaz piel–gel–electrodo (capas de la piel, gel electrolítico, interfaz Ag/AgCl) | Capacidad entre el conductor del cable y la malla que lo rodea |
| Posición en el circuito | **En serie** en el camino de la señal, entre el cuerpo y el hilo | **En derivación**, del hilo hacia la malla |
| Valor típico a 50 Hz | 10–50 kΩ | 100 pF, es decir, $\lvert Z_C\rvert = 1/(\omega C) \approx 32$ MΩ |
| Diferencia entre electrodos | Grande: preparación de la piel, cantidad de gel, presión, secado | Pequeña: mismo tipo y longitud de cable |

```
   CUERPO (V_CM)
       │
      [Z_e]      ← en SERIE: la señal tiene que atravesarla
       │
       ●──────────────► entrada del ADS1292   (nodo "hilo", tensión V_hilo)
       │
      ═╪═ C      ← en DERIVACIÓN: fuga del hilo hacia la malla
       │
     MALLA (V_malla)
```

Sin $C$, por $Z_e$ no circularía corriente (la entrada del ADS1292 tiene impedancia muy alta), no habría caída en $Z_e$ y el hilo estaría exactamente a $V_{CM}$, fuera cual fuera el valor de $Z_e$. Con $C$ se forma un **divisor de tensión** ($Z_e$ arriba, $C$ abajo) y el hilo deja de estar exactamente a $V_{CM}$. **Ninguno de los dos elementos causa el error por separado: lo causa su combinación.**

#### 4.1.2 Intuición en dos pasos

La corriente en la rama cuerpo → $Z_e$ → $C$ → malla es, de forma exacta:

$$
I = \frac{V_{CM} - V_{malla}}{Z_e + \frac{1}{j\omega C}} = \frac{\Delta V}{Z_e + Z_C}, \qquad \Delta V \equiv V_{CM} - V_{malla}
$$

Como $Z_e \approx 20$ kΩ es el 0.06 % de $\lvert Z_C\rvert \approx 31.8$ MΩ, puede despreciarse **en el denominador**:

$$
I \approx j\omega C\,\Delta V
$$

1. **La capacidad fija cuánta corriente circula**, porque es la impedancia dominante de la rama (igual que en una serie de 1 Ω y 1 MΩ la corriente la fija la de 1 MΩ).
2. **El electrodo convierte esa corriente en una tensión de error**, porque está en serie justo antes del punto que mide el ADS: $V_{hilo} = V_{CM} - I\,Z_e$.

$$
\text{error en un hilo} \approx -\,\Delta V\cdot j\omega\,C\,Z_e
$$

#### 4.1.3 Derivación formal (divisor exacto y aproximación de primer orden)

Tomando la malla como referencia, el divisor exacto es:

$$
V_{hilo} - V_{malla} = \Delta V\cdot\frac{Z_C}{Z_e + Z_C} = \frac{\Delta V}{1 + j\omega C Z_e}
$$

$$
V_{hilo} = V_{malla} + \frac{\Delta V}{1 + j\omega C Z_e} \qquad\text{(exacta)}
$$

Con $x = j\omega C Z_e$ y $|x| = 2\pi\cdot 50\cdot 100\,\text{pF}\cdot 20\,\text{k}\Omega \approx 6.3\cdot10^{-4} \ll 1$, se aplica el desarrollo de Taylor $\frac{1}{1+x} \approx 1 - x$:

$$
V_{hilo} \approx \underbrace{V_{malla} + \Delta V}_{=\,V_{CM}} - \Delta V\cdot j\omega C Z_e
$$

Para las dos entradas (electrodo 1 en P, electrodo 2 en N):

$$
V_P \approx V_{CM} - \Delta V\cdot j\omega\,C_1 Z_1, \qquad V_N \approx V_{CM} - \Delta V\cdot j\omega\,C_2 Z_2
$$

Al restar, $V_{CM}$ se cancela (función del amplificador diferencial), pero los errores no, porque no son iguales:

$$
\boxed{V_{DM}^{error} = V_P - V_N \approx \Delta V\cdot j\omega\,(Z_2 C_2 - Z_1 C_1)}
$$

Si el circuito fuera perfectamente simétrico ($Z_1 C_1 = Z_2 C_2$), los errores también se cancelarían: **el problema es la asimetría, no la existencia de los errores.**

Con el mismo cable en ambos electrodos ($C_1 = C_2 = C$):

$$
\boxed{V_{DM}^{error} \approx \Delta V \cdot j\omega C \cdot \Delta Z}, \qquad |V_{DM}^{error}| \approx |\Delta V|\cdot\omega C\cdot|\Delta Z|
$$

El factor $j$ indica únicamente que el error está desfasado 90° respecto a $\Delta V$ (corriente capacitiva).

#### 4.1.4 Ejemplo numérico

Malla a masa ($\Delta V = V_{CM}$), $V_{CM} = 10$ mV, $C = 100$ pF, $Z_1 = 20$ kΩ, $Z_2 = 40$ kΩ:

| Magnitud | Cálculo | Valor |
|---|---|---|
| $\omega C$ | $2\pi\cdot 50\cdot 100\,\text{pF}$ | 31.4 nS |
| Error en P | $10\,\text{mV}\cdot 31.4\,\text{nS}\cdot 20\,\text{k}\Omega$ | 6.3 µV |
| Error en N | $10\,\text{mV}\cdot 31.4\,\text{nS}\cdot 40\,\text{k}\Omega$ | 12.6 µV |
| **Diferencia** (lo que ve el ADS) | 12.6 − 6.3 | **6.3 µV** |

#### 4.1.5 Simplificaciones de este paso

- Las resistencias de 22 kΩ de la PCB no intervienen: la corriente por $C$ va del hilo a la malla dentro del cable y no las atraviesa.
- La capacidad de entrada del ADS1292 y de las pistas ($C_{in}$) repite el mismo mecanismo con una capacidad unas 10 veces menor; se incluye en el modelo completo (§7).

#### 4.1.6 Conclusión

La expresión $V_{DM}^{error} \approx \Delta V \cdot j\omega C\,\Delta Z$ es la base de todo el análisis: **la topología de apantallado determina $\Delta V$** (§6–§8) y **los electrodos determinan $\Delta Z$**.

### 4.2 CMRR efectivo del sistema

Si la malla está a GNDA, $\Delta V = V_{CM}$ y el mecanismo 2 actúa como un "CMRR por desequilibrio":

$$
\text{CMRR}_{deseq} = \frac{1}{\omega\,C_s\,\Delta Z}
$$

Con $C_s = 100$ pF y $\Delta Z = 20$ kΩ a 50 Hz:

$$
\text{CMRR}_{deseq} = \frac{1}{2\pi\cdot 50\cdot 100\,\text{pF}\cdot 20\,\text{k}\Omega} \approx 1590 \;\Rightarrow\; \approx 64\,\text{dB}
$$

El CMRR del sistema completo combina ambos términos:

$$
\frac{1}{\text{CMRR}_{sist}} \approx \frac{1}{\text{CMRR}_{AFE}} + \frac{1}{\text{CMRR}_{deseq}}
$$

**Conclusión importante:** aunque el ADS1292 tenga ~120 dB, el sistema real queda limitado a ~60 dB **por el cable y los electrodos**, no por el integrado. Por eso la topología de apantallado es un problema de primer orden.

---

## 5. El lazo RLD como realimentación negativa del modo común

### 5.1 Estructura

```
 IN1P ──200k──┐  (resistencias internas, selección por RLD_SENS)
 IN1N ──200k──┼──► RLDINV ──[ −G(s) ]──► RLDOUT ──R_s = 390k──► INRLD ──Z_RLD──► CUERPO
              │    (promedio = V_CM)           ▲                                     │
              │                          R_f=1M ∥ C_f=1.5nF                          │
              └──────────────────────────────── V_CM ◄───────────────────────────────┘
```

El amplificador RLD mide el modo común (promedio de las entradas seleccionadas), lo invierte, lo amplifica y lo **inyecta de vuelta al cuerpo** a través del electrodo RLD. Es una realimentación negativa sobre $V_{CM}$.

### 5.2 Ganancia del lazo

Con N entradas seleccionadas en RLD_SENS, las resistencias internas de sensado quedan en paralelo: $R_{in} = 200\,\text{k}\Omega / N$.

$$
G(s) = \frac{R_f / R_{in}}{1 + s R_f C_f}, \qquad G_0 = \frac{1\,\text{M}\Omega}{100\,\text{k}\Omega} = 10 \;(N=2)
$$

$$
f_p = \frac{1}{2\pi R_f C_f} = \frac{1}{2\pi\cdot 1\,\text{M}\Omega\cdot 1.5\,\text{nF}} \approx 106\,\text{Hz}
$$

$$
|G(50\,\text{Hz})| = \frac{10}{\sqrt{1 + (50/106)^2}} \approx 9.0
$$

Frecuencia de cruce aproximada del lazo: $f_c \approx f_p\sqrt{G_0^2 - 1} \approx 1.05$ kHz.

### 5.3 Atenuación del modo común

Balance de corrientes en el cuerpo (despreciando $C_{cuerpo}$ frente al camino del RLD):

$$
I_d = \frac{V_{B} - V_{RLDOUT}}{R_s + Z_{RLD}}, \qquad V_{RLDOUT} = -G\,V_B
$$

$$
\boxed{V_B = \frac{I_d\,(R_s + Z_{RLD})}{1 + G}}
$$

Con $I_d = 145$ nA, $R_s = 390$ kΩ, $Z_{RLD} = 20$ kΩ y $G = 9$: $V_B \approx 5.9$ mV (frente a ~2.3 V sin RLD).

**El RLD no elimina el modo común: lo atenúa en un factor $(1+G)$.** Además fija el nivel DC del cuerpo a medio rail (mecanismo 3).

### 5.4 Por qué $R_s$ no se puede reducir libremente

$R_s = 390$ kΩ limita la corriente que puede circular por el paciente en caso de fallo (salida del amplificador saturada a un rail):

$$
I_{max} \approx \frac{3.3\,\text{V}}{390\,\text{k}\Omega} \approx 8.5\,\mu\text{A}
$$

Es un requisito de **seguridad del paciente**, por lo que $R_s$ es una restricción de diseño, no un parámetro libre.

---

## 6. El problema del apantallado: la malla no es gratis

### 6.1 Dos mecanismos de captación

Existen dos caminos por los que la red eléctrica introduce interferencia en los cables. La malla **cambia uno por otro**:

| Mecanismo | Qué ocurre | Sin malla | Con malla |
|---|---|---|---|
| **A. Captación directa en el conductor** | La red induce $I_l = V_{red}\,j\omega C_{hilo}$ en cada conductor; atraviesa $Z_e$ y produce $I_l\,\Delta Z$ diferencial | Dominante | Eliminado: la malla intercepta la corriente |
| **B. Divisor $Z_e$–$C_s$** | La tensión entre cuerpo y malla carga $C_s$ (~50–100 pF/m) a través de $Z_e$ | No existe | **Aparece** |

### 6.2 Caminos de corriente: sin malla frente a malla a GND

```
 A · SIN MALLA
 ═══════════════════════ RED 230 V (cableado, lámparas) ═════════════════════
      │                                 │
     ═╪═ C_red                         ═╪═ C_hilo ≈ 0.2 pF
      │                                 │  (1)
 ┌────┴─────┐                           │
 │  CUERPO  ├───[ Z_e ]─────────────────●─────────────────────────────► entrada ADS
 │   V_CM   │   ◄── (1)
 └──────────┘
 (1) red → C_hilo → hilo → Z_e → cuerpo          ≈ 14 nA    atraviesa Z_e → error GRANDE


 B · MALLA A GND (T1)
 ═══════════════════════ RED 230 V (cableado, lámparas) ═════════════════════
      │                                         │
     ═╪═ C_red                                 ═╪═ C_ext
      │                                         │  (2)
 ┌────┴─────┐            ┌ ─ ─  MALLA  ─ ─ ─ ─ ─┴─ ─ ─ ─ ┐
 │  CUERPO  ├───[ Z_e ]──┼─────────────●─────────────────┼──────────────► entrada ADS
 │   V_CM   │   (1) ──►  ¦             │                 ¦
 └──────────┘            ¦            ═╪═ C_s ≈ 100 pF   ¦
                         ¦             │  (1)            ¦
                         └ ─ ─ ─ ─ ─ ─ ┴ ─ ─ ─ ─ ─ ─ ─ ─ ┘
                                       │
                                      GND
 (1) cuerpo → Z_e → hilo → C_s → malla → GND     ≈ 0.3 nA   atraviesa Z_e → error pequeño
 (2) red → C_ext → malla → GND                               no toca el hilo → inofensiva
```

**Lectura del panel B.** La corriente que atraviesa $Z_e$ **no procede de la malla**. Circula del punto de mayor potencial al de menor: el cuerpo está a $V_{CM}$ (elevado por la red a través de $C_{red}$) y la malla a 0 V. Entre ambos solo hay un camino: **cuerpo → $Z_e$ → hilo → $C_s$ → malla → GND**. La malla es el sumidero, no la fuente, y la corriente atraviesa $Z_e$ porque el cuerpo está al otro lado de $Z_e$. El lazo se cierra a través de la propia red, por corrientes de desplazamiento.

### 6.3 Comparación cuantitativa de la corriente que atraviesa el electrodo

| | A. Sin malla | B. Malla a GND |
|---|---|---|
| Tensión que impulsa la corriente | **Red: 230 V** | **Modo común del cuerpo: ~10 mV** (ya atenuado por el RLD) |
| Capacidad por la que entra | $C_{hilo}$ ≈ 0.2 pF (hilo lejos de la red) | $C_s$ ≈ 100 pF (malla pegada al hilo) |
| Corriente por $Z_e$ | $230\cdot 2\pi\cdot 50\cdot 0.2\,\text{pF} \approx$ **14 nA** | $0.01\cdot 2\pi\cdot 50\cdot 100\,\text{pF} \approx$ **0.3 nA** |

La capacidad es unas 500 veces mayor con malla, pero la tensión que la excita es unas 23 000 veces menor. El resultado es **~46 veces menos corriente** por el electrodo (~33 dB), y como el error es esa corriente por $Z_e$, el error baja en la misma proporción. Sin malla, además, la captación depende del recorrido de cada cable (cercanía a enchufes y lámparas), lo que añade asimetría entre los dos hilos.

**Conclusión: sin malla el sistema funcionaría mucho peor.**

### 6.4 Función de la malla

La malla actúa como un **paraguas**: se interpone entre la red y el hilo, recoge la corriente de desplazamiento de la red (por $C_{ext}$) y la conduce a un potencial conocido **sin que atraviese $Z_e$**. El precio es que, al estar pegada al hilo, crea $C_s$, por la que circula una corriente pequeña impulsada por $\Delta V = V_{CM} - V_{malla}$.

| Función | Efecto |
|---|---|
| Interceptar el campo eléctrico de la red | Elimina la fuente grande (230 V) del entorno del hilo |
| Fijar el entorno del hilo a un potencial conocido | Sustituye un acoplamiento desconocido y variable por uno conocido ($C_s$) |
| **Precio:** $C_s$ entre hilo y malla | Pequeña conversión de modo común, proporcional a $\Delta V$ |

### 6.5 Consecuencia: el potencial de la malla es la variable de diseño

| Situación | Lo que impulsa corriente por $Z_e$ | Resultado |
|---|---|---|
| Sin malla | La red (230 V) a través de $C_{hilo}$ | Muy malo |
| Malla a GND (T1) | $V_{CM}$ completo a través de $C_s$ | Bueno |
| Malla a INRLD (T2) | Solo $I_d \cdot Z_{RLD}$ a través de $C_s$ (§7) | Mejor, si el electrodo RLD tiene buen contacto |
| Malla activa (T3) | ≈ 0 | Óptimo |

> **Elegir la topología de apantallado es elegir a qué potencial se pone la malla para minimizar $\Delta V_{C_s}$ (tensión entre la malla y el conductor que protege), sin introducir un modo de fallo peor.**

El potencial ideal para la malla es **el del propio conductor**, que es aproximadamente el del cuerpo. Con ello $\Delta V_{C_s} \to 0$ y la capacidad del cable deja de convertir modo común en diferencial. Este es el principio del *guarding* o apantallado activo.

---

## 7. Modelo equivalente y ecuaciones de nodo

### 7.1 Circuito

```
   Red 230 V~
     ║ C_red (2 pF)           ║ C_ext (1 pF por malla)        ║ C_hilo (solo sin malla)
     ▼ I_d                    ▼ I_sh                           ▼ I_l
 ┌─────────┐              ┌─────────┐
 │ CUERPO  │──── Z_RLD ───│ INRLD N │──── R_s 390k ────◄ RLDOUT = −G · V_B
 │   V_B   │              └─────────┘
 │         │── Z_e1 ──┬── cable ── 22k ──► IN1P ─┐
 │         │         C_s ═ malla                  C_in (ADS + PCB, ~10 pF)
 │         │── Z_e2 ──┬── cable ── 22k ──► IN1N ─┘
 └─────────┘         C_s ═ malla
      ║ C_cuerpo (200 pF)
     GNDA
```

### 7.2 Resolución para la malla conectada a INRLD

Leyes de Kirchhoff de corrientes (KCL), despreciando $C_{cuerpo}$ y la corriente por $C_s$ frente a la del RLD:

**Nodo B (cuerpo):** toda la corriente de desplazamiento del cuerpo sale por el electrodo RLD:

$$
I_d = \frac{V_B - V_N}{Z_{RLD}} \;\Rightarrow\; V_N = V_B - I_d\,Z_{RLD}
$$

**Nodo N (INRLD):** entran la corriente del cuerpo y la captada por la malla; salen por $R_s$ hacia RLDOUT:

$$
I_d + I_{sh} = \frac{V_N - V_{RLDOUT}}{R_s} = \frac{V_N + G\,V_B}{R_s}
$$

Combinando:

$$
V_B = \frac{I_d\,(R_s + Z_{RLD}) + I_{sh}\,R_s}{1+G}, \qquad \boxed{V_B - V_N = I_d\,Z_{RLD}}
$$

### 7.3 Interpretación

1. La tensión entre conductor y malla ($V_B - V_N$) **solo depende de la corriente de desplazamiento del cuerpo y de la impedancia del electrodo RLD**.
2. La corriente que capta la cara exterior de la malla ($I_{sh}$) **no atraviesa el electrodo RLD**: sale por $R_s$ hacia la salida del amplificador. Su único efecto es elevar $V_B$ (modo común del cuerpo), que solo se convierte a diferencial a través de $C_{in}$, que es pequeña.

### 7.4 Comparación con la malla a GNDA

| Malla en | $\Delta V_{C_s}$ |
|---|---|
| GNDA | $V_B = \dfrac{I_d\,(R_s + Z_{RLD})}{1+G}$ |
| INRLD | $I_d\,Z_{RLD}$ |

$$
\frac{\Delta V_{C_s}^{INRLD}}{\Delta V_{C_s}^{GNDA}} = \frac{Z_{RLD}\,(1+G)}{R_s + Z_{RLD}} < 1 \;\Longleftrightarrow\; \boxed{Z_{RLD} < \frac{R_s}{G}}
$$

Con $R_s = 390$ kΩ y $|G(50\,\text{Hz})| \approx 9$: **$Z_{RLD} \lesssim 43$ kΩ**.

**Regla de diseño:** conectar las mallas a INRLD reduce la tensión sobre la capacidad del cable mientras la impedancia del electrodo RLD a 50 Hz sea menor que $R_s/G$. Esta condición se cumple con electrodos Ag/AgCl con gel y piel preparada.

---

## 8. Taxonomía de topologías de apantallado

| Id | Topología | Potencial de la malla | $\Delta V_{C_s}$ | Destino de $I_{sh}$ | Coste |
|---|---|---|---|---|---|
| T0 | Malla flotante / sin malla | Indefinido | — | Acoplada al conductor | 0 |
| T1 | Pasiva a GNDA (un solo extremo) | 0 | $V_B$ | GNDA (impedancia nula) | 0 |
| T2 | Al nodo del electrodo RLD (INRLD) | $V_B - I_d Z_{RLD}$ | $I_d Z_{RLD}$ | RLDOUT, a través de $R_s$ | 0 |
| T3 | Activa: buffer del modo común | ≈ $V_B$ | ≈ 0 | Salida del buffer | 1 op-amp + 4 pasivos |
| T4 | Guard individual por conductor | Sigue a cada conductor | ≈ 0 | Salida de cada buffer | 1 op-amp por conductor |
| T5 | A la salida del RLD (RLDOUT) | $-G\,V_B$ | $(1+G)\,V_B$ | Salida del RLD | 0 |

### 8.1 Por qué T5 (malla a RLDOUT) es la peor opción

RLDOUT es la salida del amplificador RLD: vale $-G\,V_B$, en **contrafase** con el cuerpo y amplificada. La tensión sobre la capacidad del cable es $(1+G)\,V_B$, unas diez veces mayor que con la malla a GNDA. Además, carga directamente la salida de un amplificador que forma parte de un lazo de realimentación con varios cientos de pF, con riesgo para su estabilidad.

### 8.2 Diferencia clave entre T2 y T5

Ambas conectan la malla "al RLD", pero en nodos distintos separados por $R_s$:

- **RLDOUT (T5):** salida del amplificador, en contrafase con el cuerpo. **Empeora.**
- **INRLD (T2):** nodo del electrodo, al otro lado de $R_s$. En lazo cerrado sigue aproximadamente al cuerpo. **Mejora** si $Z_{RLD} < R_s/G$.

### 8.3 T3: apantallado activo

Un buffer de ganancia ≈ 1 reproduce el modo común medido en las entradas (divisor de alta impedancia, ≥ 10 MΩ, tomado después de las resistencias de 22 kΩ) y excita las mallas a través de una resistencia de aislamiento $R_{iso}$ de 100 Ω–1 kΩ.

- Reduce la capacidad efectiva del cable: $C_{ef} = C_s\,(1 - A_{buf}) \approx 0$.
- Crea un lazo de **realimentación positiva** (malla → $C_s$ → entradas → buffer → malla). Es estable si $A_{buf} < 1$ y $R_{iso}$ aísla la carga capacitiva del cable.
- Requiere un op-amp CMOS con corriente de polarización del orden de pA y estable con carga capacitiva.

Es la mejor opción técnica, pero su beneficio solo es relevante con electrodos de alta impedancia (secos o textiles).

---

## 9. Resultados cuantitativos

### 9.1 Parámetros del modelo

| Parámetro | Valor | Justificación |
|---|---|---|
| $V_{red}$ | 230 V rms, 50 Hz | Red europea |
| $C_{red}$ | 2 pF | Acoplamiento cuerpo–red típico en interior |
| $C_{ext}$ | 1 pF por malla, 4 mallas | Captación de la cara exterior de cada cable (~1 m) |
| $C_{hilo}$ | 0.2 pF por hilo | Solo para T0 |
| $C_s$ | 100 pF | Capacidad conductor–malla, ~1 m de cable |
| $C_{in}$ | 10 pF | Entrada del ADS1292 + pistas |
| $Z_{e1}$, $Z_{e2}$ | 20 kΩ, 40 kΩ ($\Delta Z$ = 20 kΩ) | Ag/AgCl con gel a 50 Hz, desbalance moderado |
| $R_s$ | 390 kΩ | Esquemático |
| $G_0$, $f_p$ | 10, 106 Hz | $R_f$ = 1 MΩ, $C_f$ = 1.5 nF, RLD_SENS con 2 entradas |

Corrientes resultantes: $I_d \approx 145$ nA, $I_{sh} \approx 289$ nA.

### 9.2 Amplitud de la interferencia diferencial a 50 Hz ($Z_{RLD}$ = 20 kΩ)

| Topología | $V_{DM}$ (50 Hz) | Respecto a T1 |
|---|---|---|
| T0 Sin malla | ~290 µV | +37 dB |
| T1 GNDA | 4.1 µV | 0 dB (referencia) |
| **T2 INRLD** | **2.9 µV** | **−3 dB** |
| T3 Activa | 0.4 µV | −21 dB |
| T5 RLDOUT | 38 µV | +19 dB |

Como referencia, el ruido propio del ADS1292 es del orden de 8 µVpp en 150 Hz de ancho de banda (G = 6).

### 9.3 Sensibilidad de T2 a la impedancia del electrodo RLD

| $Z_{RLD}$ a 50 Hz | Situación típica | T2 respecto a T1 |
|---|---|---|
| 10 kΩ | Ag/AgCl, piel preparada (limpieza + abrasión suave) | −6 dB |
| 20 kΩ | Ag/AgCl, piel limpia | −3 dB |
| 50 kΩ | Ag/AgCl, piel sin preparar | +2 dB |
| 500 kΩ | Electrodo seco o textil | ≈ +16 dB (estimado) |

### 9.4 Lectura de los resultados

1. La ordenación es **T5 > T0 > T1 > T2 > T3** (de peor a mejor).
2. Con electrodos de gel, **T2 es la mejor opción pasiva**, con una mejora moderada (3–6 dB) sobre T1.
3. La mejora de T2 es menor de lo que indicaría $\Delta V_{C_s}$ por sí sola, porque $I_{sh}$ eleva el modo común del cuerpo y este se convierte parcialmente a diferencial por $C_{in}$.
4. Los valores absolutos dependen fuertemente del entorno ($C_{red}$, longitud de cable, $\Delta Z$). **La ordenación relativa y la regla $Z_{RLD} < R_s/G$ son lo robusto del análisis.**

---

## 10. Modos de fallo y estabilidad

### 10.1 Desconexión del electrodo RLD (lead-off)

Si el electrodo RLD se despega ($Z_{RLD} \to \infty$):

- **En cualquier topología:** el cuerpo deja de estar controlado por el lazo y su potencial vuelve a $\approx I_d / (\omega C_{cuerpo})$, del orden de voltios. Puede saturar el rango de modo común del PGA.
- **Además, en T2:** el nodo INRLD queda conectado a RLDOUT solo a través de $R_s$, y la captación de las mallas produce en ellas una tensión respecto al cuerpo de

$$
V_{malla} \approx I_{sh}\,R_s \approx 289\,\text{nA}\cdot 390\,\text{k}\Omega \approx 110\,\text{mV}
$$

que se traduce en unos 70 µV de 50 Hz diferenciales. **El modo de fallo de T2 es peor que el de T1.**

**Mitigación:** activar la detección de lead-off del electrodo RLD en el ADS1292 (bit RLD_LOFF_SENS del registro RLD_SENS; comprobar en el datasheet) y marcar como inválidos los tramos afectados.

### 10.2 Estabilidad del lazo RLD

Conectar las mallas a INRLD añade capacidades a ese nodo:

- La capacidad conductor–malla ($C_s$, en total ~400 pF para 4 cables) va de INRLD a los conductores, cuyo potencial es ≈ el del cuerpo. Queda **en paralelo con $Z_{RLD}$** y crea una singularidad hacia $1/(2\pi Z_{RLD}\,C_{s,tot}) \approx 20$ kHz.
- La capacidad de la malla al entorno ($C_{ext}$ + otras, del orden de decenas de pF) va a un nodo de referencia y forma con $R_s$ un polo hacia $1/(2\pi R_s\,C) \approx 20$ kHz.

Ambas singularidades quedan una década por encima del cruce del lazo (~1 kHz), por lo que el impacto sobre el margen de fase debería ser pequeño. **Debe verificarse** con un análisis en frecuencia del lazo completo (respuesta $V_B/V_{red}$ entre 1 Hz y 100 kHz, buscando picos de resonancia).

> Nota: en una primera estimación informal se había situado este polo cerca de 1 kHz suponiendo toda $C_s$ referida a masa. Al referirla correctamente al cuerpo, el efecto se desplaza a frecuencias mayores.

### 10.3 Seguridad del paciente

La PCB **no tiene aislamiento galvánico**. Si el equipo está conectado por USB a un PC alimentado por la red, existe un camino entre el paciente y tierra. Las adquisiciones sobre personas deben hacerse **solo con batería** o con un aislador USB. En T2 las mallas quedan conectadas a un nodo del paciente, lo que refuerza esta precaución.

---

## 11. Decisión de diseño e implementación en la PCB

### 11.1 Decisión

**Topología por defecto: T2 (mallas a INRLD)**, con T1 como alternativa seleccionable y T3 preparada como opción futura.

### 11.2 Implementación

```
 J20..J24 (mallas) ──► net SHIELD ──┬── R_A  0 Ω   (MONTADA) ──► INRLD      T2 por defecto
                                    ├── R_B  0 Ω   (DNP)     ──► GNDA       T1 alternativa
                                    └── R_iso      (DNP) ◄── U_buf (DNP) ◄── divisor V_CM (DNP)   T3 futuro
```

| Decisión | Justificación |
|---|---|
| T2 por defecto | Mínimo $\Delta V_{C_s}$ entre las opciones pasivas con $Z_{RLD} < R_s/G$; sin componentes adicionales |
| Selección con resistencias de 0 Ω o solder jumpers junto al conector | Un jumper de pines añade un bucle y una antena en un nodo sensible; el 0 Ω en 0603 es compacto y reproducible |
| Nunca montar R_A y R_B a la vez | Cortocircuitaría INRLD a GNDA: el lazo RLD quedaría cargado a masa a través de $R_s$ |
| Malla conectada solo en el extremo de la placa | Conectarla en ambos extremos crea bucles de masa y hace circular corriente por la malla |
| Conector de mallas físicamente distinto al de los electrodos | Evita conectar por error un electrodo a la malla; en T1 fijaría el cuerpo a GNDA contra el RLD, y en T2 crearía un segundo electrodo RLD |
| Detección de lead-off del RLD en firmware | Cubre el modo de fallo propio de T2 |
| Footprint de T3 | Permite migrar a electrodos secos sin rediseñar la placa |

### 11.3 Requisitos del footprint de T3 (si se monta en el futuro)

- Divisor de modo común: 2 × 10 MΩ tomados después de las resistencias de 22 kΩ, para no degradar la impedancia de entrada ni el equilibrio entre canales.
- Buffer: op-amp CMOS de baja corriente de polarización (pA), estable con carga capacitiva, en encapsulado SOT-23-5.
- $R_{iso}$ de 100 Ω–1 kΩ en serie con la salida, para aislar la capacidad de los cables.

---

## 12. Protocolo de validación experimental

### 12.1 Montaje

1. **Fantoma de paciente:** un nodo "cuerpo" con los electrodos conectados a través de resistencias que representen $Z_e$ (por ejemplo 20 kΩ) y la red de desbalance 51 kΩ ∥ 47 nF en serie con uno de ellos. Comprobar en la norma IEC 60601-2-25 la cláusula exacta del ensayo de rechazo al modo común.
2. **Inyección controlada de modo común:** un generador de 50 Hz conectado al nodo "cuerpo" a través de un condensador conocido (por ejemplo 2 pF, o un valor mayor con menor amplitud). Así $I_d$ es conocida y el experimento es repetible.
3. **Alimentación solo con batería** durante la medida.

### 12.2 Procedimiento

1. Para cada configuración (T1, T2 y, si se monta, T3): adquirir 60 s con el ADS1292.
2. Estimar la amplitud de 50 Hz mediante FFT del flujo crudo con ventana de Hann.
3. Barridos:
   - $Z_{RLD}$: 10, 20 y 50 kΩ (resistencia en serie con el electrodo RLD del fantoma).
   - Longitud de cable.
   - Desconexión del electrodo RLD.

### 12.3 Resultados esperados

- Ordenación T1 > T2 > T3 en la amplitud de 50 Hz.
- Cruce entre T1 y T2 cerca de $Z_{RLD} \approx R_s/G$.
- Degradación de T2 respecto a T1 con el electrodo RLD desconectado.

Si el cruce experimental aparece donde predice el modelo, se obtiene una validación cuantitativa de la expresión de la sección 7.4.

### 12.4 Relación con el modelo de denoising

El conjunto de entrenamiento del modelo U-Net + Mamba utiliza ruido de la base NSTDB (EM, MA y BW), que **no incluye interferencia de red**. El residuo de 50 Hz medido en este protocolo es una entrada fuera de la distribución de entrenamiento. Su amplitud medida permite justificar numéricamente si es necesario un filtro notch previo a la red o añadir interferencia de red al aumento de datos.

---

## 13. Errores conceptuales frecuentes

| Afirmación incorrecta | Por qué es incorrecta | Formulación correcta |
|---|---|---|
| "El ruido diferencial de cada electrodo afecta al CMRR" | El problema no es ruido diferencial de los electrodos, sino interferencia **en modo común** que se **convierte** en diferencial por el desequilibrio de impedancias | "El desequilibrio de impedancias de los electrodos, junto con la capacidad de los cables apantallados, convierte parte del modo común en señal diferencial y limita el CMRR efectivo del sistema" |
| "Conectar las mallas al RLD invierte la interferencia y la cancela" | Las mallas no se invierten ni cancelan nada. Lo que hace el RLD es excitar el cuerpo con el modo común invertido. Conectar la malla a la salida invertida (RLDOUT) **empeora** el resultado | "El nodo INRLD sigue aproximadamente el potencial del cuerpo; conectando ahí las mallas se reduce la tensión entre conductor y malla y, con ella, la conversión modo común → diferencial" |
| "Con la malla a masa el problema está resuelto" | La malla a masa elimina la captación directa pero introduce la conversión por $C_s$ | "La malla sustituye un mecanismo de interferencia por otro; la topología decide cuál domina" |
| "La corriente que causa el error viene de la malla y va directa a GND" | La malla es el sumidero, no la fuente. La corriente la impulsa el modo común del cuerpo y circula cuerpo → $Z_e$ → hilo → $C_s$ → malla → GND; por eso atraviesa $Z_e$ | "El cuerpo, elevado a $V_{CM}$ por la red, se descarga hacia la malla a través del electrodo y de la capacidad del cable" |
| "Sin malla funcionaría mejor, porque desaparece $C_s$" | Sin malla, la red (230 V) acopla directamente al hilo; la corriente por $Z_e$ es ~46 veces mayor | "La malla sustituye una fuente de cientos de voltios por otra de milivoltios, a costa de una capacidad mayor" |
| "El CMRR del sistema es el del integrado (120 dB)" | El desequilibrio de electrodos y cables lo limita a ~60 dB | "El CMRR efectivo lo fija el eslabón más débil de la cadena" |
| "Más ganancia en el RLD siempre es mejor" | Aumentar G reduce $V_B$ pero compromete la estabilidad del lazo; $R_s$ está fijada por seguridad | "G es un compromiso entre atenuación del modo común y margen de fase" |

---

## 14. Hipótesis, limitaciones y trabajo futuro

### 14.1 Hipótesis del modelo

- Parámetros concentrados; se ignora la naturaleza distribuida del cable (válido a 50 Hz, longitud de onda ~6000 km).
- $Z_e$ modelada como resistencia pura a 50 Hz. Un modelo más fiel de Ag/AgCl incluye $R_s + (R_{ct} \parallel C_{dl})$.
- Amplificador RLD de un polo; se ignoran su ruido y su rango de salida.
- En el modelo analítico se desprecia $C_{cuerpo}$ frente al camino del RLD.
- **Masa única.** El modelo trata la masa del equipo como un solo nodo. A 50 Hz es indiferente que GND y GNDA estén unidas por un net tie (0 Ω) o compartan un plano continuo: la impedancia entre ambas (mΩ) es despreciable frente a los acoplamientos capacitivos (MΩ). La división de planos solo afecta a los retornos de alta frecuencia (flancos del SPI), que este modelo no cubre; un plano continuo con partición por ubicación acerca el sistema real al modelo.
- Valores de acoplamiento a la red ($C_{red}$, $C_{ext}$) estimados; dependen fuertemente del entorno.

### 14.2 Limitaciones

- Los valores absolutos son órdenes de magnitud; la conclusión se apoya en la ordenación relativa y en la regla $Z_{RLD} < R_s/G$.
- No se ha modelado la captación magnética (bucles formados por los cables), que se minimiza trenzando o agrupando los cables.

### 14.3 Trabajo futuro

- Montar y caracterizar T3 con electrodos secos.
- Modelo de electrodo dependiente de la frecuencia.
- Validación con sujetos (solo con batería).

---

## 15. Glosario

| Término | Definición |
|---|---|
| AFE | Analog Front-End: etapa analógica de adquisición (aquí, el ADS1292) |
| CMRR | Common-Mode Rejection Ratio: relación entre ganancia diferencial y ganancia en modo común |
| CMIR | Common-Mode Input Range: rango de tensión de modo común admisible en la entrada |
| RLD | Right-Leg Drive: amplificador que realimenta el modo común invertido al cuerpo |
| RLDOUT | Salida del amplificador RLD del ADS1292 |
| INRLD | Nodo del electrodo RLD en la PCB, tras la resistencia de protección de 390 kΩ |
| $I_d$ | Corriente de desplazamiento inyectada en el cuerpo por el acoplamiento con la red |
| $C_s$ | Capacidad entre conductor y malla del cable |
| Guarding / apantallado activo | Excitar la malla con el potencial del conductor para anular la tensión sobre $C_s$ |
| Lead-off | Desconexión o mal contacto de un electrodo |
| DNP | Do Not Populate: footprint presente en la PCB pero sin componente montado |

---

## 16. Referencias

> Verificar los datos bibliográficos exactos antes de incluirlos en la memoria.

1. Texas Instruments, *ADS1291, ADS1292, ADS1292R Low-Power, 2-Channel, 24-Bit Analog Front-End for Biopotential Measurements*, datasheet SBAS502C.
2. B. B. Winter y J. G. Webster, "Driven-right-leg circuit design", *IEEE Transactions on Biomedical Engineering*, vol. BME-30, n.º 1, 1983.
3. A. C. Metting van Rijn, A. Peper y C. A. Grimbergen, "High-quality recording of bioelectric events. Part 1: Interference reduction, theory and practice", *Medical & Biological Engineering & Computing*, 1990.
4. R. Pallàs-Areny y J. G. Webster, "Common mode rejection ratio in differential amplifiers", *IEEE Transactions on Instrumentation and Measurement*, 1991.
5. J. G. Webster (ed.), *Medical Instrumentation: Application and Design*, Wiley.
6. IEC 60601-2-25, *Medical electrical equipment — Particular requirements for the basic safety and essential performance of electrocardiographs*.
