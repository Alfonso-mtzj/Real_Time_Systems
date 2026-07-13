
# HW 6 – Synchronization

## Part A — Spot the Race

### Snippet 1

```c
static int balance = 100;

void deposit(int amount) {
    balance += amount;
}
```

**Race Condition:** Yes.

This is a **lost update** race condition. The statement `balance += amount` is actually three operations (read, modify, write). If two tasks execute it at the same time, one update can overwrite the other.

**Fix:** Use an **atomic operation** (preferred for a single integer) or a **mutex**.

```c
#include <stdatomic.h>

static atomic_int balance = 100;

void deposit(int amount) {
    atomic_fetch_add(&balance, amount);
}
```

---

### Snippet 2

```c
static volatile struct sensor_reading_t latest_reading;

void IRAM_ATTR sensor_isr(void) {
    latest_reading.x = read_x();
    latest_reading.y = read_y();
    latest_reading.z = read_z();
    latest_reading.timestamp = esp_timer_get_time();
}

void main_task(void) {
    process(latest_reading);
}
```

**Race Condition:** Yes.

This is a **torn read**. The ISR updates the structure one field at a time while another task may read it in the middle of the update. `volatile` only prevents compiler optimization; it does **not** make the structure update atomic.

**Fix:** Copy the structure inside the same critical section used by the ISR, then process the copy.

```c
static portMUX_TYPE spin = portMUX_INITIALIZER_UNLOCKED;

void main_task(void) {
    struct sensor_reading_t snapshot;

    taskENTER_CRITICAL(&spin);
    snapshot = latest_reading;
    taskEXIT_CRITICAL(&spin);

    process(snapshot);
}
```

---

### Snippet 3

```c
static SemaphoreHandle_t mux;

void worker(void *p) {
    for (;;) {
        if (xSemaphoreTake(mux, 0) == pdTRUE) {
            update_state();
            // no give
            vTaskDelay(pdMS_TO_TICKS(10));
        }
    }
}
```

**Race Condition:** Yes.

This is **self-lockout** because the mutex is taken but never released. The task also delays while holding the mutex, increasing blocking time.

**Fix:** Release the mutex before delaying.

```c
void worker(void *p) {
    for (;;) {
        if (xSemaphoreTake(mux, portMAX_DELAY) == pdTRUE) {
            update_state();
            xSemaphoreGive(mux);
        }

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

---

# Part B — Choose Your Primitive

### 1. ISR signals a task that one button press happened

**Primitive:** Direct task notification

**Why:** It is the fastest and lightest way for an ISR to wake a single task.

---

### 2. A single global integer that Core 0 task and Core 1 task both increment

**Primitive:** Atomic operation

**Why:** The increment is a read-modify-write operation, so an atomic operation prevents lost updates without using a mutex.

---

### 3. A pool of 4 DMA buffers; producer borrows, consumer returns

**Primitive:** Counting semaphore

**Why:** The semaphore count represents the number of available buffers. Taking one borrows a buffer, and giving one returns it.

---

### 4. Wait for all 3 sensor tasks to finish their first read before starting fusion

**Primitive:** Event group

**Why:** Each sensor task sets its own event bit, and the fusion task waits until all bits are set.

---

### 5. A long-running task should be killable from another task

**Primitive:** Direct task notification

**Why:** Another task can send a notification telling the worker to exit cleanly and delete itself.

---

# Part C — Priority Inversion Forensics

Tasks:

- H (Priority 15) – Sensor handler
- M (Priority 10) – Communications relay
- L (Priority 5) – Telemetry logger

H and L share a FreeRTOS mutex.

### Timeline

1. L starts running and locks the mutex.
2. L enters its critical section.
3. H becomes ready and preempts L.
4. H tries to lock the mutex but blocks because L owns it.
5. L resumes execution.
6. M becomes ready and preempts L because M has higher priority than L.
7. M runs while H remains blocked waiting for the mutex.
8. With priority inheritance enabled, L temporarily inherits H's priority.
9. M can no longer preempt L.
10. L finishes its critical section and releases the mutex.
11. L returns to its original priority.
12. H immediately acquires the mutex and continues running.

### Worst-Case Wait for H

Without priority inheritance, H's wait is **unbounded** because M can continue delaying L.

With priority inheritance enabled, H waits for **at most one critical section of L**, plus a small amount of scheduling and mutex overhead.

---

# Part D — Industry Anchor

Before software is released, developers use different methods to find priority inheritance bugs. Static analysis tools like Coverity and Klocwork can catch problems such as missing mutex releases, locking mistakes, or possible deadlocks before the program is even tested. Runtime tracing tools like Tracealyzer and SystemView let developers see how tasks actually run on the hardware, making it easier to spot when a high-priority task is being blocked by a lower-priority one. Code reviews also help make sure mutexes are released correctly, critical sections are kept short, and the correct APIs are used in ISRs. For safety-critical systems, tools like TLA+, SPIN, and UPPAAL can test different task interactions to check for deadlocks and blocking issues. Using these methods together helps developers catch race conditions and priority inversion problems before the software is released.
