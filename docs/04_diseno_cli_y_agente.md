# 4. Diseño del CLI *Agent-First* y Operativa del Agente IA

Para que un agente como Claude Code pueda controlar el robot con solvencia, el CLI no puede diseñarse solo para ojos humanos: debe ser **predecible, introspectivo, estructurado y seguro**.

---

## 4.1 Principios de diseño del CLI (`wacli`)

1. **Salida JSON determinista (`--json`)**: todo comando soporta `--json` (o `WACLI_OUTPUT=json`) devolviendo un esquema fijo con `ok`, `exit_code`, `reason`, `state` y `telemetry`.
2. **Códigos de salida semánticos**: permiten encadenar comandos y que el agente distinga un fallo de red de un obstáculo físico:
   - `0`: éxito / objetivo cumplido.
   - `1`: error interno o de ejecución del código.
   - `2`: detenido por obstáculo (Sharp).
   - `3`: detenido por timeout.
   - `4`: condición objetivo perdida (ej. perdió la línea negra).
   - `5`: parada de emergencia (`E-Stop`) o cancelación manual.
   - `6`: error de conexión con el daemon o hardware no disponible.
   - `7`: atasco detectado (*stall*: PWM aplicado pero encoders sin movimiento).
3. **Introspección nativa**: el agente puede preguntar al robot qué es capaz de hacer sin adivinar.
4. **Control remoto transparente**: mediante `--host` o variable `WACLI_HOST` (ej. `http://192.168.1.50:8000`), el mismo comando funciona desde el portátil o desde la propia Raspberry Pi.

---

## 4.2 Catálogo de comandos propuestos

### Estado, introspección y sensores
```bash
wacli status --json               # Conexión, modo (real/sim), comportamiento activo, batería/uptime
wacli capabilities --json         # Hardware disponible, rangos, comportamientos builtin y custom
wacli sense --json                # Lectura instantánea: Sharp (cm), CNY70 (raw + on_line), encoders, pose (x,y,theta)
wacli sense --watch --hz 5        # Stream continuo de sensores (útil para depuración)
wacli odom reset                  # Pone la pose (x, y, theta) a (0, 0, 0)
```

### Primitivas de movimiento (síncronas y acotadas)
```bash
wacli move 30 --speed 0.5 --timeout 5 --json        # Avanza 30 cm con PID de encoders
wacli turn 90 --speed 0.4 --timeout 4 --json        # Gira +90° (izquierda) sobre su eje
wacli stop                                          # Parada de emergencia inmediata (máxima prioridad)
```

### Comportamientos y composición
```bash
wacli behaviors list --json                         # Lista comportamientos builtin y custom
wacli run follow_line --timeout 20 --param speed=0.4 --json
wacli run scan_360 --json                           # Gira 360° y devuelve mapa polar de distancias Sharp
wacli run behaviors_custom/mi_busqueda.py --sim     # Prueba el script generado en el simulador
wacli run behaviors_custom/mi_busqueda.py --timeout 15 --json  # Lo ejecuta en el robot real
```

### Composición secuencial (Misiones / Pipelines)
El agente puede encadenar comandos directamente en bash (`&&`) o ejecutar un plan declarativo YAML/JSON para reducir llamadas de red:

```bash
wacli sequence --json << 'EOF'
steps:
  - cmd: move
    args: { distance_cm: 25, speed: 0.45 }
  - cmd: turn
    args: { angle_deg: 90 }
  - cmd: run
    args: { behavior: follow_line, timeout: 15 }
EOF
```

### Calibración interactiva
```bash
wacli calibrate cny70             # Guía o muestrea blanco/negro y actualiza calibration.yaml
wacli calibrate motors            # Sube PWM poco a poco detectando con encoders el umbral de arranque
wacli calibrate encoders          # Mide ticks reales por vuelta
```

---

## 4.3 Ejemplo de respuesta JSON optimizada para LLMs

En lugar de volcar 500 lecturas crudas que saturan la ventana de contexto del agente, el CLI devuelve un **resumen compacto con eventos clave**:

```json
{
  "ok": false,
  "exit_code": 2,
  "behavior": "follow_line",
  "reason": "OBSTACLE_DETECTED",
  "duration_s": 6.42,
  "start_pose": { "x_cm": 0.0, "y_cm": 0.0, "theta_deg": 0.0 },
  "end_pose": { "x_cm": 54.3, "y_cm": 12.1, "theta_deg": 18.5 },
  "distance_traveled_cm": 58.0,
  "final_sensors": {
    "sharp_cm": 8.7,
    "cny70_left": { "raw": 490, "on_line": true },
    "cny70_right": { "raw": 835, "on_line": false }
  },
  "events": [
    { "t": 2.1, "type": "LINE_CURVE_LEFT" },
    { "t": 6.42, "type": "OBSTACLE_DETECTED", "sharp_cm": 8.7 }
  ]
}
```

Con ese único JSON, el agente entiende perfectamente **qué hizo el robot, dónde está ahora, qué ven sus sensores y por qué se detuvo**.

---

## 4.4 Cómo construye la IA sus propios comportamientos en tiempo real

Para que la IA pueda escribir comportamientos fiables en `behaviors_custom/`, la API interna de `Behavior` debe ser **mínima, limpia y difícil de usar mal**:

```python
from wacli.behaviors.base import Behavior, BehaviorStatus, ControlCommand

class PatrolUntilObstacle(Behavior):
    """Avanza en zigzag suave hasta detectar un obstáculo a menos de 12 cm."""

    def setup(self, ctx):
        self.phase = 0.0

    def step(self, state, dt):
        if state.sharp_cm is not None and state.sharp_cm < 12.0:
            return self.finish(BehaviorStatus.OBSTACLE, reason="Objeto detectado < 12 cm")

        self.phase += dt
        omega = 0.6 if int(self.phase) % 2 == 0 else -0.6
        return ControlCommand(linear_v=0.15, angular_w=omega)
```

### Ventajas de esta interfaz para un agente de IA
- **La IA nunca toca pines GPIO ni PWM crudos**: devuelve un `ControlCommand(linear_v, angular_w)` (o `left_speed, right_speed` normalizado `-1..1`), y el núcleo se encarga de la zona muerta, la inversión del motor derecho y los límites de seguridad.
- **Validación previa (`validator.py`)**: antes de ejecutar código en `behaviors_custom/`, el daemon revisa el AST para comprobar que:
  - No importa librerías de hardware directo (`gpiozero`, `spidev`, `RPi.GPIO`) saltándose la capa de seguridad.
  - No realiza operaciones peligrosas de sistema (`os.system`, `subprocess`).
  - Implementa correctamente `step()` y no contiene bucles bloqueantes infinitos.
- **Ciclo de aprendizaje por error en tiempo real**:
  1. **Generar**: la IA escribe el archivo en `behaviors_custom/`.
  2. **Simular**: ejecuta `wacli run behaviors_custom/nuevo.py --sim --json`.
  3. **Ejecutar**: si pasa la simulación, lo lanza en el robot real con un `--timeout` prudente.
  4. **Diagnosticar y corregir**: si recibe `exit_code: 7` (atasco) o `end_pose` no coincide con lo esperado, lee la telemetría, ajusta parámetros/lógica y reintenta.

### El papel de `AGENTS.md` en la raíz del repositorio
Cuando se inicialice el repositorio, `AGENTS.md` (o `CLAUDE.md`) actuará como el **manual operativo del agente**, indicándole:
- Cómo comprobar conectividad (`wacli status --json`).
- Que siempre debe probar nuevos comportamientos con `--sim` antes de hardware real.
- Qué significan los límites físicos del robot (Sharp no fiable < 4 cm ni > 25 cm; CNY70 a ~3 mm del suelo; pilas AA con caída de tensión).
- Que solo debe crear/editar archivos en `behaviors_custom/`.
