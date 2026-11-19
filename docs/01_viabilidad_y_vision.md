# 1. Análisis de Viabilidad y Visión de WaCLI

## 1.1 Evaluación de la estructura piramidal original

La propuesta inicial plantea una pirámide de 4 niveles:

1. **Base**: constantes, funciones y utilidades para mover motores, leer sensores o reproducir sonidos.
2. **Procesos**: programas que combinan funciones de la base para adoptar comportamientos, ejecutar tareas o seguir estrategias.
3. **CLI**: interfaz de línea de comandos para invocar procesos, encadenarlos y combinar comportamientos.
4. **Cima (Agente de IA)**: agente tipo Claude Code que traduce lenguaje natural a llamadas del CLI o **escribe nuevos procesos en tiempo real**, adaptándose a errores.

### Qué acierta esta propuesta (y por qué es muy buena base)

- **Coincide con la arquitectura clásica de robótica móvil (arquitectura de 3 capas)**: en robótica profesional (ROS, robots autónomos) siempre se separan las primitivas de control rápido (*Behavioral*), la coordinación de tareas (*Executive*) y la planificación de alto nivel (*Deliberative*).
- **El CLI como contrato universal**: los agentes de código actuales (Claude Code, Codex, Gemini CLI, etc.) están extraordinariamente bien entrenados para usar herramientas de terminal, leer código fuente, editar archivos Python, interpretar trazas de error y reintentar. Usar un CLI en lugar de inventar un protocolo propietario es una decisión muy acertada.
- **Extensibilidad en tiempo real (*Embodied Code-as-Policies*)**: permitir que la IA no solo elija entre comandos predefinidos, sino que redacte nuevos comportamientos usando las primitivas del robot, convierte a WaCLI en un sistema abierto capaz de resolver tareas no previstas.

---

## 1.2 ¿Es viable que la IA se comporte como "un robot de verdad" que construye lo que necesita?

**Respuesta corta: Sí, es completamente viable**, siempre que se respete una regla de oro de la robótica:

> **El LLM nunca debe estar dentro del bucle de control en tiempo real.**
> El LLM es el *córtex* (piensa en segundos); el código Python en la Raspberry Pi es el *cerebelo y la médula espinal* (actúa en milisegundos).

### Por qué esta separación es crítica en nuestro hardware

Con los datos reales de [conexiones_robot.md](./conexiones_robot.md):

- Un agente de IA tarda entre **1 y 10 segundos** por decisión (enviar contexto, razonar, invocar herramienta, recibir respuesta).
- En 1 segundo, un robot con ruedas de 65 mm de diámetro puede desplazarse **20–40 cm**.
- El sensor **Sharp GP2Y0A41SK0F** solo mide de forma fiable entre **4 cm y 25 cm**, y el aviso de obstáculo está en **< 10 cm**.
- Los sensores **CNY70** necesitan correcciones a **50–100 Hz** para no salirse de una línea negra en curva.

Si la IA intentara leer sensores y decidir velocidades paso a paso desde su bucle de chat, el robot se chocaría o perdería la línea antes de que llegara la siguiente respuesta del modelo.

### Cómo SÍ funciona de forma real y fluida

La IA opera en un **bucle deliberativo de alto nivel**:

1. **Percibe resúmenes estructurados**: llama a `wacli sense` o `wacli scan` y recibe un JSON con distancias, estado de los CNY70 y odometría `(x, y, theta)`.
2. **Delega el control fino a comportamientos locales**: en vez de decir *"pon PWM izquierdo a 45% durante 0.1s"*, ejecuta *"sigue la línea negra a velocidad media hasta que el Sharp detecte < 10 cm o recorras 1 metro"*. Ese comportamiento corre localmente en la Raspberry Pi a 50 Hz.
3. **Crea nuevas habilidades cuando no existen**: si el usuario pide *"haz un baile en zigzag evitando obstáculos y pita al terminar"* y no existe ese proceso, la IA:
   - Lee la documentación/interfaz de primitivas del robot.
   - Escribe un nuevo script de comportamiento en `behaviors/custom/zigzag_dance.py`.
   - Lo valida primero en modo **simulación / dry-run** (comprobación de sintaxis y lógica).
   - Lo ejecuta en el robot real mediante `wacli run custom/zigzag_dance --timeout 15`.
   - Lee la telemetría devuelta (por ejemplo: *"detenido a los 4.2 s por obstáculo a 8.5 cm"* o *"motor derecho no arrancó al 25% PWM"*).
   - **Corrige el código** (sube PWM mínimo al 32% o cambia el ángulo de evasión) y vuelve a intentarlo.

---

## 1.3 Los 4 huecos de la pirámide original que debemos resolver

Para que la idea funcione en la práctica y no se rompa al primer uso, la pirámide original necesita cuatro refuerzos arquitectónicos:

| Problema en la pirámide simple | Qué pasaría en la vida real | Solución en WaCLI |
|---|---|---|
| **1. CLI efímero sin estado** | Si cada comando abre GPIO/SPI y luego cierra, **se pierden los encoders** entre comandos (sin odometría) y dos comandos simultáneos chocan por el hardware. | Un **Daemon local (`wacli-daemon`)** en la Raspberry Pi que mantiene abierto el hardware, lee encoders continuamente y atiende peticiones del CLI. |
| **2. Riesgo físico por código generado** | Si la IA escribe un bucle infinito o se cae el Wi-Fi, los motores siguen girando hasta estrellar el robot o agotar las pilas. | Una **Capa de Seguridad inmutable** dentro del daemon: *watchdog* de timeout obligatorio, parada de emergencia inmediata y reflejo anti-colisión del Sharp. |
| **3. Control remoto real** | El agente de IA (Claude Code) normalmente corre en un portátil/PC, mientras que los pines están en la Raspberry Pi. | Separar **Cliente CLI** (puede correr en tu PC o en la Pi) y **Servidor/Daemon** (corre en la Pi), comunicados por HTTP/WebSocket local (Wi-Fi) o túnel SSH. |
| **4. Pruebas a ciegas en hardware** | Cada error tonto de código de la IA provocaría golpes físicos o desgaste de pilas. | Una **Capa de Abstracción de Hardware (HAL)** con un **Simulador 2D ligero** (`--sim`) para validar comportamientos antes de pasarlos al robot real. |

Con estas cuatro piezas, la pirámide original no se descarta: **evoluciona hacia una arquitectura robusta, segura y verdaderamente agéntica**.
