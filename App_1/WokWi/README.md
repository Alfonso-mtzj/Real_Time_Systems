# App 1 — ThemePark Ride Controller

Theme: Industrial / Theme-Park

I decided to go with the Theme park theme because I love theme parks and because real-time systems are important in ride safety and dispatch systems to keep the people safe. The LED blinking tells us that a dispatch is ready, and the webpage acts as a console for the operator.

## Wokwi Link

https://wokwi.com/projects/465292104139129857

## Run Instructions

1. Open the Wokwi project.
2. Start the simulation.
3. The dispatch beacon LED blinks at 1 Hz.
4. The serial monitor shows the themed operator console logs.

## Concurrency Diagram

[ Core 1 ]
blink_task / dispatch_beacon
        |
        | shared volatile bool led_on
        v
[ Core 0 ]
HTTP server / operator console

## Engineering analysis prompts (answer in your README)

1. **Why two tasks?** What's the failure mode of a single super-loop that polls the web server AND blinks the LED?

  The program uses two tasks so the real-time blinking behavior is separated from the web server. The blink_task takes care of toggling the dispatch beacon at a constant 1 Hz rate, while the HTTP server shows the beacon status on the operator console webpage.

  A busy web client could cause a super-loop like poll_web_server(); toggle_led(); delay(1000); to be delayed. This tells us that the LED might not toggle every second because the processor could be stuck handling web requests. That is dangerous in a real-time system when timing becomes variable.
   
2. **Why pin to specific cores?** What problem does pinning solve that round-robin scheduling doesn't?

  The scaffold attaches the HTTP server to Core 0 and blink_task to Core 1. This, in my opinion, keeps the networking/observability requirement and the timing-critical task apart. While the web server may experience delays based on client requests, the dispatch beacon requires consistent timing.

  The HTTP server and the blink job may compete for CPU time if both processes use the same core. Jitter or delays in the LED timing could be caused by a busy web client. The networking plane and the real-time plane are more segregated by pinning them to separate cores.

3. **Why does a single `bool` (our `led_on`) not need a mutex here, but a `struct { bool; uint32_t; }` would?** Reference what you know about atomic reads on Xtensa.

  The shared variable volatile bool led_on is read by one core and written by the other core. This is safe in this scaffold because it is a single simple value, and aligned reads and writes of small values on the Xtensa architecture are atomic. The volatile keyword also tells the compiler not to optimize away repeated reads of the variable.

  If led_on was replaced with struct { bool on; uint32_t count; }, the same pattern would be unsafe because the reader could see one field updated but another field still old. That could create an inconsistent snapshot of the shared state.

## AI Tool Disclosure

I used ChatGPT to help debug the ESP32 scaffold, improve the theme wording, and better understand some of the engineering analysis questions. I tested and modified the code myself in Wokwi before submission.

ChatGPT share link:
https://chatgpt.com/share/6a18d205-c658-83ea-844a-2901c2fbe0fa

## Screenshots

<img width="1264" height="1217" alt="image" src="https://github.com/user-attachments/assets/1e07f239-481e-4337-90f8-cb3727f3671c" />
