# Robot: conexiones y calibraciones

Raspberry Pi + placa Bee 2.0 · L298N · 2 motores N20 con encoder · 2 CNY70 · Sharp GP2Y0A41SK0F · MCP3008
Pines en numeración **BCM** (los terminales **Bxx** de la Bee).

---

## 1. Alimentación

| Desde | Hasta | Nota |
|---|---|---|
| Portapilas (4 pilas, ~6 V) **+** | Línea **+** de pilas (breadboard) → tornillo **+12V** de la L298N | |
| Portapilas **−** | Línea **−** de pilas → tornillo **GND** de la L298N | |
| **GND** de la Bee | Línea **−** de pilas | **Masa común obligatoria** |
| **3,3 V** de la Bee | Línea de **3,3 V** (otro lado de la breadboard) | Encoders y CNY70 |
| **5 V** de la Bee | Solo el **Sharp** (rojo) | |
| Power bank | Raspberry Pi | |

- Tornillo **+5V** de la L298N: **vacío**.
- Jumper del **regulador de 5 V** de la L298N: **puesto**.
- Jumpers de **ENA y ENB**: **quitados** (si no, meten 5 V en un GPIO de 3,3 V).
- Las dos líneas de la breadboard (pilas y 3,3 V) **nunca se unen**. Nada de la Raspberry va a la línea **+** de pilas.

---

## 2. Motores (L298N)

| Motor | Cables del motor | Salida L298N |
|---|---|---|
| Izquierdo (A) | M1 / M2 | OUT1 / OUT2 |
| Derecho (B) | M1 / M2 | OUT3 / OUT4 |

| Señal L298N | Pin Bee | Función |
|---|---|---|
| IN1 | **B17** | Sentido motor izq. |
| IN2 | **B27** | Sentido motor izq. |
| ENA | **B13** (PWM1) | Velocidad motor izq. |
| IN3 | **B22** | Sentido motor der. |
| IN4 | **B23** | Sentido motor der. |
| ENB | **B12** (PWM0) | Velocidad motor der. |

| IN1 | IN2 | Resultado |
|---|---|---|
| 1 | 0 | Un sentido |
| 0 | 1 | Sentido contrario |
| 0 | 0 | Rueda libre |
| 1 | 1 | Freno |

> Los motores van en espejo: el derecho se invierte **por software** (`INVERTIR_DER = True`).
> Velocidad por **ENA/ENB** con `PWMOutputDevice`, no con `gpiozero.Motor` (ese hace el PWM en IN1/IN2).

---

## 3. Encoders

| Cable | Izquierdo | Derecho |
|---|---|---|
| VCC | 3,3 V | 3,3 V |
| GND | GND | GND |
| C1 (canal A) | **B26** | **B16** |
| C2 (canal B) | **B21** | **B20** |

- Siempre a **3,3 V**, nunca a las pilas.
- `DigitalInputDevice(pin, pull_up=None, active_state=True)`, decodificación **x4**.

---

## 4. Sensores analógicos (MCP3008 por SPI)

| Sensor | Canal MCP3008 | Alimentación |
|---|---|---|
| Sharp (amarillo = Vo) | **A0 / CH0** | Rojo → **5 V**, negro → GND |
| CNY70 izquierdo | **A1 / CH1** | 3,3 V |
| CNY70 derecho | **A2 / CH2** | 3,3 V |

- Jumpers del MCP3008: **J2, J11, J14, J18**. SPI activado.
- SPI: `/dev/spidev0.0`, 1 MHz, Vref = 3,3 V.

### Cableado de cada CNY70 (red de resistencias de la Bee)

| Pata CNY70 | Izquierdo | Derecho |
|---|---|---|
| A (LED +) | 3,3 V | 3,3 V |
| C (colector) | 3,3 V | 3,3 V |
| B (LED −) | J15, fila de R8 (220 Ω) | J15, fila de R9 (220 Ω) |
| D (emisor) | J15, fila de R10 (10 kΩ) | J15, fila de R11 (10 kΩ) |
| Salida | Otro pin de la fila de R10 → **A1** | Otro pin de la fila de R11 → **A2** |

Interruptores de **U3**: **9, 7, 8 y 6 → GND**. El resto de U3 y U4 en el centro (abiertos).

---

## 5. Pines ocupados / reservados

| Uso | Pines |
|---|---|
| Motores | B17, B27, B13, B22, B23, B12 |
| Encoders | B26, B21, B16, B20 |
| SPI (MCP3008) | B8–B11 — **no usar** |
| I2C | B2 / B3 — **no usar** |
| Zumbador | **Pendiente**: J3 no es su jumper (J3 = salida PWM0/PWM1). No unir PWM0 con el GND central de J3 |

---

## 6. Calibraciones

### Motores — PWM a 1 kHz

Periodo = 1.000.000 ns · `duty_cycle (ns) = % × 10.000`

| Motor | Sentido | Arranque | Parada | Máx. velocidad |
|---|---|---|---|---|
| Motor 1 (izq.) | Horario | 28 % (280.000 ns) | 16 % (160.000 ns) | 100 % (1.000.000 ns) |
| Motor 1 (izq.) | Antihorario | 30 % (300.000 ns) | 18 % (180.000 ns) | 100 % (1.000.000 ns) |
| Motor 2 (der.) | Horario | No medido | No medido | No medido |
| Motor 2 (der.) | Antihorario | No medido | No medido | No medido |

> Por debajo de ~30 % puede no arrancar desde parado.

### Encoders

| Parámetro | Valor |
|---|---|
| CPR (cuentas por vuelta, x4) | **1200** (provisional, falta medir dando 1 vuelta exacta) |
| `INVERTIR_ENC_IZQ` | `True` |
| `INVERTIR_ENC_DER` | Comprobar: si baja al avanzar → `True` |

### Sharp (A0)

| Distancia | Tensión media | ADC |
|---|---|---|
| 4 cm | 2,340 V | 725 |
| 5 cm | 2,025 V | 628 |
| 10 cm | 1,115 V | 346 |
| 15 cm | 0,625 V | 194 |
| 20 cm | 0,539 V | 167 |
| 25 cm | 0,482 V | 149 |

- Curva ajustada: **d = 10,41 · V^(−1,077)** (d en cm, V en voltios).
- En código: interpolación lineal sobre la tabla. Aviso de obstáculo a **< 10 cm**.
- No fiable por debajo de ~4 cm.

### CNY70 (media de 100 lecturas, ADC 0–1023)

| Sensor | Blanco | Negro | Umbral |
|---|---|---|---|
| Izquierdo (A1) | 970,5 | 485,7 | **728,1** |
| Derecho (A2) | 841,9 | 320,5 | **581,2** |

- **Por debajo del umbral = sobre la línea negra.**
- Usar un umbral **distinto para cada sensor**.
- Cara del sensor a ~3 mm del suelo (rango útil 0–5 mm).

---

## 7. Geometría

| Parámetro | Valor |
|---|---|
| Diámetro de rueda | 65 mm (R = 32,5 mm) |
| Ancho total con ruedas | ≈149 mm |
| Distancia entre ruedas (L) | **Por medir** (centro a centro de la banda de rodadura) |
| Avance por cuenta | 2π·R / CPR ≈ 0,17 mm (con CPR = 1200) |

---

## 8. Checklist antes de encender

- [ ] GND de la Bee unido a la línea − de pilas
- [ ] Jumpers ENA/ENB quitados, jumper de 5 V de la L298N puesto
- [ ] Tornillo +5V de la L298N vacío
- [ ] Encoders y CNY70 a 3,3 V; Sharp a 5 V
- [ ] Líneas de pilas y 3,3 V separadas
- [ ] Jumpers J2, J11, J14, J18 puestos y SPI activado
- [ ] Sin puntas peladas que se toquen

---

## Anexo: registro de cables por color (tal como lo anotaste)

| Cable | Conexión |
|---|---|
| Amarillo izquierda | GND |
| Negro derecha | PWM1 (ENA) |
| Amarillo derecha | IN1 → B17 |
| Azul derecha | IN2 → B27 |
| Verde derecha | IN3 → B22 |
| Negro derecha (más arriba) | PWM0 (ENB) |
| Verde derecha (más arriba) | IN4 → B23 |
| Azul derecha breadboard | 3,3 V |
| Verde derecha breadboard | GND Bee |
| Verde motor izquierdo | B26 |
| Amarillo motor izquierdo | B21 |
| Amarillo motor derecho | B20 |
| Naranja motor derecho | B16 |
| Naranja sensor izquierdo | J8 → A0 |
| Marrón breadboard derecho | J15 → 2.º desde el borde |
| Blanco breadboard derecho | J15 → 3.º desde el borde |
| Gris breadboard derecho | J15 → 4.º desde el borde |
| Morado breadboard derecho | J15 → 5.º desde el borde |
| Negro breadboard derecho | J12 → A1 |
| Azul breadboard derecho | J12 → A2 |
