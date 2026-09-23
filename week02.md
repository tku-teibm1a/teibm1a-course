# Week 02 — One Node, No Network

**In class:** 1 hour · **Outside class:** ~1 hour · **Covered by:** Checkpoint 1

Nothing connects to anything else this week. Understand one process completely before making three of them argue.

---

## 1 · What you will have at the end

- MicroPython flashed onto one Pico, with a working REPL
- The on-board LED blinking under your control
- A temperature sensor read, printed once per second
- A measurement of how **stable** your loop timing actually is

**Optional:** the ultrasonic sensor. It needs a voltage divider built correctly, and nothing above depends on it.

---

## 2 · Workload budget

| Task | When | Budget |
|---|---|---|
| Flash firmware, REPL, blink | In class | ~20 min |
| Temperature sensor reading | In class | ~30 min |
| Loop-timing measurement | After class (runs unattended) | ~20 min of your time |
| Model specification + write-up | After class | ~40 min |
| Ultrasonic sensor and divider | Optional | ~40 min |

If something will not work, **stop and write down where you got to**. A clear account of an unresolved problem is worth marks; silence is not.

---

## 3 · ⚠️ Read before wiring anything

You have **two Picos and no spares**. There is no protection circuitry worth relying on.

- **Pico GPIO is 3.3 V only.** Applying 5 V damages a pin — sometimes immediately, sometimes slowly.
- **Never power a sensor from a GPIO pin.** Use `3V3(OUT)` or `VBUS` (5 V), and share a common `GND`.
- **Unplug USB before rewiring.** Every time.

---

## 4 · Your board, and how it differs from a Pi

Your board is a **Raspberry Pi Pico 2 WH**: RP2350 chip, wireless, headers pre-soldered (no soldering needed).

A Pico is not a small Raspberry Pi. The difference matters more than it first appears:

| | Pi Zero / Pi 4 | Pico |
|---|---|---|
| What it is | A Linux computer | A bare microcontroller, **no operating system** |
| How you connect | SSH | USB cable; the IDE pushes files onto the board |
| Installing things | `apt`, `pip` | No `apt`. Built-in modules, or `mip` over Wi-Fi |
| Where files live | Full filesystem on SD | A small flash area holding your `.py` files |
| Running a program | `python3 foo.py` | Save as `main.py` — it runs on power-up |

You cannot SSH into a Pico. There is nothing there to SSH into.

---

## 5 · Tooling: Thonny

Install **Thonny** from [thonny.org](https://thonny.org) (it is already present on Raspberry Pi OS). It is the standard editor for MicroPython and what most documentation assumes.

1. Plug the Pico in over USB.
2. Look at the **bottom-right** of the Thonny window — it shows the current interpreter. Click it and choose **MicroPython (Raspberry Pi Pico)**.
3. The panel at the bottom is the **Shell** — a live REPL on the board. Type into it directly.
4. When you press **Run**, Thonny asks whether to save to *This computer* or *Raspberry Pi Pico*. Choose the Pico.

**The single most confusing thing about Thonny:** it shows you two filesystems — your laptop and the board. Saving to the wrong one is the most common source of "my code isn't running".

**If a program will not stop:** press the red Stop button, or `Ctrl+C` in the Shell. If `main.py` contains a loop that blocks reconnection entirely, hold BOOTSEL while plugging in and reflash the firmware — this erases the board's files.

If you prefer VS Code, the **MicroPico** extension works well and has better completion. Thonny is the path of least resistance for week one.

---

## 6 · Flashing MicroPython

1. Download the MicroPython `.uf2` for **Pico 2 W** from the official Raspberry Pi site.
   ⚠️ The builds for *Pico W* and for plain *Pico 2* look almost identical on the download page and **will not work**.
2. Hold **BOOTSEL** while plugging in USB.
3. The board appears as a USB drive named `RP2350`. *(If you see `RPI-RP2`, you are holding a different board — tell the instructor.)*
4. Drag the `.uf2` onto it. The board reboots automatically.
5. In Thonny, select the MicroPython (Raspberry Pi Pico) interpreter.

If no drive appears, suspect the cable — many micro-USB cables carry power but no data, and they fail silently.

---

## 7 · Task A — First contact

Type directly into the Shell:

```python
import sys
sys.implementation
import machine
machine.freq()
```

You are now talking to a live process on a separate machine. It has its own memory and its own notion of time, and it keeps running whether or not you are watching. That is exactly this week's lecture definition of a process.

---

## 8 · Task B — Blink

⚠️ **On the wireless Picos the on-board LED is not GPIO 25.** It is driven through the wireless chip. `Pin(25)` produces **no error and no light** — this is the most common reason a first script appears to do nothing. Most tutorials online were written for the original Pico.

```python
from machine import Pin
import time

led = Pin("LED", Pin.OUT)

while True:
    led.toggle()
    time.sleep(0.5)
```

Save it to the board as `main.py`. It now runs on power-up with no computer attached — your board is a node rather than a peripheral.

---

## 9 · Task C — Read a sensor

The kit contains five sensors. This week uses one:

| Sensor | Interface | Used in |
|---|---|---|
| Temp / humidity | One-wire digital, driver built in | **This week** |
| Ultrasonic | Trigger / echo pulse timing | Optional, §11 |
| PIR motion | Digital, event-triggered | Week 3 |
| Ambient light | Usually I²C | Week 5, optional |
| Pressure | Usually I²C | Week 5, optional |

Kits ship either a **DHT11** (blue) or a **DHT22** (white).

```python
from machine import Pin
import dht, time

sensor = dht.DHT22(Pin(16))   # TODO: DHT11 or DHT22? which GPIO?

while True:
    try:
        sensor.measure()
        print(sensor.temperature(), sensor.humidity())
    except OSError as e:
        print("read failed:", e)
    time.sleep(2)
```

Note the `try`. These sensors fail a read fairly often, and a bare script crashes on the first failure.

**Two details that will otherwise cost you an hour:**

- **Minimum interval.** DHT22 needs roughly 2 s between readings, DHT11 about 1 s. Poll faster and `measure()` simply raises an error.
- **Pull-up resistor.** The data line needs a 4.7 k–10 kΩ pull-up to 3V3. Most kit modules have it on the small PCB; bare three-pin sensors do not. *Check yours.*

> If **every** read raises `OSError`, the wiring is wrong before the code is.

---

## 10 · Task D — Measure your own timing

The lecture claimed that process speeds are not bounded in any useful way. Test that claim on your own node.

```python
import time

while True:
    t0 = time.ticks_us()
    sensor.measure()          # or your sensor read
    dt = time.ticks_diff(time.ticks_us(), t0)
    print(dt)
    time.sleep(1)
```

Let it run for a few minutes unattended. Record the **minimum, maximum and typical** values.

**What you should find:** the times are not constant. Occasional readings take far longer than typical ones — inside a single, idle, dedicated processor with no operating system and nothing else to do.

Now extrapolate. If *one* node cannot promise a stable response time on its own, a gateway waiting on *three* such nodes across shared Wi-Fi certainly cannot. This is the asynchronous model, observed rather than asserted.

**Keep these numbers.** You will use them in Week 8 when choosing a timeout.

---

## 11 · Optional — Ultrasonic distance

⚠️ **The HC-SR04 can damage your Pico.** It runs at 5 V and its `ECHO` pin outputs 5 V into a 3.3 V input.

Use a voltage divider on `ECHO`:

```
ECHO ──[ 1 kΩ ]──┬── Pico GPIO
                 │
              [ 2 kΩ ]
                 │
                GND
```

Roughly 2:1 works; use what your resistor pack contains. `TRIG` may be driven directly from a GPIO — only `ECHO` needs this. Most online tutorials omit it, and boards damaged this way often keep half-working, which makes the fault very hard to diagnose later.

```python
from machine import Pin, time_pulse_us
import time

trig = Pin(3, Pin.OUT)   # TODO: your wiring
echo = Pin(2, Pin.IN)    # TODO: via voltage divider

def distance_cm():
    trig.low()
    time.sleep_us(2)
    trig.high()
    time.sleep_us(10)
    trig.low()
    dur = time_pulse_us(echo, 1, 30000)   # 30 ms timeout
    if dur < 0:
        return None            # no echo returned
    return (dur * 0.0343) / 2
```

Returning `None` rather than a number is deliberate. "No reading" is a different outcome from "a reading of zero", and conflating them is how bad data enters a system.

---

## 12 · What to submit

Everything goes in `week02/` of your team repository, following the structure in the [Submission Guide](../SUBMISSION_GUIDE.md).

- **Code** — your sensor-reading script, running standalone from `main.py`
- **Timing data** — min, max and typical loop duration, and how long you sampled
- **Report §3** — from your timing data: is your node's response time bounded in any useful way, and what does that tell you about which timing model you can honestly assume?
- **Model specification** — the half-page from the lecture: your timing assumption, your failure assumption, and how a failed node would be detected

`model-spec.md` is a **living document**. You will revise it most weeks, and in Week 14 it becomes the opening section of your project proposal. Teams who treat it as disposable now pay for it later.

---

## 13 · When it does not work

| Symptom | Most likely cause |
|---|---|
| No USB drive appears | Charge-only cable, or BOOTSEL not held while plugging in |
| Firmware copies, board never reappears | Wrong `.uf2` — the Pico W or plain Pico 2 build |
| On-board LED does nothing | Using `Pin(25)` instead of `Pin("LED")` |
| Code runs on the laptop, not the board | Saved to "This computer" instead of "Raspberry Pi Pico" |
| Every sensor read raises `OSError` | Wrong pin, missing pull-up, or polling too fast |
| Values look plausible but never change | Reading a floating pin, or the sensor has no power |
| Distance always `None` | TRIG/ECHO swapped, or no common ground |

Change one thing at a time, and predict the result before you test it.

---

## 14 · Where to look things up

| Source | Use it for |
|---|---|
| MicroPython docs | The `machine`, `time` and `dht` modules — the authoritative API reference |
| Raspberry Pi docs | Firmware downloads, pinout diagrams, board differences |
| Your sensor's datasheet | Operating voltage, timing limits, whether a pull-up is built in |

Be careful with blog tutorials: most were written for the **original Pico** and fail silently on the wireless models. §8 is the canonical example.

Working from official documentation and a datasheet is not an obstacle to this course. At graduate level, it is part of it.
