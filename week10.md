# Week 10 — Aligning the Clocks

**Time:** about 2 hours 30 min, in two sittings (30 min of it unattended) · **Where:** anywhere with a table and a power outlet — see [LAB_SETUP.md](../LAB_SETUP.md) · **Covered by:** Checkpoint 2

In Week 3 you measured two boards disagreeing about time. This week you synchronise both to the gateway, repeat that experiment, and find out **how much disagreement is left**.

The answer will not be zero. The number that remains — the residual — is your **ε**, and it decides which phenomena you are entitled to fuse.

You are not building a better clock. You are measuring how good your synchronisation actually is, and putting an honest error bar on it.

---

## 1 · Workload budget

| Task | When | Budget |
|---|---|---|
| Wiring, and the gateway time server | Sitting 1 | ~20 min |
| The exchange on both Picos | Sitting 1 | ~30 min |
| Experiment A — how much do the estimates scatter? (twice) | Sitting 1 | ~20 min |
| Experiment B — Week 3 again, synchronised | Sitting 2 | ~40 min, 30 of them unattended |
| Analysis and write-up | Sitting 2 | ~40 min |

Each task ends with a **Check before moving on**.

---

## 2 · ⚠️ Before you start

**Remove the Week 9 jumper wires** (GP14 and GP15 to GND). This week GP14 is the PIR input again.

**Power:** as before — Pico A on your laptop's USB, Pico B from the Pi Zero's 5 V header. **Never plug Pico B's USB in while it is powered from the header.**

**The shared PIR, as in Week 3** — one sensor, seen by both boards:

| PIR pin | Connect to |
|---|---|
| VCC | Pi Zero **pin 4** (5 V) — pin 2 already powers Pico B |
| OUT | **GP14** on Pico A *and* **GP14** on Pico B |
| GND | The common ground. Pico B already shares the Pi Zero's ground; also join **Pico A's GND** to it (e.g. Pi Zero pin 9) |

**All grounds joined is essential** — without it, one Pico logs nothing or nonsense. As in Week 3, check that your PIR module's output is 3.3 V logic before connecting it to a GPIO.

**Do not restart the gateway program during Experiment B.** Gateway time starts from zero when it starts; a restart moves the reference that both Picos have just synchronised to.

---

## 3 · The idea — one reference, everyone else follows

The gateway acts as the time reference. Each Pico estimates its own offset from the gateway, and stamps its events in **gateway time**.

The gateway's clock does **not** have to be correct. It only has to be **shared**. Fusion needs the streams aligned with each other, not with any clock in the outside world. You do not need internet time, GPS, or NTP servers — you need two Picos agreeing with one Pi Zero.

**Use milliseconds, not microseconds.** On many MicroPython ports `ticks_us()` wraps around after about 18 minutes, and a 30-minute experiment would cross that boundary. `ticks_ms()` wraps after days. On Wi-Fi your ε will be milliseconds to tens of milliseconds, so microseconds would only measure noise. *Choose a resolution to match your error, not the finest one available.*

**Port:** `9200 + team number` (team 03 uses 9203).

---

## 4 · Task A — The gateway: a time server

The same program answers time requests from the Picos, and logs the events they report.

```python
# gateway_time.py — on the Pi Zero: time server and event log
import socket, time

TEAM  = 3                                       # TODO: your team number
PORT  = 9200 + TEAM
START = time.monotonic()

def now_ms():                                   # gateway time: ms since this program started
    return int((time.monotonic() - START) * 1000)

srv = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
srv.bind(("0.0.0.0", PORT))
log = open("events.txt", "a")
print("time server on port", PORT)

while True:
    data, addr = srv.recvfrom(128)
    t2 = now_ms()                               # as early as possible
    p = data.decode().split()
    if p[0] == "T":                             # a time request: T <t1>
        t3 = now_ms()                           # as late as possible
        srv.sendto(("%s %d %d" % (p[1], t2, t3)).encode(), addr)
    elif p[0] == "E":                           # an event: E <node> <gateway time> <eps>
        line = "%s %s %s %d" % (p[1], p[2], p[3], t2)
        print(line)
        log.write(line + "\n")
        log.flush()
```

Notice where `t2` and `t3` are taken: `t2` as early as possible, `t3` as late as possible. Anything the server does between them is subtracted out of the delay automatically.

Start it with `python3 gateway_time.py`, and leave it running for the rest of the session.

---

## 5 · Task B — The exchange, on both Picos

Upload `sync_lib.py` to **both** Picos (with `config.py`). Change `NODE` on each.

```python
# sync_lib.py — upload to BOTH Picos
import socket, time, network
from config import SSID, PASSWORD               # see LAB_SETUP.md section 4

NODE   = "pico-a"                               # TODO: pico-a or pico-b
TEAM   = 3                                      # TODO: your team number
SERVER = ("192.168.1.50", 9200 + TEAM)          # TODO: the Pi Zero's address

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, PASSWORD)
for _ in range(30):
    if wlan.isconnected():
        break
    time.sleep(0.5)

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.settimeout(1.0)               # a lost reply is an omission failure: never wait forever

def sample():
    t1 = time.ticks_ms()
    s.sendto(("T %d" % t1).encode(), SERVER)
    data, _ = s.recvfrom(64)                    # raises OSError on timeout
    t4 = time.ticks_ms()
    echo, t2, t3 = [int(x) for x in data.decode().split()]
    if echo != t1:
        raise OSError("stale reply")            # the answer to an earlier, timed-out request
    delay = time.ticks_diff(t4, t1) - (t3 - t2)
    offset = ((t2 - t1) + (t3 - t4)) / 2
    return delay, offset

def sync(n=10):
    results = []
    for _ in range(n):
        try:
            results.append(sample())
        except OSError:
            pass                                # lost or stale: skip it
        time.sleep_ms(100)
    if not results:
        return None
    return min(results, key=lambda r: r[0])     # the shortest round trip: (delay, offset)

def expA(rounds=20, gap_s=3):
    # Experiment A: keep EVERY sample, not just the best one
    with open("expA.csv", "w") as f:
        f.write("round,delay_ms,offset_ms\n")
        for r in range(rounds):
            for _ in range(10):
                try:
                    d, o = sample()
                    f.write("%d,%d,%.1f\n" % (r, d, o))
                    print(r, d, o)
                except OSError:
                    pass
                time.sleep_ms(100)
            time.sleep(gap_s)
    print("saved expA.csv")
```

Three details that matter:

- **`settimeout(1.0)`** — a lost reply is an omission failure. Without a timeout, your node simply stops.
- **The echo check.** If a reply arrives *after* its request timed out, the next `sample()` would read it and pair it with the wrong `t1`. The server sends `t1` back, so a stale reply is recognised and skipped.
- **`sync()` keeps the shortest round trip** — the sample least distorted by queuing, and so the most likely to be symmetric. Its delay is also your error bar: the offset can be wrong by **at most half the round trip**, so **ε = δ / 2**.

**Check before moving on** — in Thonny's Shell on Pico A:

```python
>>> from sync_lib import sync
>>> sync()
(7, -183342.5)
```

You get `(delay, offset)`. Your numbers will differ — the offset is simply the difference between two arbitrary starting points. Run it three times: the delays should be small (a few ms to a few tens of ms), and the offsets should agree to within a few milliseconds. If you get `None`, no reply arrived — check the address and the port.

---

## 6 · Task C — Experiment A: how much do the estimates scatter?

This time keep **every** sample, not just the best one:

```python
>>> from sync_lib import expA
>>> expA()
```

It runs 20 rounds of 10 exchanges, a few seconds apart (about two minutes), and saves `expA.csv` on the Pico. Download it through Thonny's Files pane.

1. **Plot offset against delay** — a spreadsheet is fine. Samples with a larger delay should scatter more.
2. Note the spread of offsets among **only the lowest-delay samples**. That spread is a realistic ε.
3. **Run it twice**: once with the network quiet, and once while it is busy — someone streaming video, for example. Record the conditions.

The relationship between delay and scatter is the **symmetry assumption becoming visible**. Long round trips are long because something queued, and queuing is rarely symmetric.

*(Over two minutes the offset also creeps by a few milliseconds — that is drift, and Experiment B measures it properly.)*

**Check before moving on:** your plot shows a fan shape — tight at low delay, wider at high delay. If every delay is identical, you are probably plotting the wrong column.

---

## 7 · Task D — Experiment B: Week 3 again, synchronised

Now the payoff. Save this as `main.py` on **both** Picos — it synchronises once at start-up, then stamps every PIR event in gateway time and sends it to the gateway:

```python
# main.py — Experiment B, on BOTH Picos
from machine import Pin
import time
from sync_lib import s, SERVER, NODE, sync

best = sync()
while best is None:                             # no reply at all: keep trying
    time.sleep(2)
    best = sync()
DELAY, OFFSET = best
EPS = DELAY / 2                                 # the error bar on OFFSET
print("synced: offset %.1f ms, delay %d ms, eps %.1f ms" % (OFFSET, DELAY, EPS))

pir = Pin(14, Pin.IN)                           # TODO: your Week 3 GPIO
MAX = 200
stamps = [0] * MAX                              # preallocated, as in Week 3
count = 0

def on_motion(pin):
    global count
    if count < MAX:
        stamps[count] = time.ticks_ms()         # raw local time: keep the handler tiny
        count += 1

pir.irq(trigger=Pin.IRQ_RISING, handler=on_motion)

sent = 0
while True:
    while sent < count:                         # convert and report outside the handler
        gw = stamps[sent] + OFFSET              # the same instant, in gateway time
        s.sendto(("E %s %d %.1f" % (NODE, round(gw), EPS)).encode(), SERVER)
        sent += 1
    time.sleep_ms(50)
```

The interrupt handler stores only the raw local tick — the conversion to gateway time happens afterwards, in the main loop. As in Week 3, **keep the handler tiny**.

1. Make sure `gateway_time.py` is running.
2. Power Pico B (it starts by itself), then run `main.py` on Pico A. Pico A prints `synced: offset …, delay …, eps …` in Thonny. Pico B has no screen — but every event it reports carries its ε as the third field on the gateway's line. **Write both ε values down.**
3. Trigger the PIR with one clear movement. The gateway prints **two lines** — one per Pico — with nearly the same time.
4. Trigger again every few minutes, for **20–30 minutes**. You can leave it running in between.
5. Analyse:

```bash
python3 analyze10.py events.txt
```

```python
# analyze10.py — run on the Pi Zero:  python3 analyze10.py events.txt
import sys

ev = {"pico-a": [], "pico-b": []}
eps = {}
for line in open(sys.argv[1]):
    node, t, e, arrival = line.split()
    ev[node].append(int(t))
    eps[node] = float(e)

pairs = []                                       # the same PIR trigger, seen by both boards
for ta in ev["pico-a"]:
    near = [tb for tb in ev["pico-b"] if abs(tb - ta) < 1000]
    if near:
        tb = min(near, key=lambda x: abs(x - ta))
        pairs.append((ta, ta - tb))

if len(pairs) < 2:
    sys.exit("need at least two events seen by both boards")
t0 = pairs[0][0]
print("minutes  residual_ms (A - B)")
for t, r in pairs:
    print("%7.2f  %6d" % ((t - t0) / 60000, r))

xs = [(t - t0) / 60000 for t, _ in pairs]        # minutes
ys = [r for _, r in pairs]
n = len(xs); mx = sum(xs) / n; my = sum(ys) / n
sxx = sum((x - mx) ** 2 for x in xs)
slope = sum((x - mx) * (y - my) for x, y in zip(xs, ys)) / sxx if sxx else 0.0
print("\npairs: %d   first residual: %d ms   slope: %.2f ms/min  (= %.1f ppm)"
      % (n, ys[0], slope, slope / 60000 * 1e6))
print("eps: pico-a %.1f ms, pico-b %.1f ms  ->  pair bound %.1f ms"
      % (eps.get("pico-a", 0), eps.get("pico-b", 0), eps.get("pico-a", 0) + eps.get("pico-b", 0)))
```

**This time you do not need the "align on the first event" trick from Week 3.** The timestamps are in a shared time base now, so you compare them directly. That change is the whole point of the week.

**Check before moving on:** the first trigger produces one line from each Pico, and `analyze10.py` pairs them.

---

## 8 · What you should find

Right after synchronising, the two boards agree to within roughly **ε_A + ε_B** — a large improvement on Week 3.

Then, over the session, the disagreement **grows again**, at roughly the drift rate you measured in Week 3. Synchronisation removed the **offset**. It did nothing about the **drift**, because it cannot: the two crystals still run at different rates.

From your slope, work out **how often you would need to resynchronise** to keep the residual below a target — 50 ms, for example. That interval is a real design parameter, and you now have the numbers to set it.

---

## 9 · Optional — keep it bounded

Add a resynchronisation every few minutes to `main.py` — call `sync()` again and replace `OFFSET` and `EPS` — and repeat Experiment B. The residual should now stay roughly flat instead of growing. Choose the interval from your own slope, and say so.

---

## 10 · Answer these in your report

**Report §3** has three parts:

1. **What is your ε, and how did you derive it?**
2. **How often must you resynchronise to keep the residual under 50 ms?**
3. **Using the fusion rule from the lecture, name one phenomenon your cluster can now fuse reliably, and one it still cannot.**

The third matters most. An answer that names a phenomenon you *cannot* measure, and says why, shows you understood the rule — not just the protocol.

---

## 11 · What to submit

Everything goes in `week10/` — see the [Submission Guide](../SUBMISSION_GUIDE.md) and the [Report Guide](../REPORT_GUIDE.md).

- **Code** — `gateway_time.py` (time server), `sync_lib.py` (sync routine), and `main.py` (your Week 3 logger, now using gateway time)
- **Experiment A data** — `expA.csv` from both runs, the plot, and your derived ε
- **Experiment B data** — `events.txt` and the `analyze10.py` output: the residual per event, over the session
- **Report §3** — the three questions in §10
- **Model spec update** — your timing section can now state an actual bound (ε) and a resynchronisation interval

---

## 12 · When it does not work

| Symptom | Most likely cause |
|---|---|
| `sync()` returns `None` | No reply at all — wrong gateway address or port, or `gateway_time.py` not running |
| Offset jumps wildly between runs | Using every sample instead of the lowest-delay one |
| Offset is off by a large round number | Units mismatch — seconds on one side, milliseconds on the other |
| Residuals jump suddenly partway through | The gateway program was restarted, so gateway time restarted at zero |
| Negative delay | `t2` and `t3` swapped, or taken in the wrong order |
| Only one Pico reports events | Grounds not joined, or one OUT wire on the wrong pin |
| A Pico reports events nobody made | A Week 9 jumper wire is still on GP14 |

---

## 13 · Checkpoint 2

**Checkpoint 2 covers Weeks 8, 9 and 10**, and closes at **23:59 on Tuesday 1 December**. Classes continue next week as normal.

1. Tag your repository `cp2` and push the tag.
2. Upload **one PDF per team** to iClass — a cover page listing **every member's name, student ID and GitHub username**, your Week 8–10 reports, and `model-spec.md`. An integrated report bringing the three weeks together is welcome, not required.

Check before you upload:

- The **Week 8 video** is in the repository, and **plays on GitHub**.
- `model-spec.md` has three new sections: **detection** (Week 8), **election** (Week 9), and **timing bound** — your ε and resynchronisation interval (Week 10).

The full procedure is in [CHECKPOINT_SUBMISSION.md](../CHECKPOINT_SUBMISSION.md).

---

## 14 · Next week

The camera finally comes out. Inference runs on the Pi Zero because it has to — a Pico cannot even hold an image — and you will measure what that costs compared with sending raw images somewhere else.

**Check now that you have a camera cable that fits the Pi Zero.** The cable supplied with the camera module does not. Without the right one, next week cannot start.
