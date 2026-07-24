
# App 5 – IPC Pipeline (ThemePark)

## Overview

This project demonstrates three FreeRTOS IPC mechanisms running on an ESP32 dual-core system. Core 1 runs the real-time ride pipeline while Core 0 runs the system monitor. The ThemePark pipeline simulates a ride sensor sending data to a safety controller, which then notifies the operator display. An emergency stop button can interrupt the system at any time using a direct task notification from an ISR.

---

# System Architecture

## Core 1 (Real-Time Tasks)

### Ride Sensor Task
- Generates ride status data every 50 ms.
- Sends each sample to the Safety Controller using a Queue.
- Sets the `EV_BIT_DATA_PRODUCED` event bit.

### Safety Controller Task
- Receives ride samples from the queue.
- Checks ride speed and restraint status.
- Determines whether the ride is safe.
- Sets the `EV_BIT_DATA_PROCESSED` event bit.

### Ride Coordinator Task
- Waits until both event bits are set.
- Confirms the ride cycle has completed.
- Sends a direct task notification to the responder.

### Emergency Responder / Operator Display
- Wakes when notified.
- Handles emergency stop events from the button ISR.
- Reports the ride status.

---

## Core 0 (Monitor)

The monitor displays:

- Queue depth
- Event group bits
- Last processed ride sample
- Queue drops
- Notification latency
- Binary semaphore latency
- Task heartbeat counters

Keeping the monitor on Core 0 prevents monitoring from interfering with the real-time tasks on Core 1.

---

# IPC Mechanisms

## Queue

A Queue transfers `RideStatus` structures from the Ride Sensor task to the Safety Controller. This keeps the producer and consumer independent while preserving FIFO ordering.

---

## Event Group

The Event Group synchronizes the producer and consumer.

The coordinator waits until:

- Data Produced
- Data Processed

have both occurred before notifying the responder.

Using an Event Group is simpler than using multiple semaphores because it can wait on multiple events at once.

---

## Direct Task Notification

The responder receives direct task notifications from:

- The coordinator after a completed ride cycle.
- The emergency button ISR.

I used direct task notifications because they are a faster way to wake up one specific task than using a binary semaphore.

---

# Queue Design

The queue depth was set to **8 items**.

The producer creates one ride sample every **50 ms (20 Hz)**.

The consumer processes approximately one sample every **60 ms**.

Worst-case burst:

- Producer = 20 samples/sec
- Consumer ≈ 16–17 samples/sec

An 8-item queue provides enough room for temporary bursts while keeping memory usage reasonable.

---

# Back-Pressure Policy

If the queue becomes full, the producer drops the newest sample and records the event.

Example:

```text
[ride-sensor] queue full; dropped newest sample
```

This keeps the producer from blocking and allows the real-time system to continue operating.

---

# Event Group vs Multiple Semaphores

An Event Group was chosen because the coordinator must wait until multiple events have happened before continuing.

Using separate semaphores would require more synchronization logic and additional code.

I chose an Event Group because I needed to wait for both events before moving on. It was easier than trying to manage multiple semaphores.

---

# Direct Notification vs Binary Semaphore

Measured wakeup latency from the serial monitor:

### Direct Task Notification

- 10 µs
- 98 µs
- 738 µs
- 2545 µs

### Binary Semaphore

- 15,758 µs
- 16,392 µs
- 20,903 µs
- 24,574 µs

The direct task notification consistently woke the responder faster than the binary semaphore because it signals the target task directly without using an additional kernel object.

---

# Why Pin the Web Server to Core 0?

The web server and monitor run on Core 0 so networking and logging do not interfere with the real-time tasks running on Core 1.

If the web server were placed on Core 1, handling HTTP requests and Wi-Fi traffic could delay the producer, consumer, or emergency responder tasks. Keeping the monitor on Core 0 lets the real-time tasks on Core 1 run without getting slowed down by the web server or logging.
---

# Concurrency Diagram

```text
                     CORE 1

 Ride Sensor Task
        |
        | Queue
        v
 Safety Controller
        |
        | Event Group
        v
 Ride Coordinator
        |
        | Direct Notification
        v
 Emergency Responder

 Button
    |
   ISR
    |
 Direct Notification
    |
 Emergency Responder


                     CORE 0

      System Monitor
      ├── Queue Depth
      ├── Event Bits
      ├── Last Ride Sample
      ├── Queue Drops
      ├── Heartbeats
      └── Latency Measurements
```

---

# AI Usage

The starter scaffold provided for App 5 was used as the base for this project. I also reused parts of my App 1 web server and App 3 latency measurement code as suggested in the assignment. The ThemePark pipeline, queue logic, synchronization, and testing were completed for this application.
