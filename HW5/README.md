# HW 5 – Interrupts

**Pairs with:** Week 5 Deck (Interrupts) · Interrupt Response Time Handout · Application 3

---

# Part A — Latency Budget

## Given

- Peripheral A maximum response time = **100 µs**
- Peripheral A ISR = **30 µs**
- Peripheral B ISR = **25 µs**
- Peripheral B critical section = **20 µs**

### Worst-Case Interrupt Response Time

The worst-case response time for Peripheral A :

- Peripheral B critical section
- Peripheral B ISR
- Peripheral A ISR

Calculation:

```text
Worst-case response time
= 20 µs + 25 µs + 30 µs
= 75 µs
```

### Does Peripheral A meet the requirement?

Maximum allowed response:

```text
100 µs
```

Actual response:

```text
75 µs
```

Slack:

```text
100 µs − 75 µs = 25 µs
```

**Answer:** Yes. Peripheral A meets the 100 µs response requirement with **25 µs of slack**.

---

# Part B — ISR / Task Split

Assumption:

- GPIO 4 is connected to a push button.
- A button press requires a 3 ms I²C read followed by a log message.
- The ISR only wakes a task.
- The bottom-half task performs the I²C transaction and logging.

## Top-Half ISR

```c
#define BTN_GPIO GPIO_NUM_4

static SemaphoreHandle_t button_sem;

static void IRAM_ATTR gpio4_isr(void *arg)
{
    BaseType_t higher_woken = pdFALSE;

    xSemaphoreGiveFromISR(button_sem, &higher_woken);

    portYIELD_FROM_ISR(higher_woken);
}
```

### ISR Safety Rules

| Rule | Explanation |
|------|-------------|
| `IRAM_ATTR` | Places the ISR in internal RAM to reduce interrupt latency. |
| `xSemaphoreGiveFromISR()` | ISR-safe API used to wake a task. |
| `portYIELD_FROM_ISR()` | Allows the scheduler to immediately switch to the awakened task. |
| No I²C | I²C transactions take several milliseconds and should not occur in an ISR. |
| No `printf()` | Printing can block and uses internal locks. |
| No dynamic memory | `malloc()` is not deterministic inside an ISR. |
| No delays | ISRs must never sleep or block. |

## Bottom-Half Task

```c
static void button_bottom_half_task(void *arg)
{
    uint8_t data[8];

    while (1)
    {
        if (xSemaphoreTake(button_sem, portMAX_DELAY) == pdTRUE)
        {
            i2c_master_read_from_device(
                I2C_NUM_0,
                0x50,
                data,
                sizeof(data),
                pdMS_TO_TICKS(3));

            ESP_LOGI("GPIO4",
                     "Button pressed - I2C read completed.");
        }
    }
}
```

The ISR only wakes the task. The task then performs the I²C read and logging because those operations take much longer and should not be done inside an interrupt.
---

# Part C — Identify the Broken ISR

## Original ISR

```c
static int g_count;
static char *g_msg;

void button_isr(void) {
    printf("Button pressed!\n");
    g_count++;
    g_msg = malloc(64);
    snprintf(g_msg, 64, "Pressed %d times", g_count);
    vTaskDelay(pdMS_TO_TICKS(10));
    xSemaphoreTake(some_mutex, portMAX_DELAY);
    do_complex_thing();
    xSemaphoreGive(some_mutex);
}
```

## Problems and Fixes

| Bug | Why it is a problem | Fix |
|------|---------------------|-----|
| `printf()` inside ISR | Can block and uses internal locks | Move logging to a task |
| `malloc()` inside ISR | Dynamic allocation is slow and unpredictable | Allocate memory outside the ISR |
| `snprintf()` inside ISR | Too much processing for an ISR | Format strings inside the task |
| `vTaskDelay()` | ISRs cannot block or sleep | Remove delay from ISR |
| `xSemaphoreTake()` | May block forever | Never take a semaphore inside an ISR |
| Mutex usage | Mutexes are for tasks, not interrupts | Wake a task instead |
| `do_complex_thing()` | Long execution increases interrupt latency | Move work to bottom-half |
| Global variable update | Shared variables may create race conditions | Protect shared data or update in task |
| Repeated `malloc()` | Can leak memory and fragment the heap | Use static memory |
| Missing `IRAM_ATTR` | ISR may execute from flash and increase latency | Add `IRAM_ATTR` |

## Correct ISR

```c
static void IRAM_ATTR button_isr(void *arg)
{
    BaseType_t higher_woken = pdFALSE;

    xSemaphoreGiveFromISR(button_sem, &higher_woken);

    portYIELD_FROM_ISR(higher_woken);
}
```

The corrected ISR is much shorter. It simply wakes the task and lets the task handle everything else, which keeps interrupt latency low.
---

# Part D — ARM Cortex-M NVIC Summary

The ARM Cortex-M NVIC controls interrupt priorities and lets higher-priority interrupts run before lower-priority ones. Some Cortex-M processors also use subpriority to decide which interrupt runs first when two interrupts have the same preemption priority. In FreeRTOS, `configMAX_SYSCALL_INTERRUPT_PRIORITY` sets a limit on which interrupts can safely use FreeRTOS API functions. Higher-priority interrupts should not call the FreeRTOS API so they can respond as quickly as possible, while lower-priority interrupts can use ISR-safe functions like `xSemaphoreGiveFromISR()`. This helps keep the system responsive while protecting the RTOS.
