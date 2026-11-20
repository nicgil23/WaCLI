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

## Datos que faltan

- [ ] Pulsos por vuelta del encoder, relación de reducción y si se cuenta en cuadratura.
- [ ] Alto real de la cajita del motor. De él dependen la altura de los CNY70, la de los Sharp y la altura libre.
- [ ] Vía real medida en el robot montado.
- [ ] Modelo exacto de los Sharp y su rango.
- [ ] Ancho de la línea de la pista.
- [ ] Asignación de pines en la placa Bee.
- [ ] Peso del robot montado (útil para estimar aceleraciones).
