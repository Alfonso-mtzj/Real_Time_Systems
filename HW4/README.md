# Real-Time Scheduling Assignment — RMS / EDF / RTA

## Overview
This assignment uses Liu & Layland utilization tests and Response Time Analysis (RTA) to check if different task sets are schedulable under RMS and EDF.

---

# Part A — Utilization + Liu & Layland Bound

## Set 1
Tasks:
- τ₁ (T=10, C=2)
- τ₂ (T=25, C=5)
- τ₃ (T=100, C=20)

Utilization:
- 2/10 = 0.2  
- 5/25 = 0.2  
- 20/100 = 0.2  

Total U = 0.6  

RMS bound (n=3):
≈ 0.78  

So:
- 0.6 < 0.78 → RMS is guaranteed schedulable  
- EDF: 0.6 ≤ 1 → schedulable  

---

## Set 2
U = 0.25 + 0.333 + 0.3 ≈ 0.883  

RMS bound (n=3) ≈ 0.78  

- 0.883 > 0.78 → not guaranteed under RMS  
- EDF still works since U ≤ 1  

---

## Set 3
Each task is 0.2 utilization:

Total U = 0.8  

RMS bound (n=4) ≈ 0.756  

- 0.8 > 0.756 → not guaranteed  
- EDF: still schedulable (U ≤ 1)

---

# Part B — Response Time Analysis (Set 1)

Priority order (RMS): τ₁ → τ₂ → τ₃

---

## τ₁
R₁ = C₁ = 2  
Deadline = 10 → fine

---

## τ₂
Higher priority: τ₁

Start:
- R₂ = 5  
- R₂ = 5 + ⌈5/10⌉·2 = 7  
- R₂ stays 7 after that  

So R₂ = 7 ≤ 25 → OK

---

## τ₃
Higher priority: τ₁, τ₂

Start:
- R₃ = 20  
- R₃ = 29  
- R₃ = 36  
- R₃ = 38  
- stabilizes at 38  

So R₃ = 38 ≤ 100 → OK

---

# Part C — Failure Scenario (Set 3, C₄ changed)

Now C₄ = 25 ms

## Utilization
U = 0.2 + 0.2 + 0.2 + 0.5 = 1.1  

So:
- U > 1 → not schedulable at all (EDF or RMS)

---

## RTA check (τ₁ and τ₂ only)

### τ₁
R₁ = 1 → meets deadline

### τ₂
- R₂ = 2 + ⌈2/5⌉·1 = 3  
- stable at 3  

So τ₂ is still fine

---

## What this means
Even though the full system fails, the higher priority tasks still meet deadlines because the overload is caused by τ₄.

---

## Fixes (no hardware changes)

1. Reduce C₄  
   - like from 25 → 15 ms  
   - lowers total CPU usage

2. Increase T₄  
   - like 50 → 80 or 100 ms  
   - makes it run less often so system load drops

---

# Part D — Industry Use (AUTOSAR / DO-178C)

In real systems like AUTOSAR or avionics (DO-178C), scheduling is proven using tools like Simulink or timing analysis tools (like Rapita).

They calculate WCET and run response time analysis to prove all deadlines are met. This gets included in certification documents that are reviewed by safety authorities.

If RTA shows a deadline miss, they don’t ignore it — they have to change task timing, execution time, or system design until it passes.

Basically, it has to be mathematically proven, not just “it seems fine in testing.”
