# Real-Time Scheduling Assignment — Liu & Layland + RTA Analysis

## Overview
This assignment applies real-time scheduling theory (RMS + EDF) using Liu & Layland utilization bounds and Response Time Analysis (RTA). The goal is to determine whether different task sets are schedulable and justify it mathematically.

---

# Part A — Utilization + Liu & Layland Bound

## Set 1
Tasks:
- τ₁ (T=10, C=2)
- τ₂ (T=25, C=5)
- τ₃ (T=100, C=20)

Utilization:
- U₁ = 2/10 = 0.20  
- U₂ = 5/25 = 0.20  
- U₃ = 20/100 = 0.20  

Total:
- **U = 0.60**

RMS bound (n=3):
- U_LL ≈ 0.78

Result:
- 0.60 < 0.78 → **RMS guaranteed schedulable**
- EDF: 0.60 ≤ 1 → **Schedulable**

---

## Set 2
- U = 0.25 + 0.333 + 0.30 ≈ **0.883**

RMS bound (n=3):
- U_LL ≈ 0.78

Result:
- 0.883 > 0.78 → **RMS not guaranteed**
- EDF: 0.883 ≤ 1 → **Schedulable**

---

## Set 3
All tasks have U = 0.2:

Total:
- **U = 0.80**

RMS bound (n=4):
- U_LL ≈ 0.756

Result:
- 0.80 > 0.756 → **Not guaranteed under RMS bound**
- EDF: 0.80 ≤ 1 → **Schedulable**

---

# Part B — Response Time Analysis (Set 1)

Priority order (RMS):
τ₁ → τ₂ → τ₃

---

## τ₁
R₁ = C₁ = 2  
Deadline = 10 → **OK**

---

## τ₂
Higher priority: τ₁

Iterations:
- R₂⁰ = 5  
- R₂¹ = 5 + ⌈5/10⌉·2 = 7  
- R₂² = 7 (converged)

R₂ = 7 ≤ 25 → **OK**

---

## τ₃
Higher priority: τ₁, τ₂

Iterations:
- R₃⁰ = 20  
- R₃¹ = 29  
- R₃² = 36  
- R₃³ = 38  
- R₃⁴ = 38 (converged)

R₃ = 38 ≤ 100 → **OK**

---

## Part B Result
All tasks meet deadlines → **Set 1 is RMS schedulable using RTA**

---

# Part C — Failure Scenario (Set 3 modified)

Change:
- C₄ = 25 ms (instead of 10 ms)

---

## New Utilization
U = 0.2 + 0.2 + 0.2 + 0.5 = **1.1**

Result:
- U > 1 → **Not schedulable (even EDF fails)**
- RMS bound already fails

---

## RTA Check (τ₁ and τ₂)

### τ₁
R₁ = 1 → **feasible**

### τ₂
- R₂⁰ = 2 + ⌈2/5⌉·1 = 3  
- R₂¹ = 3 (converged)

R₂ = 3 ≤ 10 → **feasible**

---

## Observation
Even though the whole system fails, τ₁ and τ₂ still meet deadlines because the lowest priority task (τ₄) is what causes overload.

---

## Design Fixes (no hardware changes)

### 1. Reduce execution time of τ₄
- Example: C₄ from 25 → 15 ms  
- Lowers total CPU usage  
- Helps bring system back under schedulable range

### 2. Increase period of τ₄
- Example: T₄ from 50 → 80+ ms  
- Reduces how often heavy task runs  
- Common real-time tradeoff: slower updates for stability

---

# Part D — Industry Use (AUTOSAR / DO-178C)

In real systems like automotive (AUTOSAR) or avionics (DO-178C), scheduling is proven using formal analysis, not guessing.

Engineers use tools like **Simulink** or timing analysis tools (like Rapita) to calculate WCET and Response Time Analysis (RTA). These results are included in certification documents that are reviewed by safety authorities.

Tasks are assigned fixed priorities, and every deadline must be mathematically proven to meet timing constraints. If RTA shows a deadline miss, the system design must be changed (like reducing execution time or adjusting task periods) before it can be certified.

In safety-critical systems, “it seems like it works” is never enough — everything must be proven with analysis and documentation.
