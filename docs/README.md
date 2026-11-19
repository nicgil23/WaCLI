# WaCLI — Documentación de Diseño

**WaCLI** es una plataforma robótica modular y *agent-first* diseñada para que tanto humanos como agentes de IA (por ejemplo, Claude Code u otros agentes con acceso a terminal) puedan controlar de forma remota un robot diferencial físico basado en **Raspberry Pi + placa Bee 2.0**, interpretar sus sensores y **construir nuevos comportamientos en tiempo real**.

> [!NOTE]
> Este proyecto se encuentra actualmente en **fase de diseño arquitectónico**. Los documentos de esta carpeta definen la visión, la arquitectura, la estructura del futuro repositorio y la hoja de ruta antes de escribir el código fuente.

## Índice de documentos

| Documento | Contenido |
|---|---|
| [conexiones_robot.md](./conexiones_robot.md) | **Especificación de hardware actual**: pines BCM, alimentación, L298N, encoders, MCP3008, Sharp, CNY70, calibraciones medidas y pendientes. |
| [01_viabilidad_y_vision.md](./01_viabilidad_y_vision.md) | **Análisis de la idea piramidal y viabilidad**: qué funciona, qué retos físicos impone el mundo real a un agente de IA y cómo resolverlos. |
| [02_arquitectura.md](./02_arquitectura.md) | **Arquitectura propuesta**: evolución de la pirámide en 5 capas + Daemon de hardware, capa de seguridad, abstracción de hardware (HAL/Simulador) y control remoto. |
| [03_estructura_repositorio.md](./03_estructura_repositorio.md) | **Estructura del repositorio**: árbol de carpetas, responsabilidades de cada módulo, separación núcleo vs. código generado por IA y stack tecnológico recomendado. |
| [04_diseno_cli_y_agente.md](./04_diseno_cli_y_agente.md) | **Diseño del CLI y operativa del Agente IA**: comandos, salidas JSON, códigos de error, telemetría compacta y ciclo de creación dinámica de comportamientos. |
| [05_seguridad_y_roadmap.md](./05_seguridad_y_roadmap.md) | **Seguridad física, calibraciones pendientes y hoja de ruta**: protecciones obligatorias y plan de implementación incremental por fases. |

## Resumen del hardware objetivo

Basado en [conexiones_robot.md](./conexiones_robot.md):

- **Cómputo**: Raspberry Pi + placa Bee 2.0 (Linux, pines BCM).
- **Locomoción**: Tracción diferencial con 2 motores N20 + driver **L298N** (PWM en `ENA/ENB` a 1 kHz, zona muerta ~28–30 %).
- **Propiocepción**: 2 encoders en cuadratura (decodificación x4, ~1200 CPR provisional, rueda Ø 65 mm).
- **Exterocepción (vía MCP3008 por SPI)**:
  - 1 sensor infrarrojo de distancia **Sharp GP2Y0A41SK0F** (rango fiable 4–25 cm, obstáculo < 10 cm).
  - 2 sensores ópticos de suelo **CNY70** (detección de línea negra con umbral independiente por sensor).
- **Actuadores futuros**: Zumbador (pendiente de confirmar pin/jumper seguro) y posibilidad de añadir sensores I2C (pines `B2/B3` libres).
