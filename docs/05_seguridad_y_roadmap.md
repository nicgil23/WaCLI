# 5. Seguridad Física, Calibraciones Pendientes y Hoja de Ruta

## 5.1 Seguridad física (obligatoria desde el día 1)

Controlar un robot real mediante código generado por una IA exige protecciones mecánicas y de software que no dependan de que la IA "se porte bien":

1. **Watchdog de ejecución**:
   - Todo comando de movimiento tiene un `timeout` máximo obligatorio (por defecto 10–15 s).
   - Si el cliente CLI se desconecta (pérdida de Wi-Fi o crash del agente), el daemon detiene los motores automáticamente.
2. **Parada de emergencia (`wacli stop` + señal `SIGINT/SIGTERM`)**:
   - Al cerrar el daemon o recibir `stop`, los pines `ENA/ENB` pasan a `0` inmediatamente y `IN1..IN4` a frenado o rueda libre segura.
3. **Reflejo anti-colisión en capa de seguridad**:
   - Si el Sharp lee `< 5.5 cm` (por encima del mínimo fiable de 4 cm) mientras el robot intenta avanzar hacia delante, `SafetyGuard` bloquea el avance hacia delante (pero sigue permitiendo girar o retroceder para salir del bloqueo).
4. **Detección de atasco (*Stall detection*)**:
   - Si se aplica PWM por encima de la zona muerta durante > 0.8 s y los encoders no registran ticks, el robot se detiene y devuelve `exit_code = 7` (`STALL_DETECTED`) para proteger el L298N y los engranajes de los motores N20.
5. **Protección eléctrica por software**:
   - *Slew-rate limiter* (aceleración progresiva en ~150 ms) para evitar picos de corriente en las 4 pilas AA y evitar cambios bruscos de `+100%` a `-100%` que dañan los motores.

---

## 5.2 Calibraciones pendientes detectadas en `conexiones_robot.md`

Antes de tener odometría y giros precisos, hay **4 parámetros físicos pendientes** que conviene medir al inicio del proyecto (y para los cuales WaCLI incluirá subcomandos `wacli calibrate`):

| Parámetro | Estado actual en `conexiones_robot.md` | Impacto en WaCLI |
|---|---|---|
| **Distancia entre ruedas (`L`)** | Por medir (centro a centro de la banda de rodadura) | Imprescindible para `turn_angle` y odometría `theta`. |
| **Motor derecho (zona muerta)** | Arranque/parada no medidos (horario y antihorario) | Necesario para que el robot vaya recto a baja velocidad y no derive. |
| **CPR exacto de encoders** | Provisional `1200` (x4) + verificar `INVERTIR_ENC_DER` | Necesario para que `move 30` avance realmente 30 cm. |
| **Zumbador** | Pendiente de confirmar pin/jumper seguro (no usar J3) | Bloquea el módulo de audio/música hasta verificar conexión segura. |

> [!TIP]
> El primer uso real del CLI puede ser precisamente el modo `wacli calibrate` para medir automáticamente la zona muerta del motor derecho y validar el sentido de `INVERTIR_ENC_DER` con el robot levantado del suelo.

---

## 5.3 Hoja de ruta de implementación (por fases)

Construir WaCLI de abajo arriba (respetando tu idea piramidal) permite validar cada capa en el robot real antes de añadir complejidad:

### Fase 0 — Base del repositorio y Calibración
- Inicializar `pyproject.toml`, `config/robot.yaml`, `config/calibration.yaml`.
- Implementar scripts de verificación rápida y `wacli calibrate` para completar los 4 datos pendientes de hardware.
- Validar el checklist de seguridad eléctrica de `conexiones_robot.md`.

### Fase 1 — HAL, Safety Guard y Primitivas (`hal/`, `safety/`, `core/`)
- Drivers de L298N (con compensación de zona muerta), encoders x4 y MCP3008 (Sharp + CNY70).
- `SafetyGuard` (watchdog, reflejo Sharp, detección de atasco, rampa de aceleración).
- Odometría diferencial + PID de rumbo para `move_distance` y `turn_angle`.

### Fase 2 — Daemon y CLI Remoto (`daemon/`, `cli/`)
- `wacli-daemon` corriendo como servicio en la Raspberry Pi con API HTTP/WebSocket y exclusión mutua.
- Cliente `wacli` con comandos `status`, `sense`, `move`, `turn`, `stop`, `capabilities` y modo `--json`.
- Probar control remoto desde el portátil por Wi-Fi.

### Fase 3 — Motor de Comportamientos y Simulador (`behaviors/`, `sim_robot.py`)
- Clase base `Behavior` + `BehaviorRunner` a 50 Hz con resumen de telemetría.
- Implementar comportamientos `builtin/`: `follow_line`, `avoid_obstacle`, `scan_360`.
- Implementar `SimRobot` 2D y flag `--sim` para validar comportamientos sin hardware.
- Implementar `validator.py` para auditar scripts en `behaviors_custom/`.

### Fase 4 — Integración Agente IA (`AGENTS.md` + `behaviors_custom/`)
- Redactar `AGENTS.md` / `CLAUDE.md` con el manual operativo del robot.
- Conectar Claude Code (u otro agente) desde el PC apuntando a `WACLI_HOST`.
- Pruebas de extremo a extremo:
  1. Misión en lenguaje natural usando comportamientos existentes.
  2. Misión nueva donde el agente diseña un script en `behaviors_custom/`, lo prueba con `--sim`, lo ejecuta en el robot real y lo ajusta según la telemetría.

### Fase 5 — Expansiones futuras (opcional)
- Integrar zumbador (sonidos de estado/música) una vez verificado el pin.
- Añadir cámara (Raspberry Pi Camera) o IMU por I2C (`B2/B3` están libres) para mejorar giros y dar visión al agente multimodal.
- Mapa 2D persistente generado a partir de múltiples `scan_360` + odometría.
