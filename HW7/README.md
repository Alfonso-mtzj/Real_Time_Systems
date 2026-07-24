# HW 7 – FreeRTOS IPC: Primitive Selection and Contracts

## Part A – Pick the Primitive

### 1. Avionics: 50 Hz attitude sensor feeds a 50 Hz control loop (variable-length packet, 1-to-1)

**Primitive:** Message Buffer

**Why:** A message buffer keeps each variable-length packet together and is designed for one producer and one consumer.

**Worst alternative:** Stream Buffer

**Why it's worse:** A stream buffer only sends bytes, so the receiver would need extra logic to figure out where each packet starts and ends.

---

### 2. Medical: 1 kHz ECG sample stream from ISR into a moving-window task

**Primitive:** Stream Buffer

**Why:** ECG data is a continuous stream of samples, making a stream buffer the best fit.

**Worst alternative:** Event Group

**Why it's worse:** Event groups only signal events and cannot store the ECG samples.

---

### 3. Space: Five sensors must report ready before fusion starts

**Primitive:** Event Group

**Why:** Each sensor sets one bit, and the fusion task waits until all five bits are set.

**Worst alternative:** Direct Task Notification

**Why it's worse:** A task notification is better for one-to-one signaling and doesn't naturally represent multiple sensors becoming ready.

---

### 4. Industrial: E-stop signals one critical responder with sub-ms latency

**Primitive:** Direct Task Notification

**Why:** It is the fastest way to wake a single task with very little overhead.

**Worst alternative:** Queue

**Why it's worse:** A queue copies data and has more overhead even though no data needs to be transferred.

---

### 5. Security: Six worker tasks send logs to one signed-log task

**Primitive:** Queue

**Why:** Queues safely support multiple producers sending complete log records to one consumer.

**Worst alternative:** Message Buffer

**Why it's worse:** Message buffers are intended for one producer and one consumer, so multiple writers would require additional synchronization.

---

### 6. UART RX bytes into a parser task (variable rate)

**Primitive:** Stream Buffer

**Why:** UART receives a continuous stream of bytes, which matches the purpose of a stream buffer.

**Worst alternative:** Event Group

**Why it's worse:** Event groups can only signal that data arrived and cannot store the actual bytes.

---

### 7. CAN frames (8 bytes) from driver into multiple consumers by message ID

**Primitive:** Queue

**Why:** The driver can place complete CAN frames into a queue, and a dispatcher task can send them to the correct consumer.

**Worst alternative:** Stream Buffer

**Why it's worse:** A stream buffer does not preserve CAN frame boundaries or support routing complete messages.

---

### 8. Track count of sensor-overrange events since boot

**Primitive:** Counting Semaphore

**Why:** A counting semaphore keeps track of how many events have occurred.

**Worst alternative:** Binary Semaphore

**Why it's worse:** A binary semaphore only remembers one pending event, so multiple events could be lost.

---

# Part B – Cost & Contract

## Contract 1 – ECG ISR → Moving Window Task

| Item | Value |
|------|------|
| Primitive | Stream Buffer |
| Producer | ADC ISR |
| Consumer | ECG processing task |
| Data | int16_t ECG samples |
| Max Rate | 1000 samples/sec |
| Buffer Size | 200 bytes (100 samples) |
| Queue Depth | 100 samples |
| Sized By | 100 ms of data at 1000 samples/sec |
| Back-pressure | ISR never blocks. If full, increment overflow counter and drop the newest sample. |
| Worst-case Latency | Less than 10 ms |

### Explanation

Since there's only one ISR sending data and one task reading it, a stream buffer works well here. The 200-byte buffer stores about 100 ms of ECG data, giving the processing task enough time to catch up if it briefly falls behind.

---

## Contract 2 – Worker Tasks → Signed Log Task

```c
typedef struct {
    uint32_t timestamp;
    uint16_t workerID;
    uint8_t severity;
    uint8_t eventCode;
    char message[48];
} LogRecord_t;
```

| Item | Value |
|------|------|
| Primitive | Queue |
| Producers | 6 worker tasks |
| Consumer | Signed-log task |
| Data | LogRecord_t |
| Max Rate | 60 records/sec total |
| Queue Depth | 30 records |
| Sized By | 0.5 seconds of maximum traffic |
| Back-pressure | Wait up to 10 ms. If still full, increment dropped log counter. |
| Worst-case Latency | Less than 500 ms |

### Explanation

The queue safely allows multiple worker tasks to send complete log records to one logging task. A depth of 30 records can hold about half a second of traffic if every worker is producing logs at the maximum expected rate.

Calculation:

60 records/sec × 0.5 sec = 30 records

---

# Summary

- **Queue:** Best for fixed-size data and multiple producers.
- **Direct Task Notification:** Best for fast one-to-one signaling.
- **Event Group:** Best when waiting for multiple events.
- **Message Buffer:** Best for variable-length messages.
- **Stream Buffer:** Best for continuous byte streams.
- **Counting Semaphore:** Best for counting repeated events or available resources.
