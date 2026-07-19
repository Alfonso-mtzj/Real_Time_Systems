
# App 4 – Synchronization Quest

## Theme Park Control System

**Student:** Alfonso Martinez  
**Course:** Real-Time Systems – Summer 2026  
**Platform:** ESP32-S3 with FreeRTOS  
**Wokwi Project:** [https://wokwi.com/projects/469948224187674625]

---

## Project Overview

This project demonstrates several FreeRTOS synchronization techniques using a theme park control system. The application includes an interrupt-driven emergency alert, inspection teams competing for maintenance bays, safety stations updating a shared inspection counter, and a priority-inversion experiment using high-, medium-, and low-priority tasks.

The synchronization primitives used are:

- A binary semaphore for ISR-to-task event signaling
- A counting semaphore for managing three maintenance bays
- A mutex for protecting the shared inspection counter
- A selectable mutex or binary semaphore for demonstrating priority inversion

All output is displayed through the Wokwi serial monitor.

---

## Theme Refactor

The original scaffold was changed to use a theme park control system.

| Original role | Theme park role |
|---|---|
| Responder task | Emergency responder |
| Pool consumers | Inspection teams |
| Resource slots | Maintenance bays |
| Writer tasks | Safety stations |
| Shared integer | Number of cleared inspections |
| ISR event | Operator emergency alert |

Example themed output:

```text
[emergency-responder] operator alert received - safety team notified
[inspection-team#1] entered a maintenance bay - inspecting ride...
[inspection-team#1] released maintenance bay
[safety-station#1] cleared inspections 0 -> 1
```

---

# Synchronization Primitive Analysis

## 1. Binary Semaphore – Emergency Alert Signal

The binary semaphore is used to send an event from the GPIO 18 interrupt service routine to the emergency responder task.

The ISR gives the semaphore using:

```c
xSemaphoreGiveFromISR(sig_sem, &woken);
```

The responder waits for it using:

```c
xSemaphoreTake(sig_sem, portMAX_DELAY);
```

When the button is pressed, the responder prints:

```text
[emergency-responder] operator alert received - safety team notified
```

### Why a binary semaphore is appropriate

A binary semaphore works well because this path represents a single event: an operator pressed the emergency button and the responder needs to wake up.

The ISR should remain short. It only acknowledges that an event happened and wakes the task responsible for performing the actual work.

A binary semaphore can also be given safely from interrupt context using `xSemaphoreGiveFromISR()`.

### What would happen with a counting semaphore?

A counting semaphore could store several button events if multiple interrupts occurred before the responder processed them. That may be useful when every individual event must be counted, but it is unnecessary for this project.

The emergency responder only needs to know that an alert is pending. Using a counting semaphore could also cause the responder to process many duplicate events caused by button bounce.

### Signal flow

```text
GPIO 18 button
      |
      v
Button ISR
      |
      | xSemaphoreGiveFromISR()
      v
Binary semaphore
      |
      | wakes blocked task
      v
Emergency responder task
```

---

## 2. Counting Semaphore – Maintenance Bay Pool

The counting semaphore represents three available maintenance bays.

It is created with:

```c
pool_sem = xSemaphoreCreateCounting(3, 3);
```

Four inspection-team tasks compete for these three spaces. Each task must take one semaphore token before entering a bay.

```c
xSemaphoreTake(pool_sem, portMAX_DELAY);
```

When an inspection is complete, the task returns the token:

```c
xSemaphoreGive(pool_sem);
```

Example output:

```text
[inspection-team#1] entered a maintenance bay - inspecting ride...
[inspection-team#2] entered a maintenance bay - inspecting ride...
[inspection-team#3] entered a maintenance bay - inspecting ride...
[inspection-team#3] released maintenance bay
[inspection-team#4] entered a maintenance bay - inspecting ride...
```

The output shows that three teams can enter immediately. The fourth team must wait until another team releases a maintenance bay.

### Why the semaphore count is three

The count is three because the simulated maintenance area contains three identical bays.

Changing the count would change the amount of allowed concurrency:

| Count | Result |
|---:|---|
| 2 | Only two inspection teams could work at once |
| 3 | Three teams can work and the fourth waits |
| 4 | Every team could enter, so no task would normally block |

A count of three clearly demonstrates resource contention because there are four consumers but only three available resources.

### What if separate binary semaphores were used?

Three separate binary semaphores could represent three individual bays, but each inspection task would need extra logic to search for an available semaphore.

This could create several problems:

- Tasks may repeatedly check the same unavailable bay.
- Resource selection becomes more complicated.
- Fairness becomes harder to manage.
- More code is required to track which bay was taken.

The counting semaphore provides one shared count for all identical bays, making it simpler and more appropriate.

### Pool flow

```text
                  Counting semaphore
                  Initial count = 3
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
Inspection #1      Inspection #2    Inspection #3
enters bay         enters bay       enters bay

Inspection #4 waits until one token is returned.
```

---

## 3. Mutex – Shared Inspection Counter

Two safety-station tasks update the same shared integer:

```c
static int shared_state = 0;
```

The variable represents the number of completed and cleared ride inspections.

The update is a read-modify-write operation:

```c
int old = shared_state;
shared_state = old + 1;
```

The operation must be protected because both tasks could otherwise read the same old value and overwrite each other's updates.

Each writer takes the mutex before accessing the shared value:

```c
xSemaphoreTake(shared_mux, portMAX_DELAY);
```

It releases the mutex after finishing:

```c
xSemaphoreGive(shared_mux);
```

Example protected output:

```text
[safety-station#1] cleared inspections 0 -> 1
[safety-station#2] cleared inspections 1 -> 2
[safety-station#1] cleared inspections 2 -> 3
[safety-station#2] cleared inspections 3 -> 4
```

The counter increases by exactly one during every update.

### Why a mutex is appropriate

A mutex is designed for mutual exclusion and has ownership. The task that takes the mutex is expected to release it.

It also supports priority inheritance. If a high-priority task blocks on a mutex owned by a low-priority task, the low-priority owner can temporarily inherit the higher priority.

### What if a binary semaphore were used as a lock?

A binary semaphore may prevent simultaneous access, but it has no ownership and no priority inheritance.

This creates two main risks:

1. A task that did not take the semaphore could incorrectly give it.
2. A low-priority task holding it could be preempted by medium-priority work while a high-priority task remains blocked.

The second problem is demonstrated in the priority-inversion experiment.

---

# Induced Failure – Mutex Removed

For the induced-failure experiment, I temporarily removed the mutex protection from the two safety-station tasks.

I also added a short delay between the read and write operations:

```c
int old = shared_state;

vTaskDelay(pdMS_TO_TICKS(1));

shared_state = old + 1;
```

The normal mutex operations were temporarily commented out:

```c
// xSemaphoreTake(shared_mux, portMAX_DELAY);

// Critical section

// xSemaphoreGive(shared_mux);
```

The delay widens the race window and makes it more likely that both tasks read the same value before either task writes the result.

## Predicted symptom

I expected both safety-station tasks to occasionally read the same old counter value and then both write the same new value.

For example:

```text
[safety-station#1] cleared inspections 10 -> 11
[safety-station#2] cleared inspections 10 -> 11
```

Even though two tasks attempted to increment the value, the counter would only increase once. This is called a lost update.

## Observed symptom

The serial monitor showed both safety-station tasks reading the same value and writing the same new value.

```text
[safety-station#1] cleared inspections 0 -> 1
[safety-station#2] cleared inspections 0 -> 1

[safety-station#1] cleared inspections 1 -> 2
[safety-station#2] cleared inspections 1 -> 2

[safety-station#1] cleared inspections 2 -> 3
[safety-station#2] cleared inspections 2 -> 3
```

The race does not always occur because it depends on the exact scheduling order of the tasks. If one task completes the read-modify-write operation before the second task reads the counter, the output may still appear correct.

The problem occurs when both tasks execute this sequence:

1. Safety Station 1 reads the current value.
2. Safety Station 1 is delayed or preempted.
3. Safety Station 2 reads the same current value.
4. Both tasks calculate the same next value.
5. Both tasks write the same result.

After collecting the failure evidence, I restored the mutex in the final submitted code.

---

# Priority-Inversion Experiment

The project contains three tasks:

| Task | Relative priority | Purpose |
|---|---:|---|
| H | High | Needs the lock and measures how long it is blocked |
| M | Medium | Performs CPU-bound work but does not use the lock |
| L | Low | Takes the lock first and performs CPU-bound work |

The same scenario was tested using two lock types:

```c
#define USE_PI_MUTEX 1
```

uses a mutex with priority inheritance.

```c
#define USE_PI_MUTEX 0
```

uses a binary semaphore as a lock without priority inheritance.

The critical sections are CPU-bound instead of using `vTaskDelay()`. This is important because a delayed task is sleeping and does not compete for the CPU. If L slept inside the critical section, M could run in both modes and the difference between the two results would be less clear.

---

## Mode 1 – Mutex with Priority Inheritance

Configuration:

```c
#define USE_PI_MUTEX 1
```

The program reported:

```text
Lock mode: MUTEX (priority inheritance ON)

[PI][L] took lock @ 97059 us
[PI][H] wants lock @ 146761 us
[PI][H] ACQUIRED @ 12358948 us
waited 12212187 us (~12212 ms)
[PI][L] released lock @ 12358906 us
```

Task H waited:

```text
12,212,187 us
approximately 12.21 seconds
```

### Timeline

1. Task L acquired the mutex at `97,059 us`.
2. Task H attempted to acquire it at `146,761 us`.
3. H blocked because L owned the mutex.
4. FreeRTOS raised L's effective priority to the priority of H.
5. The medium-priority task could not preempt L while the inheritance was active.
6. L completed its CPU-bound critical section.
7. L released the mutex at `12,358,906 us`.
8. H acquired it at `12,358,948 us`.

H acquired the mutex only `42 us` after L released it:

```text
12,358,948 - 12,358,906 = 42 us
```

The wait was mostly caused by L's remaining critical-section execution time.

---

## Mode 2 – Binary Semaphore Without Priority Inheritance

Configuration:

```c
#define USE_PI_MUTEX 0
```

The program reported:

```text
Lock mode: BINARY SEM (no inheritance)

[PI][L] took lock @ 96350 us
[PI][H] wants lock @ 146761 us
[PI][M] ready @ 196761 us
[PI][M] done @ 25279765 us
[PI][L] released lock @ 37890486 us
[PI][H] ACQUIRED @ 37890527 us
waited 37743766 us (~37743 ms)
```

Task H waited:

```text
37,743,766 us
approximately 37.74 seconds
```

### Timeline

1. Task L acquired the binary semaphore at `96,350 us`.
2. Task H attempted to acquire it at `146,761 us`.
3. H blocked because L held the semaphore.
4. Task M became ready at `196,761 us`.
5. Because binary semaphores do not support priority inheritance, L remained at low priority.
6. M preempted L and ran its CPU-bound work.
7. M finished at `25,279,765 us`.
8. L eventually resumed and completed its critical section.
9. L released the lock at `37,890,486 us`.
10. H acquired it at `37,890,527 us`.

H again acquired the lock only `41 us` after L released it:

```text
37,890,527 - 37,890,486 = 41 us
```

The extra wait did not come from the lock handoff. It came from M preempting L while L still held the lock.

---

# Results Comparison

| Lock type | Priority inheritance | H wait time |
|---|---|---:|
| Mutex | Enabled | 12,212,187 us |
| Binary semaphore | Disabled | 37,743,766 us |

Difference:

```text
37,743,766 us - 12,212,187 us
= 25,531,579 us
```

Ratio:

```text
37,743,766 / 12,212,187
≈ 3.09
```

Without priority inheritance, H waited about **3.09 times longer**.

## Task responsible for the extra wait

Task M was responsible for most of the additional waiting in the PIP-off mode.

M did not use the lock, but it had a higher priority than L. Since L remained at low priority while holding the binary semaphore, M could preempt it.

H could not continue because it was blocked on L's lock. This produced the classic priority-inversion sequence:

```text
H waits for L
L waits for CPU time
M occupies the CPU
```

In mutex mode, L inherited H's priority. This prevented M from preempting L until L completed the critical section and released the mutex.

---

# Role-Appropriate Primitive Summary

| Primitive | Use in this project | Why it fits |
|---|---|---|
| Binary semaphore | ISR wakes emergency responder | Represents a single pending event and supports ISR-safe give operations |
| Counting semaphore | Three maintenance bays | Represents the number of identical resources available |
| Mutex | Protects inspection counter | Provides ownership, mutual exclusion, and priority inheritance |
| Binary semaphore as lock | Priority-inversion comparison only | Demonstrates the problems caused by no ownership or inheritance |

---

# Conclusions

This project demonstrated that FreeRTOS synchronization primitives have different intended roles.

The binary semaphore was appropriate for waking the emergency responder from an ISR because it represents a single event. The counting semaphore correctly limited the number of inspection teams using the maintenance area to three. The mutex protected the shared inspection counter and prevented lost updates.

The induced-failure test showed why the read-modify-write operation must be protected. Without the mutex, both writer tasks could read the same old value and overwrite one another's updates.

The priority-inversion experiment showed the most important difference between a mutex and a binary semaphore used as a lock. With priority inheritance enabled, H waited approximately **12.21 seconds**. Without inheritance, H waited approximately **37.74 seconds** because M preempted L while L still held the lock.

The experiment confirms that a mutex should be used for task-owned critical sections, especially when tasks have different priorities.

---

# AI Assistance Disclosure

ChatGPT was used to help organize the README and explain the priority-inversion timeline. All timestamps and measured wait values included in this report came from my own Wokwi serial-monitor runs.

Chat used: OpenAI ChatGPT, accessed July 2026.

---

# Final Submission State

The final submitted version:

- Uses the binary semaphore for ISR-to-task signaling
- Uses a counting semaphore with three maintenance-bay tokens
- Uses a mutex around the shared inspection counter
- Restores all synchronization after the induced-failure experiment
- Uses the selected `USE_PI_MUTEX` setting required for the final demonstration
