# Parámetros del robot para programar

Datos geométricos y eléctricos del robot (tracción diferencial, 2 motores con encoder, rueda loca, 2 CNY70 y 2 Sharp) para usarlos en el código de control, odometría y lectura de sensores.

> **De dónde salen los números.** Todas las cotas vienen del modelo de Fusion con el que se imprimió el chasis y de las medidas tomadas a cada componente. El robot montado puede diferir unos milímetros, sobre todo en la **vía** (distancia entre ruedas), que depende de cuánto entra cada rueda en el cabezal del motor. Lo marcado con ⚠ es una suposición o falta por medir. La sección [Calibración](#calibración) explica cómo afinar la vía y el diámetro efectivo con el robot real.

---

## Sistema de referencia

Origen en el **centro del eje de las ruedas, a ras de suelo**.

- **X**: hacia delante (el frente es el lado de los sensores).
- **Y**: hacia la izquierda del robot.
- **Z**: hacia arriba.
- **Ángulos**: positivos en sentido antihorario visto desde arriba (girar a la izquierda = ángulo positivo).

```
                    frente (+X)
          ┌──────── Sharp izq ─── Sharp dcha ────────┐   x ≈ +161,5 (lente)
          │               CNY izq  CNY dcha          │   x = +153,25
          │                                          │
          │                  ◯ rueda loca            │   x = +121
          │                                          │
          │                                          │
  rueda ██│                                          │██ rueda
   izq  ██│─────────────────── ✛ origen ─────────────│██ dcha   x = 0 (eje)
          │                                          │
          └──────────────────────────────────────────┘   x = −40 (borde trasero)
   +Y ←
```

---

## Resumen rápido

| Parámetro | Valor | Notas |
| --- | --- | --- |
| Diámetro de rueda | **65 mm** | Medido. Radio 32,5 mm |
| Circunferencia de rueda | **204,2 mm** | π · 65 |
| Ancho de rueda / banda de contacto | 7,5 mm / 3,5 mm | Medido |
| Vía (centro a centro de las ruedas) | **≈ 141,5 mm** ⚠ | Del diseño; medir en el robot montado y calibrar |
| Distancia eje → rueda loca | **121 mm** | Hacia delante |
| Distancia eje → CNY70 | **153,25 mm** | Hacia delante |
| Separación entre CNY70 | **24 mm** | Centro a centro (y = ±12 mm) |
| Altura de los CNY70 sobre el suelo | **≈ 3 mm** ⚠ | Depende del alto real de la cajita del motor |
| Distancia eje → lente de los Sharp | **≈ 161,5 mm** | Hacia delante |
| Separación entre Sharp | **56 mm** | Centro a centro (y = ±28 mm), los dos en paralelo mirando al frente |
| Altura de los Sharp (centro) | **≈ 58 mm** ⚠ | Sobre el suelo |
| Largo total del robot | 204 mm | De x = −40 a x = +164 |
| Ancho total con ruedas | ≈ 149 mm ⚠ | Base de 110 mm + ruedas por fuera |
| Altura total | ≈ 135 mm ⚠ | Hasta el asa de la placa de la universidad |
| Radio de giro sobre sí mismo | ≈ 172 mm | Punto más alejado del eje: esquinas del soporte de los Sharp |

---

## Ruedas y tracción

### Geometría

| Dato | Valor |
| --- | --- |
| Diámetro D | 65 mm |
| Radio r | 32,5 mm |
| Circunferencia C = π · D | 204,2 mm |
| Vía L (entre los centros de las bandas de contacto) | ≈ 141,5 mm ⚠ |
| Posición de cada rueda | y = +70,75 mm (izquierda) y y = −70,75 mm (derecha) ⚠ |
| Rueda loca | Bola de Ø16,5 mm, apoyo en (x = +121, y = 0) |

La vía sale de suponer que la rueda queda a ras del extremo del cabezal del motor. Si entra más o menos, cambia unos milímetros. Para giros precisos, usa el valor calibrado.

### Cinemática diferencial

Con `v_izq` y `v_dcha` como velocidades lineales de cada rueda (mm/s):

```
v     = (v_dcha + v_izq) / 2          # velocidad lineal del robot (mm/s)
omega = (v_dcha - v_izq) / L          # velocidad angular (rad/s), + = giro a la izquierda
R     = v / omega                     # radio de la curva que describe el centro del eje
```

Al revés, para conseguir una `v` y una `omega` dadas:

```
v_izq  = v - omega * L / 2
v_dcha = v + omega * L / 2
```

### Giros sobre sí mismo

Para girar un ángulo θ (en radianes) sobre el centro del eje, cada rueda recorre en sentidos opuestos:

```
s = θ · L / 2
```

| Giro | Recorrido de cada rueda | Vueltas de cada rueda |
| --- | --- | --- |
| 90° | ≈ 111,1 mm | ≈ 0,544 |
| 180° | ≈ 222,3 mm | ≈ 1,089 |
| 360° | ≈ 444,5 mm | ≈ 2,177 |

---

## Encoders

⚠ **Falta el dato más importante:** cuántos pulsos da el encoder por vuelta **de la rueda**. Si el encoder va en el eje del motor, antes de la reductora, hay que multiplicar por la relación de reducción. Si se cuentan los flancos de los dos canales (cuadratura x4), hay que multiplicar por 4.

```
pulsos_por_vuelta_rueda = pulsos_por_vuelta_motor × reduccion × multiplicador_cuadratura
mm_por_pulso            = C / pulsos_por_vuelta_rueda = 204,2 / pulsos_por_vuelta_rueda
```

Se puede medir sin hoja de datos: marca la rueda, gírala a mano exactamente 10 vueltas, lee los pulsos acumulados y divide entre 10.

### Odometría

En cada ciclo, con `dL` y `dR` como mm recorridos por cada rueda desde el ciclo anterior:

```
d       = (dR + dL) / 2
dtheta  = (dR - dL) / L
x      += d * cos(theta + dtheta / 2)
y      += d * sin(theta + dtheta / 2)
theta  += dtheta
```

---

## Sensores de línea CNY70

| Dato | Valor |
| --- | --- |
| Número | 2 |
| Posición | x = +153,25 mm; y = +12 mm (izquierdo) e y = −12 mm (derecho) |
| Separación centro a centro | 24 mm |
| Altura de la cara del sensor sobre el suelo | ≈ 3 mm ⚠ |
| Orientación | Mirando hacia abajo, en el borde delantero |
| Tamaño del sensor | 6,5 × 6,5 × 5,5 mm (medido) |

### Qué esperar de la lectura

- El CNY70 da más señal cuanto más cerca está del suelo. Según la hoja de datos de Vishay, el máximo está a menos de 0,5 mm, y por encima del 20 % de ese máximo trabaja entre 0 y 5 mm. A 3 mm la señal es más débil que el máximo, pero suficiente para distinguir blanco y negro.
- La superficie blanca refleja más que la negra. Que eso dé un valor alto o bajo depende de cómo esté cableado: con la resistencia de carga en el emisor del fototransistor, más luz da más tensión; con la resistencia en el colector, al revés. Compruébalo con una lectura de prueba.
- **Calibra siempre en la pista real:** lee cada sensor sobre blanco y sobre negro, y usa como umbral el punto medio de los dos valores, uno por sensor.

### Lógica básica con dos sensores

Con 24 mm entre sensores, el comportamiento depende del ancho de la línea ⚠:

- **Línea más estrecha que 24 mm** (los sensores van a los lados de la línea):
  - Ninguno ve negro: centrado.
  - El izquierdo ve negro: el robot se ha ido a la derecha, así que hay que girar a la izquierda.
  - El derecho ve negro: girar a la derecha.
  - Los dos ven negro: cruce o línea perpendicular.
- **Línea más ancha que 24 mm** (los dos sensores van encima): la lógica es la contraria; perder negro en un lado indica hacia dónde corregir.

El robot "ve" la línea 153 mm por delante del eje. Por eso reacciona con antelación, pero si corrige demasiado fuerte tiende a oscilar. Un control proporcional sobre la diferencia de lecturas suaviza el seguimiento.

---

## Sensores de distancia Sharp

| Dato | Valor |
| --- | --- |
| Número | 2 |
| Posición de la lente | x ≈ +161,5 mm; y = +28 mm (izquierdo) e y = −28 mm (derecho) |
| Separación centro a centro | 56 mm |
| Orientación | Los dos en paralelo, mirando al frente (+X) |
| Altura del centro sobre el suelo | ≈ 58 mm ⚠ |
| Frente físico del robot | x = +164 mm (placa del soporte, 2,5 mm por delante de la lente) |
| Tamaño | 44 × 18 × 15 mm (medido) |
| Modelo exacto | ⚠ Por confirmar |

### A tener en cuenta al programar

- **Distancia al frente del robot:** lo que mide el Sharp es la distancia desde su lente. Para saber cuánto falta para chocar, resta los 2,5 mm de la placa frontal. Para la distancia al centro del eje, suma 161,5 mm.
- **Zona ciega:** los Sharp analógicos tienen una distancia mínima por debajo de la cual la lectura deja de ser fiable y se confunde con objetos más lejanos. En el modelo más común, el GP2Y0A21, el rango útil es de unos 10–80 cm. Si es ese, un obstáculo a menos de unos 10 cm del frente dará lecturas engañosas, así que conviene parar antes de llegar ahí. ⚠ Confirmar con la hoja de datos del modelo real.
- **La salida no es lineal:** la tensión baja al aumentar la distancia, más o menos como 1/distancia. Lo práctico es medir la lectura a varias distancias conocidas (10, 15, 20, 30, 40, 60 cm) y ajustar una curva o hacer una tabla con interpolación.
- **Ruido:** la lectura salta bastante. Conviene promediar varias muestras o quedarse con la mediana de 5.
- **Para qué sirve tener dos:** si los dos ven algo a una distancia parecida, el obstáculo está de frente. Si uno lo ve más cerca, está hacia ese lado: gira hacia el contrario. Al ir en paralelo y separados 56 mm, entre los dos cubren casi todo el ancho de la base (110 mm), pero no las ruedas, que sobresalen hasta ±74,5 mm.

---

## Dimensiones y espacio que ocupa

| Dato | Valor |
| --- | --- |
| Base | 200 × 110 mm, de x = −40 a x = +160 |
| Frente del robot (placa de los Sharp) | x = +164 mm |
| Borde trasero | x = −40 mm |
| Ancho de la base | y = ±55 mm |
| Ancho exterior de las ruedas | y ≈ ±74,5 mm ⚠ |
| Radio barrido al girar sobre sí mismo | ≈ 172 mm (esquinas delanteras); por detrás y a los lados no pasa de ≈ 81 mm (el borde de las ruedas) |
| Altura libre bajo los motores | ≈ 23 mm ⚠ |
| Punto más bajo del chasis | Soporte de los CNY70, ≈ 3 mm sobre el suelo ⚠ |
| Altura total | ≈ 135 mm ⚠ |

Al girar sobre sí mismo, la parte delantera barre un círculo de unos 17 cm de radio alrededor del centro del eje. Antes de girar cerca de una pared, comprueba que no haya obstáculos dentro de ese círculo.

---

## Alimentación y electrónica

| Elemento | Detalle |
| --- | --- |
| Raspberry Pi | Alimentada por el power bank (USB). Solo se conecta a la placa Bee y a la alimentación |
| Placa Bee | Recibe casi todas las señales: sensores, encoders y control del L298N |
| Motores | Alimentados por el portapilas de 4 pilas a través del L298N |
| L298N | Controla dirección (IN1–IN4) y velocidad (ENA/ENB, PWM) de los dos motores |
| Masa | El L298N y la placa Bee deben compartir GND, o el control no funciona bien |

- **El L298N pierde tensión:** es un puente con transistores bipolares y se come del orden de 2 V. Con 4 pilas AA (unos 6 V con alcalinas nuevas, 4,8 V con recargables), a los motores les llegan unos 3–4 V. Eso limita la velocidad máxima, y además la velocidad baja según se gastan las pilas. Para recorrer distancias exactas, controla por encoders y no por tiempo.
- **Zona muerta del PWM:** por debajo de cierto ciclo de trabajo, los motores no arrancan. Mídela subiendo el PWM poco a poco y empieza siempre los movimientos por encima de ese valor.
- **Los dos motores no son idénticos:** con el mismo PWM, uno irá algo más rápido. Corrígelo con un control por encoders, o al menos con un factor de compensación medido.

### Pines (a rellenar)

| Señal | Pin en la Bee / GPIO |
| --- | --- |
| Motor izquierdo IN1 / IN2 / ENA | |
| Motor derecho IN3 / IN4 / ENB | |
| Encoder izquierdo A / B | |
| Encoder derecho A / B | |
| CNY70 izquierdo / derecho | |
| Sharp izquierdo / derecho | |

---

## Constantes para el código

```python
import math

# --- Geometría (mm) — origen en el centro del eje, X adelante, Y izquierda ---
WHEEL_DIAMETER_MM = 65.0
WHEEL_RADIUS_MM = WHEEL_DIAMETER_MM / 2
WHEEL_CIRCUMFERENCE_MM = math.pi * WHEEL_DIAMETER_MM      # 204.2
WHEEL_TRACK_MM = 141.5            # ⚠ calibrar con el robot real
CASTER_X_MM = 121.0

# --- Encoders ⚠ a rellenar ---
TICKS_PER_WHEEL_REV = None        # pulsos por vuelta de rueda (con reducción y cuadratura)
MM_PER_TICK = None                # = WHEEL_CIRCUMFERENCE_MM / TICKS_PER_WHEEL_REV

# --- CNY70 ---
CNY_X_MM = 153.25
CNY_LEFT_Y_MM = 12.0
CNY_RIGHT_Y_MM = -12.0
CNY_SPACING_MM = 24.0
CNY_HEIGHT_MM = 3.0               # ⚠ depende del alto real del motor

# --- Sharp ---
SHARP_X_MM = 161.5                # posición de la lente
SHARP_LEFT_Y_MM = 28.0
SHARP_RIGHT_Y_MM = -28.0
SHARP_SPACING_MM = 56.0
SHARP_TO_FRONT_MM = 2.5           # restar a la lectura para tener la distancia al frente
SHARP_MIN_RELIABLE_MM = 100       # ⚠ si es un GP2Y0A21; confirmar el modelo

# --- Envolvente ---
ROBOT_FRONT_X_MM = 164.0
ROBOT_REAR_X_MM = -40.0
ROBOT_HALF_WIDTH_MM = 74.5        # con ruedas
SPIN_RADIUS_MM = 172.0            # radio barrido al girar sobre sí mismo


def wheel_distance_for_spin(angle_rad: float) -> float:
    """mm que recorre cada rueda (en sentidos opuestos) para girar angle_rad sobre el eje."""
    return angle_rad * WHEEL_TRACK_MM / 2


def wheel_speeds(v_mm_s: float, omega_rad_s: float) -> tuple[float, float]:
    """Velocidades (izquierda, derecha) en mm/s para una velocidad lineal y angular dadas."""
    half = omega_rad_s * WHEEL_TRACK_MM / 2
    return v_mm_s - half, v_mm_s + half
```

---

## Calibración

Los valores de diseño sirven para empezar, pero la odometría mejora mucho si se calibran dos números con el robot real.

### 1. Diámetro efectivo de las ruedas

1. Marca en el suelo una recta de 1 m (o 2 m, mejor).
2. Haz que el robot avance en línea recta un número de pulsos conocido, o empújalo a mano a lo largo de la recta.
3. Mide la distancia real recorrida y anota los pulsos de cada encoder.
4. `mm_por_pulso_real = distancia_real / pulsos`. Si las dos ruedas dan valores distintos, guarda uno por rueda.

### 2. Vía efectiva

1. Con el diámetro ya calibrado, haz que el robot gire sobre sí mismo 10 vueltas completas (3600°) según la odometría.
2. Mide cuánto ha girado de verdad, por ejemplo marcando la dirección inicial con una cinta.
3. `L_real = L_diseño × (ángulo_ordenado / ángulo_real)`. Si gira de más, la vía real es mayor que la que usa el código.
4. Repite en el otro sentido y promedia.

### 3. Sensores

- **CNY70:** lectura sobre blanco y sobre negro de la pista real, y umbral en el punto medio, por sensor.
- **Sharp:** lecturas a 10, 15, 20, 30, 40 y 60 cm de una superficie plana y clara, y una tabla o curva de conversión por sensor.

---

## Datos que faltan

- [ ] Pulsos por vuelta del encoder, relación de reducción y si se cuenta en cuadratura.
- [ ] Alto real de la cajita del motor. De él dependen la altura de los CNY70, la de los Sharp y la altura libre.
- [ ] Vía real medida en el robot montado.
- [ ] Modelo exacto de los Sharp y su rango.
- [ ] Ancho de la línea de la pista.
- [ ] Asignación de pines en la placa Bee.
- [ ] Peso del robot montado (útil para estimar aceleraciones).
