# Análisis del esquemático — PCB ECG (ADS1292 + ESP32-S3)

**Revisión 2** · Plano de masa único · Cambio de topología de mallas incluido

---

## 0. Fuentes y leyenda

### 0.1 Documentación consultada

| Componente | Documento | Estado |
|---|---|---|
| ADS1292 | TI SBAS502C, *ADS1291/ADS1292/ADS1292R*, rev. abril 2020 | Revisado en detalle |
| MCP73871 | Microchip DS20002090C | Revisado en detalle |
| ESP32-S3-WROOM-2 | Espressif, *ESP32-S3-WROOM-2 Datasheet* v1.7 y *ESP Hardware Design Guidelines (ESP32-S3)* | Revisadas las secciones de EN, alimentación e IO MUX |
| XC6220, LSM6DS3, USBLC6-2SC6 | — | **No revisados**: lo indicado es criterio general; verificar en sus datasheets |

### 0.2 Leyenda

| Marca | Significado |
|---|---|
| ✅ | Comprobado en el datasheet (se indica la sección) |
| ⚙️ | Criterio de diseño: justificado, pero no lo exige el datasheet |
| 🔶 | Estimación de orden de magnitud; debe medirse |

---

## 1. Cambios respecto a la revisión 1

| # | Revisión 1 decía | Revisión 2 | Motivo |
|---|---|---|---|
| 1 | VCAP1 = 22 µF | **VCAP1 = 1 µF es correcto** | Error de la rev. 1: 22 µF corresponde a otra familia de TI. Fig. 30 y Fig. 73 indican 1 µF |
| 2 | AVDD y DVDD: 10 µF + 0.1 µF obligatorio | **1 µF + 0.1 µF en cada pin es válido**; 10 µF recomendado en el nodo +3.3VA | El datasheet se contradice (Fig. 73 frente a §11.1.1.1). Ver §3.2 |
| 3 | Net tie de 0 Ω entre GND y GNDA | **Plano de masa único** | Decisión de diseño. Ver §3.2.2 |
| 4 | Quitar los 5 V del conector UART | **Se mantienen** | USB y UART nunca se conectan a la vez |
| 5 | Opción de reloj externo para muestrear a 360 SPS | **Imposible** | El modulador debe ir a 128 kHz y el reloj externo admite solo 485–562.5 kHz o 1.94–2.25 MHz (§8.3.7). Ver §10.1 |
| 6 | SPI a 20 MHz | 20 MHz **solo para leer datos**; los registros exigen SCLK ≤ 2·f_CLK o pausas entre bytes | §8.5.1.2 y §8.5.2.10. Ver §3.6 |
| 7 | Pines sin usar no revisados | **Tres grupos de pines mal terminados** (IN3P/IN3N, RLDIN/RLDREF, GPIO1/2) | Tabla *Pin Functions* y §8.5.1 (GPIO). Ver §3.3 |
| 8 | PWDN/RESET a GPIO: mejora opcional | **Necesario**: atado a 3.3VA no cumple la secuencia de arranque | §10.1 *Power-Up Sequencing* |
| 9 | Resistencias de sensado del RLD de 200 kΩ, G = 10 | **400 kΩ por entrada**; G = 5 o 10 según RLD_SENS | Fig. 40. Ver §3.4 |
| 10 | Detección de lead-off del RLD en operación | **Solo funciona en el arranque** | §8.3.10.2.3 |
| 11 | Mallas a GNDA | Nueva net `SHIELD` seleccionable (INRLD o GND) | Ver §4 |

---

## 2. Arquitectura del sistema

```
                 USB µB ──USBLC6──┐ D+/D-  (IO19/IO20, USB-Serial-JTAG nativo)
                  VBUS            │
UART J3 ──USBLC6──┤ +5V ◄─────────┘   (USB y UART nunca simultáneos)
                  ▼
          ┌─────────────┐  OUT        ┌────────────┐ +3.3V ┌───────────────────┐
LiPo J1 ◄─┤ MCP73871    ├────────────►│ XC6220 LDO ├──┬───►│ ESP32-S3-WROOM-2  │
  VBAT    │ (load-share)│             └────────────┘  │    │ Flash+PSRAM octal │
          └─────────────┘                             │    └───┬───────┬───────┘
                                          FB1 (ferrita)        │SPI    │I²C
                                                      ▼        │       │
                                   +3.3VA ─────► ┌────────┐◄───┘   ┌────▼────┐
                                                 │ADS1292 │ DRDY    │LSM6DS3  │
 Electrodos J15–J18 ─22k─┬─► IN1/IN2             │ 2 ch   ├──► IO9  │ INT1/2  │
                     470pF (diferencial)         │ + RLD  │         └─────────┘
 J19 ◄── INRLD ──390k── RLDOUT ◄─────────────────┘
 J20–J24 (mallas) ──► SHIELD ──► INRLD (por defecto) / GND (alternativa)

 Masa: un único plano continuo (GND = AVSS = DGND), con partición por ubicación
```

---

## 3. ADS1292

### 3.1 Referencia: VREFP, VREFN, VCAP1 y VCAP2

#### 3.1.1 Qué dicen las Figuras 30, 73 y 74

| Pin | Tu esquemático | Fig. 30 (§8.3.6, p. 27) | Fig. 73 (§11.1.1.1.1, p. 67) | Fig. 74 (§11.1.1.1.2, p. 68) | Veredicto |
|---|---|---|---|---|---|
| VREFP–VREFN | 1 µF ∥ 0.1 µF | 10 µF ∥ 0.1 µF | 10 µF ∥ 0.1 µF | 10 µF ∥ 0.1 µF | ❌ **Cambiar 1 µF → 10 µF** ✅ |
| VREFN | A GNDA | — | A masa (= AVSS, alimentación unipolar) | A −1.5 V (= AVSS, alimentación bipolar) | ✅ Correcto |
| VCAP1 | 1 µF | 1 µF | 1 µF | 1 µF (a AVSS) | ✅ Correcto |
| VCAP2 | 1 µF | — | 1 µF | 1 µF (a AVSS) | ✅ Correcto |

**Lectura de las Figuras 73 y 74.** En ambas, los condensadores de 0.1 µF y 10 µF están **en paralelo entre VREFP y VREFN**. La Fig. 74 aclara a qué nodo va VREFN: con alimentación bipolar se conecta a −1.5 V, es decir, a **AVSS** y no a una "masa" genérica. Lo confirma la tabla *Pin Functions*: VREFN *must be connected to AVSS*. Con alimentación unipolar (tu caso), AVSS = 0 V = GND, así que tu topología es correcta. Solo falla el valor del condensador grande.

En la Fig. 74 se ve también que **VCAP1 y VCAP2 retornan a AVSS**. En tu placa, AVSS = GND, así que también es correcto.

#### 3.1.2 Física de cada condensador

**VCAP1** filtra la salida del bandgap. La Fig. 30 muestra una resistencia interna R1 = 100 kΩ (para VREF = 2.42 V), así que el condensador externo forma un filtro paso bajo:

$$
f_c = \frac{1}{2\pi R_1 C_{VCAP1}} = \frac{1}{2\pi\cdot 100\,\text{k}\Omega\cdot 1\,\mu\text{F}} \approx 1.6\,\text{Hz}
$$

Según §8.3.6, para ECG de alta calidad el ancho de banda de la referencia debe quedar **por debajo de 10 Hz** para que su ruido no domine. Con 1 µF se cumple con margen. ✅

El tiempo de asentamiento de VCAP1 al 1 % es de **0.5 s** (tabla de características eléctricas, *System Monitors*). El firmware debe descartar los datos de ese primer intervalo tras el arranque. ✅

**VREFP** es la salida del buffer de referencia que alimenta al modulador ΔΣ. El condensador de 10 µF cumple dos funciones:

1. **Limitar el ancho de banda del ruido** de la referencia (§8.3.6). ✅
2. **Actuar como depósito de carga**: el modulador toma muestras a $f_{MOD}=128\,\text{kHz}$ (§8.3.4.3), y en cada muestreo extrae un pequeño paquete de carga del nodo de referencia. Cuanto mayor es el condensador, menor es la perturbación de tensión $\Delta V = Q/C$ en cada ciclo. ⚙️

**Colocación:** el 0.1 µF lo más cerca posible de los pines 9–10; el 10 µF justo detrás. La nota de las Fig. 73 y 74 lo pide para todos los condensadores de alimentación, referencia, VCAP1 y VCAP2. ✅

#### 3.1.3 Dieléctrico de VCAP1

Según §11.1.1.1, los cerámicos de clase 2 (X7R, X5R…) son **piezoeléctricos**: si la placa vibra, generan ruido eléctrico. Para placas sometidas a vibración, el datasheet recomienda en VCAP1 un condensador **de tántalo o de clase 1 (C0G/NP0)**. ✅

**En un dispositivo de ECG pegado al cuerpo**, el movimiento del paciente es justamente esa vibración. Recomendación: **tántalo de 1 µF en VCAP1**. Un C0G de 1 µF existe, pero es grande y caro. ⚙️

### 3.2 Alimentación AVDD y DVDD con plano de masa único

#### 3.2.1 Qué dice el datasheet (se contradice)

| Fuente | AVDD | DVDD |
|---|---|---|
| Fig. 73 (alimentación unipolar) | 1 µF + 0.1 µF | 0.1 µF + 1 µF |
| Texto §11.1.1.1 *Power Supplies and Grounding* | 10 µF + 0.1 µF | 10 µF + 0.1 µF |
| **Tu esquemático** | 1 µF + 0.1 µF | 1 µF + 0.1 µF |

Tus valores coinciden con la Fig. 73 y son válidos. ✅

Rangos de alimentación (§6.3 *Recommended Operating Conditions*): AVDD − AVSS de 2.7 a 5.25 V; DVDD − DGND de 1.7 a 3.6 V. Con 3.3 V en ambos, dentro de rango. ✅ Con AVDD = 3.3 V la referencia interna debe configurarse a 2.42 V (§8.3.6). ✅

#### 3.2.2 Qué cambia al unificar la masa

**Los valores de los condensadores no dependen de cómo esté organizada la masa.** Dependen de la corriente transitoria que hay que suministrar y del ruido que hay que filtrar. Lo que cambia es el **camino de retorno**:

| Aspecto | Con GND/GNDA separadas | Con plano único |
|---|---|---|
| Retorno de cada condensador de desacoplo | Hacia GNDA, que solo une con GND en el net tie | Vía corta directa al plano, bajo el propio condensador |
| Retorno de los flancos del SPI | Rodeo hasta el net tie (lazo grande) | Bajo la propia pista |
| Protección del analógico | Por el corte | **Por ubicación** (§11.1.1.1: los retornos digitales no deben cruzar el camino de retorno analógico del ADS1292) ✅ |

La propia Fig. 73 dibuja AVSS y DGND hacia símbolos de masa sin separarlos: con alimentación unipolar ambos están a 0 V. El plano único es coherente con el datasheet. ✅

**En KiCad:** o se fusionan los nets GND y GNDA, o se mantiene GNDA con un componente *net tie* para que el DRC acepte la unión. Las dos opciones dan el mismo cobre.

#### 3.2.3 Interacción con la ferrita FB1 (si se mantiene)

AVDD y DVDD se alimentan desde +3.3VA, a la salida de FB1. A baja frecuencia una ferrita se comporta como una inductancia, así que con los condensadores forma un filtro LC con resonancia:

$$
f_0 = \frac{1}{2\pi\sqrt{L\,C}}
$$

Con una ferrita típica (del orden de 1 µH en su zona inductiva; depende del modelo) 🔶:

| Capacidad total en +3.3VA | $f_0$ aproximada |
|---|---|
| ~2.2 µF (1 µF + 0.1 µF en AVDD y en DVDD) | ~107 kHz |
| ~12 µF (añadiendo 10 µF) | ~46 kHz |

**Por qué importa.** El modulador muestrea a $f_{MOD}=128\,\text{kHz}$, y el datasheet advierte que AVDD tiene transitorios a $f_{CLK}$ y que debe eliminarse el ruido no síncrono con el dispositivo (§11.1.1.1). ✅ Una resonancia poco amortiguada cerca de $f_{MOD}$ amplifica justo esas componentes, y las que caigan cerca de múltiplos de $f_{MOD}$ se pliegan a la banda útil al muestrear, donde el filtro de diezmado ya no puede eliminarlas. ⚙️

El factor de calidad del resonador es:

$$
Q = \frac{1}{R}\sqrt{\frac{L}{C}}
$$

donde $R$ agrupa la resistencia de la ferrita y la ESR de los condensadores. Más capacidad baja $f_0$ por debajo de $f_{MOD}$ y además reduce $Q$.

#### 3.2.4 Recomendación final

| Nodo | Valor | Marca |
|---|---|---|
| Pin AVDD (12) | 0.1 µF + 1 µF pegados al pin | ✅ Fig. 73 (ya lo tienes) |
| Pin DVDD (23) | 0.1 µF + 1 µF pegados al pin | ✅ Fig. 73 (ya lo tienes) |
| Nodo +3.3VA, tras FB1 | **Añadir 10 µF** | ⚙️ Cumple también el texto de §11.1.1.1 y amortigua el LC |
| Alternativa opcional | Alimentar **DVDD desde +3.3V** (antes de FB1) y dejar FB1 solo para AVDD | ⚙️ Las corrientes de conmutación de las salidas digitales (DOUT, DRDY) no entran en el nodo de AVDD. DVDD admite 1.7–3.6 V |

Si en el layout final se elimina FB1, el problema de resonancia desaparece. El 10 µF sigue siendo recomendable como reserva de carga cerca del ADS1292.

### 3.3 Pines de control y pines sin usar

| Pin | Tu esquemático | Datasheet | Veredicto |
|---|---|---|---|
| CLKSEL (14) | A +3.3VA | 1 = oscilador interno (Tabla 9, §8.3.7) | ✅ Correcto |
| CLK (17) | Sin conectar | Con CLKSEL = 1 y CLK_EN = 0, el pin queda en alta impedancia (Tabla 9) | ✅ Correcto; mantener CLK_EN = 0 |
| START (16) | A GND | Si se usa el comando START por SPI, el pin debe estar a nivel bajo (§8.5.1.9 *START*) | ✅ Correcto |
| **PWDN/RESET (15)** | A +3.3VA | Todas las entradas digitales deben estar **a nivel bajo durante el arranque** hasta que la alimentación sea estable; después se espera $t_{POR}$ y se aplica un pulso de RESET (§10.1) | ❌ **Llevar a un GPIO con pull-down de 10 kΩ** ⚙️ |
| **RESP_MODP/IN3P (31), RESP_MODN/IN3N (32)** | Sin conectar | *Connect unused analog inputs to AVDD* (nota 1 de la tabla *Pin Functions*) | ❌ **Conectar a AVDD** |
| **RLDIN/RLDREF (29)** | Sin conectar | *Connect to AVDD if not used* (tabla *Pin Functions*); no se usa si la referencia del RLD es interna (RLDREF_INT) | ❌ **Conectar a AVDD** |
| **GPIO1 (26), GPIO2 (25)** | Sin conectar | Tras el arranque son entradas y *must be driven (do not float)*; si no se usan, *shorted to DGND with a series resistor* (§8.5.1.7 *GPIO*) | ❌ **A GND con resistencia en serie** (p. ej. 10 kΩ ⚙️) |
| RLDINV (28) | Red 1 MΩ ∥ 1.5 nF a RLDOUT | Valores típicos de la Fig. 40 | ✅ Correcto |
| PGA1N/P, PGA2N/P | 4.7 nF | Fig. 73 | ✅ Correcto |

**Sobre los condensadores de 4.7 nF del PGA** (§8.3.4) ✅: deben ir **pegados a los pines**. La capacidad parásita de cada nodo a masa debe ser menor de 20 pF, y el desequilibrio entre ambos degrada el CMRR según la Ecuación 4:

$$
\text{CMRR} = 20\log\left(\frac{\text{Gain}}{2\pi\cdot 2\cdot 10^{3}\cdot\Delta C_P\cdot 60}\right)
$$

Por ejemplo, 20 pF de desequilibrio con ganancia 6 limitan el CMRR a 112 dB. Rutado simétrico. Dieléctrico C0G recomendado por estabilidad ⚙️.

**Secuencia de arranque en firmware** (§10.1 y §8.5.1.8) ✅:

1. Mantener PWDN/RESET a nivel bajo hasta que la alimentación sea estable.
2. Liberar PWDN/RESET y esperar $t_{POR}=2^{12}\,t_{MOD}\approx 32\,\text{ms}$.
3. Aplicar un pulso de RESET (por pin o con el comando RESET) y esperar 18 $t_{CLK}$.
4. Enviar SDATAC y escribir los registros. Todos los registros deben reescribirse tras un power-down.
5. Esperar el asentamiento de VCAP1 (0.5 s) antes de considerar válidos los datos.

### 3.4 Amplificador RLD

| Aspecto | Valor | Marca |
|---|---|---|
| Realimentación | $R_{EXT}$ = 1 MΩ ∥ $C_{EXT}$ = 1.5 nF | ✅ Valores típicos de la Fig. 40 |
| Resistencia de sensado por entrada | **400 kΩ**, tomada del lado del PGA (no de la entrada) | ✅ Fig. 40 |
| Referencia | Interna, $(AVDD+AVSS)/2 = 1.65\,\text{V}$ (RLDREF_INT) | ✅ §8.3.10.2.4 |
| Resistencia serie hacia el paciente | 390 kΩ, limita la corriente a $3.3\,\text{V}/390\,\text{k}\Omega\approx 8.5\,\mu\text{A}$ en el peor caso | ⚙️ |

**Ganancia del lazo según RLD_SENS.** Con N entradas seleccionadas, las resistencias de sensado quedan en paralelo, $R_{in}=400\,\text{k}\Omega/N$, y $G_0=R_{EXT}/R_{in}$:

| Entradas en RLD_SENS | $R_{in}$ | $G_0$ |
|---|---|---|
| Canal 1 (RLD1P, RLD1N) | 200 kΩ | 5 |
| Ambos canales (4 entradas) | 100 kΩ | 10 |

**Detección de lead-off del RLD** (§8.3.10.2.3) ✅: el ADS1292 solo comprueba el electrodo RLD **en el arranque**, porque durante la operación normal exigiría apagar el amplificador. Para vigilarlo en funcionamiento hay dos alternativas: enrutar RLDOUT a un canal libre (§8.3.10.1.1), lo que no es posible si los dos canales se usan para ECG, o detectarlo en firmware a partir de un aumento brusco de la componente de 50 Hz.

### 3.5 Rango de modo común y ganancia del PGA

La Ecuación 5 (§8.3.4.1) ✅ fija el rango de modo común admisible:

$$
\text{AVSS} + 0.2\,\text{V} + \frac{G\cdot V_{MAX\_DIFF}}{2} < V_{CM} < \text{AVDD} - 0.2\,\text{V} - \frac{G\cdot V_{MAX\_DIFF}}{2}
$$

donde $V_{MAX\_DIFF}$ es la máxima tensión diferencial a la entrada del PGA, **incluido el offset DC entre electrodos**, que en electrodos de gel puede ser de cientos de mV.

Ejemplo con AVDD = 3.3 V, $G=6$ y $V_{MAX\_DIFF}=300\,\text{mV}$ 🔶:

$$
1.1\,\text{V} < V_{CM} < 2.2\,\text{V}
$$

El RLD fija el cuerpo en 1.65 V, en el centro de la ventana. ✅ El fondo de escala con $G=6$ es $\pm V_{REF}/G=\pm 2.42/6\approx\pm 403\,\text{mV}$ (§8.3.8), suficiente para ese offset.

### 3.6 Interfaz SPI

| Aspecto | Dato | Marca |
|---|---|---|
| SCLK máximo | 20 MHz ($t_{SCLK}\geq 50\,\text{ns}$ con DVDD de 2.7 a 3.6 V) | ✅ §6.6 *Timing Requirements* |
| Lectura de registros (RREG/WREG) | SCLK ≤ 2·f_CLK = 1.024 MHz, **o bien** pausa de 4 $t_{CLK}$ ≈ 7.8 µs entre bytes | ✅ §8.5.1.2 y §8.5.2.10 |
| Datos por muestra | 72 bits (24 de estado + 2 × 24) | ✅ Ecuación 9 |
| Modo SPI | CPOL = 0, CPHA = 1 | ✅ Nota del diagrama de temporización (§6.6) |

**Remapeo opcional al IO MUX.** La asignación actual (IO10 = MISO, IO11 = SCK, IO12 = MOSI, IO13 = CS) pasa por la GPIO matrix. En el ESP32-S3, las funciones nativas de SPI2 (FSPI) son GPIO10 = FSPICS0, GPIO11 = FSPID (MOSI), GPIO12 = FSPICLK y GPIO13 = FSPIQ (MISO) ✅ (datasheet del ESP32-S3, tabla IO MUX). Reordenando a CS = 10, MOSI = 11, SCK = 12 y MISO = 13 se usan las rutas directas. A las velocidades del ADS1292 la mejora es marginal ⚙️.

---

## 4. Cambio de topología de las mallas

### 4.1 Estado actual

```
 J20 ─► GND      J15 ─22k─┬─ IN1N        J19 ──────────── INRLD ──390k── RLDOUT
                        470pF
 J21 ─► GND      J16 ─22k─┴─ IN1P        J24 ─► GND

 J22 ─► GND      J17 ─22k─┬─ IN2N
                        470pF
 J23 ─► GND      J18 ─22k─┴─ IN2P
```

### 4.2 Propuesta

```
 J20 ─┐
 J21 ─┤
 J22 ─┼──────── SHIELD ───┬── R_SH1  0 Ω  0603  (MONTADA) ──► INRLD   (T2)
 J23 ─┤                   │
 J24 ─┘                   └── R_SH2  0 Ω  0603  (DNP)     ──► GND     (T1)
```

| Cambio | Detalle |
|---|---|
| Nueva net `SHIELD` | Etiqueta local o global. **Nunca** un símbolo de alimentación: no es una masa |
| J20–J23 | Pasan de GND a SHIELD |
| J24 (malla del cable RLD) | A SHIELD. En T2 queda al potencial de su propio conductor, lo cual es inocuo, y todas las mallas se gestionan juntas |
| R_SH1 (0 Ω, montada) | De SHIELD a **INRLD**: en el lado del electrodo de los 390 kΩ, **nunca en RLDOUT** |
| R_SH2 (0 Ω, DNP) | De SHIELD a GND. Atributo *Do not populate* en KiCad |
| Nota en el esquemático | "Montar R_SH1 **o** R_SH2, nunca ambas: cortocircuitaría INRLD a masa" |

**Layout:** R_SH1 y R_SH2 junto a J19/J24. Sacar los pads de malla del plano de masa y unirlos con una pista dedicada; SHIELD no se rellena como polígono.

**Apantallado activo (buffer de modo común):** no se incluye. Con electrodos Ag/AgCl de gel, su mejora sobre T2 queda por debajo del ruido del ADS1292, y añadiría componentes en el nodo más sensible.

El análisis teórico completo está en el documento *Interferencia en modo común y topología de apantallado de los electrodos*.

---

## 5. ESP32-S3-WROOM-2

| Punto | Estado | Recomendación |
|---|---|---|
| **EN** | R1 sin valor ("R"), C1 = 1 nF | **R = 10 kΩ, C = 1 µF** ✅ (datasheet del WROOM-2 y *Hardware Design Guidelines*) |
| Alimentación 3V3 | 10 µF + 100 nF | ✅ Espressif recomienda 10 µF en el rail por los picos de corriente al transmitir |
| BOOT (IO0) | Pulsador con 1 nF | ✅ Correcto |
| USB nativo | IO19 = D−, IO20 = D+ | ✅ Correcto |

**Sobre el circuito RC de EN.** Con 1 nF, τ = R·C es del orden de 10 µs y el chip puede salir de reset antes de que la alimentación sea estable. Espressif advierte además que con **subidas lentas de alimentación, como durante la carga de una batería**, el RC puede no bastar. Si aparecen arranques fallidos, la solución es un supervisor de tensión en EN ⚙️.

---

## 6. Alimentación: MCP73871 y XC6220

### 6.1 MCP73871 (variante -2CC)

| Pin / función | Tu esquemático | Datasheet | Veredicto |
|---|---|---|---|
| Pines duplicados | Solo se ve un pin de cada | IN = 18, 19; OUT = 1, 20; VBAT = 14, 15; VSS = 10, 11 + EP (Tabla 3-1) | ⚠️ Verificar que el símbolo y la huella conectan **todos** |
| EP | — | Debe conectarse a VSS (§3.17); vías térmicas para disipar (§6.2) | ⚠️ Verificar |
| **C2 (IN)** | Sin valor ("C") | Mínimo 4.7 µF (§3.1); la aplicación típica usa 10 µF | ❌ **Asignar 10 µF** |
| OUT | 4.7 µF | Mínimo 4.7 µF (§3.2) | ✅ |
| VBAT | 4.7 µF | Mínimo 4.7 µF (§3.6, §6.1.1.3) | ✅ |
| VPCC | A IN (+5V) | Así se deshabilita la función VPCC (§3.3) | ✅ |
| SEL | A GND | Modo USB (§3.4) | ✅ |
| PROG2 | A +5V | Límite de entrada USB 500 mA (typ. 450 mA) | ✅ |
| PROG1 = 2 kΩ | — | $I_{REG}=1000\,\text{V}/R_{PROG1}=500\,\text{mA}$ (Ec. 4-1), pero en modo USB la carga real queda limitada por el límite de entrada menos el consumo del sistema (§3.8) | ✅ |
| PROG3 = 25 kΩ | — | $I_{TERM}=1000\,\text{V}/R_{PROG3}=40\,\text{mA}$ (Ec. 4-2); rango recomendado 5–100 kΩ | ✅ |
| THERM = 10 kΩ | — | Así se deshabilita la monitorización de temperatura (§5.1.4) | ✅ Sin protección térmica de la batería |
| TE | A GND | Temporizador activo; la variante -2CC tiene 6 h y LBO a 3.1 V | ✅ |
| CE | A +5V | Carga habilitada; las entradas no deben quedar flotantes | ✅ |
| LEDs (PG, STAT1, STAT2) | Ánodo a +5V | PG no debe tener pull-up por encima de VIN (§5.2.4); +5V = VIN | ✅ |

**Térmica** (§6.1.1.2) ✅: el peor caso de disipación es la transición de precarga a corriente constante:

$$
P = (V_{DD,max} - V_{PTH,min})\cdot I_{REG,max}
$$

Con 5.25 V de entrada, umbral de precarga del ~69 % de 4.2 V (~2.9 V) y 0.5 A, $P\approx 1.2\,\text{W}$ 🔶. La resistencia térmica del encapsulado es de 50 °C/W **en placa JEDEC de 4 capas**; en 2 capas será mayor, así que el chip entrará en regulación térmica y reducirá la corriente de carga. Vías térmicas en el EP imprescindibles.

**Temporizador** 🔶: con ~450 mA de carga y un temporizador de 6 h, una batería de varios Ah podría no completar la carga. Si la batería es grande, poner TE a nivel alto para deshabilitarlo.

### 6.2 XC6220 (no verificado)

Con batería LiPo (3.0–4.2 V) y salida de 3.3 V, comprobar en su datasheet la caída de tensión a la corriente de pico del ESP32-S3 (transmisión WiFi). Si la radio no se usa durante la inferencia, el riesgo es menor ⚙️.

### 6.3 Conector UART

Se mantienen los 5 V en J3: USB y UART nunca se conectan simultáneamente. Recomendación: nota en la serigrafía junto a J3, "No conectar con USB" ⚙️.

---

## 7. Otros periféricos

| Punto | Comentario | Marca |
|---|---|---|
| LSM6DS3 | Comprobar estado de ciclo de vida en ST y que el símbolo conecta todos los pines de GND | ⚙️ No verificado |
| Pull-ups I²C (10 kΩ) | Tiempo de subida $t_r\approx 0.85\,R\,C_{bus}$; a 400 kHz, 4.7 kΩ dan más margen | ⚙️ |
| USBLC6 del USB | Es simétrico: usando 1/6 para D+ y 3/4 para D− se evita el cruce de pistas | ⚙️ Verificar en su datasheet |
| Filtro de entrada | 22 kΩ + 470 pF diferencial: $f_c=1/(2\pi\cdot 2R\cdot C)\approx 7.7\,\text{kHz}$. Filtro EMI; el antialiasing lo hacen el PGA (4.7 nF) y el filtro de diezmado | ✅ / ⚙️ |

---

## 8. Instrumentación para el TFM

| Elemento | Uso | Marca |
|---|---|---|
| Shunt o jumper en serie con el 3V3 del módulo | Medir la energía por inferencia, $E=\int V\,I\,dt$ | ⚙️ |
| Test point en un GPIO libre | Marcar inicio y fin de cada capa para medir latencias con osciloscopio | ⚙️ |
| Test points en DRDY, SCLK y CS | Analizador lógico | ⚙️ |
| Divisor de VBAT (2 × 100 kΩ + 100 nF) a IO1 o IO2 (ADC1) | Estado de la batería | ⚙️ |

---

## 9. Seguridad del paciente

La placa no tiene aislamiento galvánico. **No colocar electrodos a una persona mientras el USB está conectado a un PC enchufado a la red.** Las adquisiciones con personas deben hacerse solo con batería o con un aislador USB.

---

## 10. Implicaciones para Mamba

### 10.1 Tasa de muestreo

El modelo está entrenado a **360 Hz** (MIT-BIH). El ADS1292 ofrece de 125 SPS a 8 kSPS en potencias de 2 a partir de 125 SPS ✅. **No es posible obtener 360 SPS ajustando el reloj** ✅: el modulador debe funcionar a 128 kHz con cualquier reloj externo (§8.3.7), y el reloj externo solo admite 485–562.5 kHz (CLK_DIV = 0) o 1.94–2.25 MHz (CLK_DIV = 1). Dentro de esos rangos, la tasa de 500 SPS solo puede moverse entre ~473 y ~549 SPS.

Opciones válidas:

1. **Remuestrear 500 → 360 Hz** en el ESP32-S3 con un filtro polifásico de razón 18/25 ($500\cdot 18/25=360$).
2. **Reentrenar** el modelo a 250 o 500 Hz.

La precisión del oscilador interno es de ±0.5 % a 25 °C y ±1.5 % en todo el rango de temperatura ✅, así que la tasa real de 500 SPS puede desviarse unos pocos SPS.

**Escala:** el LSB vale $V_{REF}/(2^{23}-1)$ a la entrada del ADC (§8.3.8) ✅; referido a la entrada, $V_{REF}/(G\,(2^{23}-1))$. Para pasar a mV, como espera el modelo, se multiplica cada código por ese valor.

### 10.2 Coste del modelo

Estimación a partir de la arquitectura (U-Net con base 64 y 4 niveles; bottleneck con 2 bloques BiMamba, $d_{model}=1024$, $d_{state}=64$, expansión 2). Los ~32.04 M de parámetros del bottleneck coinciden con los ~32.05 M del docstring 🔶.

| Configuración | Parámetros | INT8 | GMAC / ventana (3600) | GMAC en Mamba | exp() / ventana |
|---|---|---|---|---|---|
| Actual (base 64, $d_{state}$ 64) | 42.9 M | 43 MB | 11.8 | 7.3 | 118 M |
| base 32, $d_{state}$ 32 | 10.7 M | 10.7 MB | 2.95 | 1.83 | 29 M |
| base 16, $d_{state}$ 16 | 2.7 M | 2.7 MB | 0.74 | 0.46 | 7.4 M |

**El modelo completo no cabe ni en INT8**: 43 MB superan los 32 MB de flash de la variante mayor del WROOM-2. El 62 % de los MACs está en el bottleneck, dominado por las proyecciones lineales.

### 10.3 Cuello de botella del scan selectivo

$$
h_t[c,n] = e^{\Delta_t[c]\,A[c,n]}\,h_{t-1}[c,n] + \Delta_t[c]\,B_t[n]\,x_t[c]
$$

La exponencial se evalúa una vez por cada $(t,c,n)$: 7.4 M veces por ventana incluso en el modelo reducido. Con una `expf` de ~100 ciclos serían ~3 s a 240 MHz 🔶. Alternativas: tabla de consulta, aproximación de Schraudolph, o **fijar A no entrenable** a $A_n=-(n+1)$, con lo que $e^{\Delta A_n}=(e^{-\Delta})^{n+1}$ requiere una sola exponencial por canal [PAPER].

### 10.4 Throughput 🔶

- **INT8 con PIE** (SIMD de 128 bits, 16 MAC/ciclo de pico): ~0.5–1.5 s por ventana en base 16.
- **FP32** (FPU escalar): ~7–15 s por ventana.
- El modelo es bidireccional con ventanas de 10 s: la latencia es ≥ 10 s por diseño.

---

## 11. Resumen de acciones

| Prioridad | Acción | Sección |
|---|---|---|
| 🔴 | VREFP: 1 µF → **10 µF** (mantener 0.1 µF) | §3.1 |
| 🔴 | PWDN/RESET a GPIO con pull-down de 10 kΩ | §3.3 |
| 🔴 | IN3P/IN3N (31, 32) y RLDIN/RLDREF (29) a AVDD | §3.3 |
| 🔴 | GPIO1/GPIO2 (25, 26) a GND con resistencia en serie | §3.3 |
| 🔴 | EN del ESP32: 10 kΩ + 1 µF | §5 |
| 🔴 | C2 (IN del MCP73871): 10 µF | §6.1 |
| 🔴 | Verificar pines duplicados y EP (MCP73871, LSM6DS3, WROOM-2) | §6.1, §7 |
| 🟡 | Net SHIELD con R_SH1 (montada, a INRLD) y R_SH2 (DNP, a GND) | §4 |
| 🟡 | 10 µF en +3.3VA tras FB1 | §3.2 |
| 🟡 | VCAP1 de tántalo | §3.1 |
| 🟡 | Vías térmicas en el EP del MCP73871 | §6.1 |
| 🟢 | DVDD desde +3.3V; remapeo del SPI al IO MUX | §3.2, §3.6 |
| 🟢 | Instrumentación: shunt, test points, divisor de VBAT | §8 |
| 🟢 | Serigrafía "No conectar con USB" en J3 | §6.3 |

---

## 12. Preguntas abiertas

1. ¿Se mantiene FB1 en el layout final? Condiciona la necesidad del 10 µF en +3.3VA.
2. ¿RLD_SENS con dos o con cuatro entradas? Cambia la ganancia del lazo RLD (5 o 10) y, con ella, el umbral de calidad del electrodo RLD para la topología de mallas.
3. ¿Qué variante del WROOM-2 se monta (N16R8V, N32R8V o N32R16V)? Decide el tamaño de modelo que cabe.
4. ¿Remuestreo en firmware o reentrenamiento a 500 Hz?
5. ¿Capacidad de la batería? Condiciona la corriente de carga (≤1C) y el temporizador de 6 h.
