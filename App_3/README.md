# App 3 – ThemePark Interrupts & Bottom-Half

## Theme

**ThemePark Ride Safety System**

This project extends my ThemePark real-time system by adding a hardware interrupt using a push button connected to GPIO 18. The interrupt follows the top-half/bottom-half design pattern where the ISR performs only the minimum amount of work required, while the bottom-half task handles the remaining processing.

---

# Wokwi Project

<<<PASTE YOUR WOKWI LINK HERE>>>

---

# Run Instructions

1. Open the Wokwi project.
2. Start the simulation.
3. Press the button connected to GPIO 18.
4. Observe the serial monitor.
5. GPIO 19 generates an ISR pulse for the logic analyzer.
6. Repeat at least 50 button presses for each run mode.
7. Repeat once with `WITH_LOAD = 0` and once with `WITH_LOAD = 1`.

---

# Interrupt Source

- GPIO 18 configured as interrupt input
- `gpio_install_isr_service()` initializes the interrupt service
- `gpio_isr_handler_add()` registers the interrupt handler
- ISR uses `IRAM_ATTR`
- GPIO 19 is toggled inside the ISR for timing measurements

---

# ISR Safety Rules

The ISR performs only short, non-blocking operations.

| ISR Code | Why it is safe |
|----------|----------------|
| Debounce check | Prevents repeated interrupts from one button press |
| Save timestamp | Used to measure interrupt latency |
| Toggle GPIO 19 | Generates a pulse for the logic analyzer |
| `xSemaphoreGiveFromISR()` | ISR-safe semaphore API |
| `vTaskNotifyGiveFromISR()` | ISR-safe task notification API |
| `portYIELD_FROM_ISR()` | Allows the scheduler to immediately run the awakened task |

### The ISR does **NOT**

- print to the serial monitor
- call `ESP_LOGI()`
- allocate memory
- perform long calculations
- wait on semaphores or mutexes
- call blocking functions

These operations are performed in the bottom-half tasks.

---

# Bottom-Half Tasks

## Binary Semaphore Task

Waits for the semaphore, calculates latency, updates the maximum latency, and logs the ride dispatch event.

## Direct Task Notification Task

Waits for the notification, calculates latency, updates the maximum latency, and logs the ride safety interrupt.

---

# Task Table

| Component | Priority | Core | Purpose |
|-----------|---------:|-----:|---------|
| ISR | Interrupt | Hardware | Handle button interrupt |
| Bottom-half (Semaphore) | 12 | 1 | Process dispatch event |
| Bottom-half (Notification) | 12 | 1 | Process ride safety event |
| Load Task A | 15 | 1 | Background compute load |
| Load Task B | 10 | 1 | Background compute load |
| Load Task C | 5 | 1 | Background compute load |
| Load Task D | 2 | 1 | Background compute load |

---

# Latency Measurements

## Idle (`WITH_LOAD = 0`)

| Method | Worst-Case Latency |
|---------|-------------------:|
| Direct Task Notification | **2263 µs** |
| Binary Semaphore | **2535 µs** |

## Loaded (`WITH_LOAD = 1`)

| Method | Worst-Case Latency |
|---------|-------------------:|
| Direct Task Notification | **12575 µs** |
| Binary Semaphore | **10371 µs** |

---

# Binary Semaphore vs Direct Task Notification

During the idle test, the direct task notification produced the lower worst-case latency.

Under background load, both signaling methods experienced higher latency because the higher-priority periodic task could delay execution of the bottom-half task.

My final implementation includes both signaling paths provided by the scaffold.

---

# Latency Under Load

The background load consisted of the four periodic tasks from App 2.

Results:

- Direct notification increased from **2263 µs** to **12575 µs**
  - **5.56× increase**

- Binary semaphore increased from **2535 µs** to **10371 µs**
  - **4.09× increase**

Load Task A runs at priority 15 while the bottom-half tasks run at priority 12. Because Task A has the higher priority, it can delay execution of the bottom-half tasks. Load Tasks B, C, and D have lower priorities and therefore cannot preempt the bottom-half once it begins executing.

---

# Induced Failure

I temporarily removed:

```cpp
portYIELD_FROM_ISR(higher_woken);
```

### Prediction

The interrupt would still wake the waiting task, but the task would not execute immediately after the ISR, increasing wake-up latency.

### Observation

The bottom-half task executed later than normal, producing higher latency. After verifying the behavior, I restored `portYIELD_FROM_ISR()`.

---

# Concurrency Diagram

<<<INSERT YOUR CONCURRENCY DIAGRAM IMAGE HERE>>>

or

```text
          GPIO18 Button
                │
                ▼
       ISR (IRAM_ATTR)
                │
      ┌─────────┴─────────┐
      │                   │
 Binary Semaphore   Direct Notification
      │                   │
      ▼                   ▼
 Bottom-half Task   Bottom-half Task
     Priority 12      Priority 12
            │
            ▼
 ThemePark Ride Control Logs

WITH_LOAD = 1

Load Task A (Priority 15)
Load Task B (Priority 10)
Load Task C (Priority 5)
Load Task D (Priority 2)
```

---

# Logic Analyzer

<<<INSERT YOUR LOGIC ANALYZER SCREENSHOT HERE>>>

The logic analyzer was connected as follows:

- Channel 0 → GPIO 18 (button)
- Channel 1 → GPIO 19 (ISR pulse)

The waveform was used to measure interrupt response time and bottom-half wake-up latency.

---

# AI Disclosure

ChatGPT was used to explain ESP-IDF interrupt concepts, review code, and assist with documentation formatting. All implementation, measurements, testing, latency values, and analysis were completed by the author.

---

# Files Included

- `main.c`
- `diagram.json`
- `README.md`
- `<<<LOGIC_ANALYZER_SCREENSHOT.png>>>`
