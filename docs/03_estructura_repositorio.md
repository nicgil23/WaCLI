# 3. Estructura Propuesta del Repositorio WaCLI

Esta estructura separa claramente:
- **Código inmutable del núcleo** (hardware, seguridad, cinemática y daemon).
- **Configuración y calibración** (datos en YAML, nunca hardcodeados en los comportamientos).
- **Zona extensible** donde el agente de IA puede crear y refinar comportamientos en tiempo real sin riesgo de romper el driver ni la seguridad.

## 3.1 Árbol de directorios propuesto

```text
WaCLI/
├── AGENTS.md                       # Instrucciones y contrato del robot para el agente de IA
├── pyproject.toml                  # Dependencias, scripts de entrada (`wacli`, `wacli-daemon`), linters
├── README.md                       # Guía rápida de instalación y uso
├── config/
│   ├── robot.yaml                  # Pines BCM, geometría (R, L), inversiones, límites de seguridad
│   └── calibration.yaml            # Valores medidos: zonas muertas PWM, tabla Sharp, umbrales CNY70, CPR
├── docs/                           # Documentación de hardware y diseño (esta carpeta)
│   ├── README.md
│   ├── conexiones_robot.md
│   ├── 01_viabilidad_y_vision.md
│   ├── 02_arquitectura.md
│   ├── 03_estructura_repositorio.md
│   ├── 04_diseno_cli_y_agente.md
│   └── 05_seguridad_y_roadmap.md
├── src/
│   └── wacli/
│       ├── __init__.py
│       ├── config.py               # Carga y validación (Pydantic) de robot.yaml y calibration.yaml
│       ├── models.py               # Esquemas compartidos: RobotState, BehaviorResult, TelemetryEvent
│       ├── hal/                    # CAPA 1: Abstracción de hardware
│       │   ├── base.py             # Interfaces abstractas (RobotHAL, MotorHAL, SensorHAL)
│       │   ├── pi_motors.py        # L298N con PWMOutputDevice en ENA/ENB + zona muerta
│       │   ├── pi_encoders.py      # Decodificación x4 por interrupciones en B26/B21 y B16/B20
│       │   ├── pi_mcp3008.py       # SPI /dev/spidev0.0 -> Sharp (CH0) y CNY70 (CH1, CH2)
│       │   ├── pi_buzzer.py        # Futuro soporte de zumbador
│       │   ├── pi_robot.py         # Ensamblado de RealRobot
│       │   └── sim_robot.py        # SimRobot 2D para pruebas seguras sin hardware
│       ├── safety/                 # PILAR TRANSVERSAL: Seguridad inmutable
│       │   ├── guard.py            # Reflejo anti-colisión, límites de PWM, rampa de aceleración
│       │   └── watchdog.py         # Timeout obligatorio y parada de emergencia (E-Stop)
│       ├── core/                   # CAPA 2: Primitivas y cinemática
│       │   ├── odometry.py         # Pose (x, y, theta) y velocidades a partir de encoders
│       │   ├── pid.py              # Controlador PID reutilizable (velocidad, rumbo, línea)
│       │   ├── motion.py           # move_distance, turn_angle, arc, stop
│       │   └── perception.py       # Filtrado de Sharp, detección de línea, detección de atasco (stall)
│       ├── behaviors/              # CAPA 3: Comportamientos / Procesos
│       │   ├── base.py             # Clase base Behavior, contexto, ciclo setup/step/teardown
│       │   ├── runner.py           # Motor de ejecución a frecuencia fija (50 Hz) + captura telemetría
│       │   ├── validator.py        # Verificación estática (AST/imports) para scripts custom
│       │   └── builtin/            # Comportamientos estables del sistema
│       │       ├── move.py
│       │       ├── turn.py
│       │       ├── follow_line.py
│       │       ├── avoid_obstacle.py
│       │       ├── scan_360.py
│       │       └── calibrate.py
│       ├── daemon/                 # SERVIDOR EN RASPBERRY PI
│       │   ├── server.py           # API HTTP/WebSocket (FastAPI/uvicorn) + lock de hardware
│       │   └── service.py          # Gestión de ciclo de vida, limpieza de GPIO en SIGTERM/SIGINT
│       └── cli/                    # CAPA 4: Interfaz CLI Agent-First
│           ├── main.py             # Punto de entrada `wacli` (Typer)
│           ├── client.py           # Cliente HTTP/WS hacia el daemon (local o remoto)
│           └── formatters.py       # Salida legible para humanos y modo `--json` para IA
├── behaviors_custom/               # ZONA EDITABLE POR LA IA (procesos creados en tiempo real)
│   ├── README.md                   # Plantilla y reglas para escribir comportamientos
│   └── .gitkeep
├── scripts/
│   ├── install_pi.sh               # Instalación de dependencias y servicio systemd en la Raspberry Pi
│   └── wacli-daemon.service        # Unidad systemd para arrancar el daemon al encender el robot
└── tests/
    ├── test_kinematics.py          # Tests unitarios de odometría, PID, curva del Sharp
    ├── test_safety.py              # Tests de watchdog, reflejo anti-colisión y E-Stop
    ├── test_behaviors_sim.py       # Pruebas de comportamientos sobre SimRobot
    └── test_cli.py                 # Tests del contrato JSON y códigos de salida del CLI
```

---

## 3.2 Decisiones clave de diseño del repositorio

### 1. ¿Por qué `behaviors_custom/` está en la raíz y separado de `src/wacli/`?
- Da a la IA un **espacio de trabajo claro y acotado**.
- Facilita reglas de permisos/revisión (por ejemplo, en `AGENTS.md` se indica: *"crea y modifica comportamientos únicamente en `behaviors_custom/`, nunca alteres `src/wacli/safety/` ni `src/wacli/hal/`"*).
- Permite enviar un archivo de `behaviors_custom/` al daemon remoto con `wacli run behaviors_custom/mi_estrategia.py` sin tener que reinstalar el paquete en la Raspberry Pi.

### 2. Configuración declarativa (`config/robot.yaml` + `config/calibration.yaml`)
Separar el cableado físico de las calibraciones tiene dos ventajas enormes:
- `robot.yaml` rara vez cambia (pines BCM de la Bee 2.0, canales SPI, diámetro de rueda).
- `calibration.yaml` cambia según el suelo, la luz ambiental (CNY70) o la carga de las pilas (zona muerta PWM). Incluso la IA puede lanzar `wacli calibrate cny70` para actualizar umbrales antes de una misión.

Ejemplo orientativo de `config/calibration.yaml`:

```yaml
motors:
  pwm_freq_hz: 1000
  left:
    invert: false
    min_start_cw: 0.28
    min_stop_cw: 0.16
    min_start_ccw: 0.30
    min_stop_ccw: 0.18
  right:
    invert: true
    min_start_cw: 0.30   # pendiente de medir
    min_stop_cw: 0.18    # pendiente de medir
    min_start_ccw: 0.30  # pendiente de medir
    min_stop_ccw: 0.18   # pendiente de medir

encoders:
  cpr_x4: 1200           # provisional
  invert_left: true
  invert_right: false    # comprobar

geometry:
  wheel_diameter_mm: 65.0
  wheel_base_mm: 130.0   # pendiente de medir (centro a centro)

sensors:
  sharp:
    obstacle_threshold_cm: 10.0
    emergency_stop_cm: 5.5
    calibration_table:
      - { volts: 2.340, cm: 4.0 }
      - { volts: 2.025, cm: 5.0 }
      - { volts: 1.115, cm: 10.0 }
      - { volts: 0.625, cm: 15.0 }
      - { volts: 0.539, cm: 20.0 }
      - { volts: 0.482, cm: 25.0 }
  cny70:
    left_threshold: 728.1
    right_threshold: 581.2
```

### 3. Stack tecnológico recomendado

| Componente | Tecnología | Motivo |
|---|---|---|
| Lenguaje | **Python 3.11+** | Ya compatible con `gpiozero` y `spidev` en Raspberry Pi, y es el mejor lenguaje para que los LLMs generen código fiable en tiempo real. |
| Hardware Pi | `gpiozero` (backend `lgpio` en Pi 4/5) + `spidev` | Coincide exactamente con las notas de `conexiones_robot.md` (`PWMOutputDevice`, `DigitalInputDevice`, `/dev/spidev0.0`). |
| CLI | **Typer** (sobre Click) | Tipado limpio, ayuda autogenerada clara para el agente y fácil soporte de `--json`. |
| Validación de datos | **Pydantic v2** | Esquemas estrictos para `robot.yaml`, API del daemon y respuestas JSON del CLI. |
| Daemon / API | **FastAPI + Uvicorn** (o `aiohttp`) | Ligero en Raspberry Pi, soporta REST para comandos y WebSockets para telemetría en vivo o cancelación instantánea. |
| Empaquetado | **uv** o **pip** con `pyproject.toml` | Instalación reproducible con extras opcionales: `pip install .[pi]` en la Raspberry Pi y `pip install .[client]` en el PC del agente. |
