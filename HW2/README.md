
## Part A

| Memory region | Address range | Typical use |
|---|---|---|
| Internal SRAM | `0x3FC88000–0x3FCFFFFF` and `0x40370000–0x403DFFFF` | Stack, heap, data, and code placed in internal RAM |
| External flash through cache | `0x42000000–0x43FFFFFF` instruction side, `0x3C000000–0x3DFFFFFF` data side | Program code and read-only data stored in external flash |
| ROM0 | `0x40000000–0x4003FFFF` | Built-in boot and ROM functions |
| ROM1 | `0x40040000–0x4005FFFF` and `0x3FF00000–0x3FF1FFFF` | More built-in ROM code/data |
| Interrupt / ISR | Not one simple memory address range | Interrupt handlers are registered by the interrupt system; time-critical ISRs should be placed in IRAM |

Putting an ISR close to the processor matters because external flash is reached through cache. If the ISR code is in flash and the cache misses, or flash is temporarily unavailable, the interrupt can take longer to run. For real-time systems, that hurts worst-case latency. Placing important ISR code in internal RAM makes the response more predictable.

## Part B 

https://wokwi.com/projects/465309953489601537

AI Disclosure

ChatGPT was used to help explain and verify the following concepts:

Bit-field syntax and 16-bit layout
Volatile and MMIO behavior

## Part C

Volatile is not enough when two FreeRTOS tasks share an int. Volatile only tells the compiler that the value can change unexpectedly, so it should actually perform the load or store. It does not make operations atomic, and it does not protect something like `x++`, which is really a read, modify, and write.

The minimum guarantee volatile gives is that the compiler should not optimize away the access or keep using an old cached register value for that volatile object. It does not give mutual exclusion, atomicity, or full synchronization between tasks.

A better primitive would be a FreeRTOS mutex, semaphore, critical section with a spinlock, or a C11 atomic variable.

## Part D

1. Keep control flow simple and avoid recursion or goto.
2. Put a fixed limit on every loop.
3. Do not use dynamic memory allocation after initialization.
4. Keep functions short and easy to review.
5. Use assertions for conditions that should always be true.
6. Keep variable scope small and declare variables close to use.
7. Check return values and validate inputs.
8. Keep preprocessor macros simple.
9. Avoid complicated pointer tricks and function pointers.
10. Compile with strict warnings and fix all warnings.

The hardest rule for me is avoiding dynamic memory after initialization. In class projects, I usually use malloc when I need an array or linked list, then free it later. For embedded code, I would need to plan memory earlier and use fixed-size buffers more often. That would make the code less flexible, but it would also make memory use and timing easier to predict.
