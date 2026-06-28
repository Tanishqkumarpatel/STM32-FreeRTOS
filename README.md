# STM32-FreeRTOS-Learning

A structured, hands-on embedded systems learning project built on the **STM32 Nucleo-F446RE** using **FreeRTOS** (via CMSIS-V2). Each exercise builds on the previous, progressing from bare GPIO control to a multi-task real-time system with inter-task communication, synchronization, and structured debug logging.

Note: Where to find Implementaion? core/src/main.c in all the project folders. 

---

## Hardware

| Component | Details |
|---|---|
| MCU Board | STM32 Nucleo-F446RE |
| Processor | ARM Cortex-M4 @ 180MHz |
| Flash | 512KB |
| SRAM | 128KB |
| RTOS | FreeRTOS via CMSIS-V2 |
| Toolchain | STM32CubeIDE + STM32CubeMX + HAL |
| Debug | ST-Link (onboard) + UART2 @ 115200 baud |
| Components | Breadboard, LEDs, resistors, photoresistor |

---

## Project Structure

```
STM32-FreeRTOS-Learning/
├── 01-gpio-blink/          # GPIO output, HAL, clock enable, memory-mapped I/O
├── 02-button-interrupt/    # EXTI interrupt, ISR callback, software debouncing
├── 03-pwm-dimming/         # TIM2 PWM, PSC/ARR calculation, duty cycle fade
├── 04-freertos-multitask/  # FreeRTOS tasks, scheduler, priorities, osDelay [vTaskDelay]
├── 05-queues-producer-consumer/  # ISR-safe queue, xQueueSendFromISR, portYIELD_FROM_ISR
├── 06-semaphore-mutex/     # Priority inversion, priority inheritance, mutex vs semaphore
├── 07-uart-debug-logging/  # Structured logger, LOG_INFO/WARN/ERROR/DEBUG macros, mutex-protected printf
└── 08-adc-reading/         # ADC polling, voltage divider, photoresistor, float conversion
```

---

## Exercises

### 01 — GPIO Blink
**Concepts:** GPIO output configuration, HAL_GPIO_TogglePin, clock enable, memory-mapped I/O, STM32CubeMX setup

The first exercise confirms the full toolchain works end to end — CubeMX → CubeIDE → ST-Link → board. Blinks the onboard green LED (PA5/LD2) and traces the full chain from C code to physical hardware register write.

Key insight: `__HAL_RCC_GPIOA_CLK_ENABLE()` must be called before any GPIO register write — forgetting it causes silent failure because the peripheral bus is gated off by default.

---

### 02 — Button Interrupt
**Concepts:** EXTI interrupt, NVIC, ISR callback pattern, software debouncing with HAL_GetTick, volatile flag, deferred processing

Implements edge-triggered interrupt on PC13 (onboard blue button). Uses the HAL callback pattern (`HAL_GPIO_EXTI_Callback`) rather than writing directly in the ISR — keeping the handler short and delegating work to the main loop via a `volatile` flag.

Key insight: mechanical buttons bounce — a single press can fire 10-50 interrupts in milliseconds. Debouncing with a timestamp prevents false triggers. The `volatile` keyword is mandatory for variables shared between an ISR and main code — without it the compiler may cache the value and never see the ISR's update.

---

### 03 — PWM LED Dimming
**Concepts:** TIM2 CH1 PWM, PSC/ARR calculation, duty cycle, HAL_TIM_PWM_Start, __HAL_TIM_SET_COMPARE

Generates a 1kHz PWM signal on PA5 using TIM2. PSC and ARR values calculated from the APB1 timer clock (84MHz):

```
Timer frequency = 84,000,000 / ((PSC+1) × (ARR+1))
1000Hz = 84,000,000 / (84 × 1000) → PSC=83, ARR=999
```

Fades LED smoothly from 0% to 100% duty cycle and back by sweeping CCR1 from 0 to 999.

---

### 04 — FreeRTOS Multitask
**Concepts:** xTaskCreate, task states, scheduler, preemptive priorities, vTaskDelay vs HAL_Delay, task starvation

Two independent FreeRTOS tasks blink two LEDs at different rates simultaneously. Experiments performed:
- Equal vs different priorities — observing preemption
- Removing vTaskDelay from high priority task — demonstrates task starvation (lower priority task never runs)

Key insight: a task must call vTaskDelay (or another blocking function) periodically to yield CPU. A task that never blocks starves all lower priority tasks — the scheduler has no opportunity to switch.

---

### 05 — Queues: Producer-Consumer
**Concepts:** xQueueCreate, xQueueSendFromISR, xQueueReceive, portYIELD_FROM_ISR, BaseType_t, ISR-safe API

Button ISR (producer) sends a flash count to a queue. Consumer task blocks on the queue with portMAX_DELAY and blinks the LED N times when data arrives.

Key implementation details:
- ISR uses `xQueueSendFromISR` not `xQueueSend` — the ISR-safe variant
- `portYIELD_FROM_ISR(xHigherPriorityTaskWoken)` triggers immediate context switch if a higher priority task unblocked
- NVIC interrupt priority set to 5 — safely below `configMAX_SYSCALL_INTERRUPT_PRIORITY` so FreeRTOS API calls are permitted inside the ISR

---

### 06 — Semaphore vs Mutex: Priority Inversion
**Concepts:** osMutexNew, osSemaphoreNew, priority inversion, priority inheritance, shared resource protection

Three tasks at different priorities (HIGH/NORMAL/LOW) compete for a shared UART resource. Demonstrates the classic priority inversion scenario and how FreeRTOS mutex priority inheritance resolves it.

**Observed behavior:**
- With semaphore (no inheritance): MID task starves LOW indefinitely — HIGH waits forever. LED toggling at full speed (solid appearance).
- With mutex (priority inheritance): LOW gets temporarily boosted to HIGH's priority — finishes quickly — HIGH gets resource promptly. LED blinks visibly slower (MID gets less CPU time).

The LED behavior itself is the visual proof of priority inheritance working.

---

### 07 — Structured UART Debug Logger
**Concepts:** Macro-based logging, __io_putchar redirect, mutex-protected printf, osThreadGetName, HAL_GetTick, log levels

Built a reusable structured logging system:

```c
#define LOG_INFO(msg)  log_msg("INFO",  msg)
#define LOG_WARN(msg)  log_msg("WARN",  msg)
#define LOG_ERROR(msg) log_msg("ERROR", msg)
#define LOG_DEBUG(msg) log_msg("DEBUG", msg)
```

Every log line includes timestamp, task name, priority level, severity, and message:

```
[1234ms] [Task1] [24] [INFO]  Sensor read complete
[1235ms] [Task2] [24] [ERROR] Threshold exceeded
```

Mutex protection ensures printf is never called simultaneously from multiple tasks — preventing UART output corruption.

---

### 08 — ADC Reading: Photoresistor
**Concepts:** ADC1 polling mode, HAL_ADC_Start/PollForConversion/GetValue, voltage divider circuit, 12-bit resolution, float printf

Reads ambient light level using a photoresistor in a voltage divider with a 10kΩ fixed resistor. Raw 12-bit ADC value (0-4095) converted to voltage:

```c
float volt = ((float)adc_raw / 4095.0f) * 3.3f;
```

Circuit:
```
3.3V → 10kΩ → [PA0/A0 ADC pin] → photoresistor → GND
```

Covering the photoresistor increases resistance, drops voltage at the ADC pin, and produces a lower reading. Output streamed over UART every 500ms.

**Debugging note — stack overflow from float printf:**
Using `%f` in printf caused the task to silently stop after 2 readings. Root cause: `%f` formatting pulls in significantly larger newlib float routines at runtime, consuming far more stack than integer formats. The default `stack_size = 128 * 4` (512 bytes) was insufficient — increasing it to `1024 * 4` (4096 bytes) resolved the issue. This is a classic embedded bug: stack overflows on STM32 with FreeRTOS produce undefined behaviour with no error message, not a clean crash. Two fixes required:
- Increase `.stack_size` in the task attributes struct
- Add `-u _printf_float` linker flag to enable float support in newlib nano

---

## Key Concepts Demonstrated

| Concept | Exercise |
|---|---|
| Memory-mapped I/O, clock gating | 01 |
| EXTI interrupts, NVIC, ISR rules | 02 |
| Timer configuration, PWM, PSC/ARR math | 03 |
| FreeRTOS scheduler, task priorities, starvation | 04 |
| ISR-safe queue API, deferred interrupt processing | 05 |
| Priority inversion, priority inheritance, mutex vs semaphore | 06 |
| Structured logging, mutex-protected UART | 07 |
| ADC polling, voltage divider, sensor interfacing | 08 |

---

## Skills

**Languages:** C, C++
**RTOS:** FreeRTOS (CMSIS-V2)
**Peripherals:** GPIO, EXTI, TIM/PWM, UART, ADC
**Tools:** STM32CubeIDE, STM32CubeMX, HAL, ST-Link, PuTTY
**Concepts:** Real-time scheduling, inter-task communication, synchronization primitives, ISR design, hardware register access

---

## Setup

1. Clone this repo
2. Open any exercise folder in STM32CubeIDE via **File → Open Projects from File System**
3. Build with `Ctrl+B`
4. Flash via Run → Run (ST-Link must be connected)
5. Open PuTTY or any serial terminal at **COM port, 115200 baud** to view UART output

---
