# Week 03 — Two Clocks, No Agreement

**In class:** 1 hour · **Outside class:** ~1 hour 10 min · **Covered by:** Checkpoint 1

Both Picos watch the same physical event — a single PIR sensor, wired to both. Each records its own timestamp, with no communication of any kind between them. Afterwards you compare the two logs and measure how far apart they are.

Still no network. When the disagreement appears, there is nothing to blame it on.

---

## 1 · What you will have at the end

- One PIR sensor wired to both Picos, each logging its events to its own flash storage
- Two logs of the same session, aligned against each other
- Your measured clock drift, in **parts per million**
- A clear separation between the part of the disagreement you could calibrate away, and the part you cannot

---

## 2 · Workload budget

| Task | When | Budget |
|---|---|---|
| PIR wiring, interrupt capture, logging to flash | In class | ~45 min |
| Start the run on both boards | In class | ~15 min |
| The 20–30 minute run itself | Unattended | none of your time |
| Read out both logs, align, plot | After class | ~40 min |
| Write-up and model spec update | After class | ~30 min |

The run is **unattended**. Trigger an event every few minutes and do something else in between — you are not expected to watch a sensor for half an hour.

**Use that time well:** flash your Pi Zero's SD card with Raspberry Pi Imager while the run goes. That is the Week 4 preparation (§14), and it needs only your laptop — the Pi Zero does not have to be powered.

---

## 3 · ⚠️ Read before wiring

The same three rules as Week 2 still apply: **GPIO is 3.3 V only**, never power a sensor from a GPIO pin, and unplug USB before rewiring.

**The PIR sensor** (typically an HC-SR501) is powered from **5 V** — use `VBUS`, not `3V3`. Its output is 3.3 V logic on most modules, which makes it safe to connect directly to a GPIO. **Confirm that against your module's documentation** rather than assuming it; the cost of being wrong is a damaged pin.

**Your kit has one PIR sensor. Share it between both Picos:**

| PIR pin | Connect to |
|---|---|
| VCC | Pico A's `VBUS` (5 V) |
| OUT | A GPIO pin on **Pico A** *and* a GPIO pin on **Pico B** |
| GND | Pico A's GND **and** Pico B's GND — all three grounds joined |

**Joining the grounds is essential.** Without a common ground, Pico B has no reference for the signal and will log nothing, or nonsense.

Both boards now see *exactly the same electrical signal*. That makes this experiment cleaner than using two sensors would: any difference in their timestamps is almost entirely clock drift.

*If you have already done this with two different sensors instead, that is also accepted — see §11.*

---

## 4 · First obstacle: two boards, one cable

Both Picos must run at the same time, but only one micro-USB cable came in the kit.

You need two **power sources**, not two data cables:

1. **Program both boards with the one cable, in turn.** Plug in A, save your code as `main.py`, unplug. Then do the same for B. Once the code is on a board it runs by itself.
2. **For the run, the second board needs power only.** The simplest source is already in your kit: the **Pi Zero's 5 V power supply** has the same micro-USB plug as the Pico. A phone charger or power bank also works.

**The consequence matters more than the inconvenience.** With no serial console attached, `print()` goes nowhere. The node has to record events into its own storage and hand them over later.

That is the first genuinely distributed thing you will build: a process that accumulates local state **while unobserved**, and can be interrogated about its past afterwards. Every failure-detection and recovery mechanism later in the course depends on nodes being able to do this.

---

## 5 · Three PIR behaviours that look like bugs

Read this before debugging anything.

- **Warm-up.** The sensor needs about **a minute** after power-on to stabilise. During that time it fires more or less at random.
- **Two potentiometers** — sensitivity and hold-time. If the output stays high for many seconds after one movement, hold-time is turned up.
- **A trigger-mode jumper.** One position re-triggers while motion continues; the other does not. They produce very different event streams.

With one shared sensor, both boards receive the same signal, so these settings affect both equally. Set the hold-time short, so separate movements produce separate events.

---

## 6 · The problem at the heart of this week

A Pico has **no battery-backed real-time clock**. Any wall-clock time it reports after boot starts from an arbitrary default. What it does have is a counter of milliseconds since power-on:

```python
import time
time.ticks_ms()     # milliseconds since THIS board booted
```

Reliable and steady — and entirely local. Now suppose board A boots, and twenty seconds later you plug in board B. At the same physical instant:

| Board | `ticks_ms()` |
|---|---|
| A | 45,000 |
| B | 25,000 |

That 20-second difference is **not clock error**. The two counters simply count from different moments. There is no shared starting point, because nothing in the system ever established one.

**So comparing raw timestamps between the two boards is meaningless** — and you have no network with which to set up a shared reference.

---

## 7 · The trick: let the physical world be the reference

Both boards can see the **same event**. Use the **first detection** as a shared origin, and express every later event as an offset from it:

| | Offsets |
|---|---|
| Board A | t<sub>A,2</sub> − t<sub>A,1</sub>, t<sub>A,3</sub> − t<sub>A,1</sub>, … |
| Board B | t<sub>B,2</sub> − t<sub>B,1</sub>, t<sub>B,3</sub> − t<sub>B,1</sub>, … |

Now the two sequences **are** comparable. Any difference between them is real disagreement between the clocks — which is exactly what you came to measure.

---

## 8 · Task A — Capture events accurately

A polling loop adds whatever delay the loop happens to have — which is precisely the quantity you are trying to measure. Use an **interrupt**, so the timestamp is taken when the pin changes.

```python
from machine import Pin
import time

pir = Pin(14, Pin.IN)      # TODO: your GPIO
MAX = 200
stamps = [0] * MAX         # preallocated — see below
count = 0

def on_motion(pin):
    global count
    if count < MAX:
        stamps[count] = time.ticks_ms()
        count += 1

pir.irq(trigger=Pin.IRQ_RISING, handler=on_motion)

while True:
    time.sleep(1)
```

**Why the list is preallocated.** Allocating memory inside an interrupt handler is restricted in MicroPython. Creating the list up front and only writing into existing slots avoids the problem, and keeps the handler short — a long handler delays everything else on the node.

**Note the bound.** When `count` reaches `MAX`, the node silently stops recording. That is a design decision, not an oversight: a node with finite memory must choose what to do when full — drop new data, overwrite old data, or stop. Yours currently drops new data. Say so in your report.

---

## 9 · Task B — Write the log to flash

So the log survives being unplugged and carried to the machine with the cable:

```python
def save(filename="log.txt"):
    with open(filename, "w") as f:
        for i in range(count):
            f.write("%d\n" % stamps[i])
    print("saved", count, "events")
```

Call it from the Shell when the run finishes, or trigger it from a button.

**Do not write on every event.** Flash wears out, and the write itself would distort the timing you are measuring.

---

## 10 · Task C — Run the experiment

1. Load **identical** code onto both boards.
2. Wire the shared PIR to both, with all grounds joined. Power both. Wait a full minute for the sensor to settle.
3. Trigger one clear, deliberate movement. **This is your shared origin.**
4. Trigger further events every few minutes, for at least **20–30 minutes**. Longer is better — drift is a rate, and a short run gives a slope you cannot tell apart from noise.
5. Call `save()` on each board, then read both logs back through the single cable, one board at a time.

---

## 11 · Task D — Analysis

Line the two logs up by their first event and tabulate:

| Event | A offset (ms) | B offset (ms) | Difference |
|---|---|---|---|
| 1 | 0 | 0 | 0 (by definition) |
| 2 | … | … | … |
| n | … | … | … |

**Plot the difference against elapsed time.** It contains two components, and separating them is the analytical point of the whole week:

| Component | What it is | Can you remove it? |
|---|---|---|
| A **steady** offset, present from the start | Any difference in how quickly each board responds to the signal | **Yes** — it can be calibrated away |
| A **growing** offset, accumulating over the session | Genuine clock drift | **No** — only re-synchronised, repeatedly, forever |

With one shared sensor the steady offset should be very small, so the difference you see is almost all drift. **If you used two different sensors**, expect a larger steady offset — each sensor reacts at a different speed — but the slope is still your drift, and that is what you report.

The slope of the growing component *is* the relative drift between your two boards. Convert it to **ppm**: a drift of *d* milliseconds over *T* seconds is (*d* / 1000) / *T* × 10⁶ ppm.

If your plot is flat, the run was too short or the sensors are mis-set. If it is noisy with no trend, same answer.

---

## 12 · What to submit

Everything goes in `week03/` of your team repository, following the [Submission Guide](../SUBMISSION_GUIDE.md).

- **Code** — the interrupt-driven logger and your save routine
- **Both raw logs**, plus your aligned comparison table
- **Your measured drift**, in ppm, with the sampling duration stated
- **Report §3** — answer all three:
  1. Convert your measured drift to **ppm**. Is it within the 20–50 ppm range quoted in the lecture?
  2. What is the **minimum event separation** your data could reliably resolve? Below that, your two nodes cannot order events at all.
  3. In the **happened-before** sense, are your two detections ordered? Justify using the definition, not intuition.
- **Model spec update** — revise last week's document now that you have measured skew on your own hardware

A raw millisecond difference with no time base is not a measurement. State how long you sampled.

---

## 13 · When it does not work

| Symptom | Most likely cause |
|---|---|
| Constant triggering | Still warming up, or sensitivity turned up too high |
| One movement logs many events | Trigger jumper set to repeat, or hold-time too short |
| One movement logs nothing on one board | Sensors aimed differently, or hold-time still counting from the last event |
| Boards log different numbers of events | With a shared PIR this should not happen — check the joined grounds and Pico B's wire. With two different sensors it is expected |
| Pico B logs nothing at all | Grounds not joined — Pico B has no reference for the signal |
| `log.txt` is empty | `save()` never called, or the board was unplugged first |

If you used two different sensors: **unpaired events are data, not failure.** Pair events by proximity in offset, and say explicitly in your report which ones you could not pair.

---

## 14 · Before Week 4 — homework

Week 4 connects your nodes for the first time, over a wired serial link and over Wi-Fi. **Bringing the Pi Zero up is preparation you do before that session, not during it.** It is mechanical, and doing it in class costs you the whole hour.

**Arrive at Week 4 with a Pi Zero you can log into.** The Week 4 guide, published before the session, walks through the setup.
