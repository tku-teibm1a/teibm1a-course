# Week 08 — Building a Failure Detector

**Time:** about 2 hours 30 min, in two sittings (20 min of it unattended) · **Where:** anywhere with a table and a power outlet — see [LAB_SETUP.md](../LAB_SETUP.md) · **Covered by:** Checkpoint 2

Two things this week:

1. **Build** heartbeats from both Picos, and a timeout-based detector on the Pi Zero that marks a node as *suspected*.
2. **Characterise** it. Measure how fast it detects a real failure, and how often it raises a false alarm, at two timeout values — and choose one.

The code is about forty lines. The measurement is the work, and the comparison is what you are really submitting.

---

## 1 · Workload budget

| Task | When | Budget |
|---|---|---|
| Heartbeat and freeze command on both Picos | Sitting 1 | ~25 min |
| Detector on the Pi Zero, first trials | Sitting 1 | ~35 min |
| Detection-time trials at two timeouts (5 each) | Sitting 2 | ~30 min |
| False-alarm runs (2 × 10 min, unattended) | Sitting 2 | ~10 min of your time |
| Reaction on suspicion, video, write-up | Sitting 2 | ~45 min |
| Continuous suspicion level | Optional | ~30 min |

Each task ends with a **Check before moving on**. Do not start the next task until you see that result.

---

## 2 · ⚠️ Before you start

**Power:** as in Weeks 5 and 6 — Pico A on your laptop's USB, Pico B from the Pi Zero's 5 V header. **Never plug Pico B's USB in while it is powered from the header.**

**Network:** all three boards and your laptop on one 2.4 GHz network. If you are somewhere new, follow [LAB_SETUP.md](../LAB_SETUP.md) §3–5 first.

**Pico B must run on its own**, without Thonny. Save its program as `main.py` on the board — together with your `config.py` — so it starts by itself when powered from the header.

---

## 3 · Design first — three decisions

Make these before writing anything, and record them in `model-spec.md`.

| Decision | Options | Notes |
|---|---|---|
| **Push or pull** | The node announces itself, or the gateway asks | Push is cheaper, and it is what MQTT is built for. This guide uses push. |
| **Period** | How often a heartbeat is sent | Shorter detects faster, and costs more traffic on a shared link. The guide starts at 1 s. |
| **Dedicated or piggybacked** | A separate heartbeat message, or treat any data message as proof of life | Piggybacking has a trap: a node that is alive but has nothing to report looks exactly like a dead one. |

### A useful realisation: this needs no shared clock

The Pi Zero records when each heartbeat *arrives*, using **its own clock**, and compares the time since then with a timeout. The sender's clock is never used.

After five weeks of learning that comparing clocks across nodes is unsound, here is a mechanism that avoids the problem completely — by only measuring durations on one machine. Many clock problems disappear if you can restate the question as *"how long since I last heard something?"*

---

## 4 · Task A — Heartbeats, and a freeze command

Each Pico sends a numbered heartbeat every second. It also listens for a **freeze** command: on receipt, it stops everything for a number of seconds — still powered, still connected, completely silent.

That freeze is how you get **honest ground truth** for detection time (§6). It also models a *hang* rather than a power loss — the more interesting failure: the node is still there, and useless.

```python
# main.py on each Pico
from umqtt.simple import MQTTClient
import network, time
from config import SSID, PASSWORD, BROKER    # see LAB_SETUP.md §4

TEAM      = "team03"         # TODO: your team
NODE      = "pico-a"         # TODO: pico-a or pico-b
PERIOD_MS = 1000             # TODO: your chosen period
HB_TOPIC  = (TEAM + "/" + NODE + "/hb").encode()
CMD_TOPIC = (TEAM + "/" + NODE + "/cmd").encode()

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, PASSWORD)
for _ in range(30):                          # bounded wait, as in Week 4
    if wlan.isconnected():
        break
    time.sleep(0.5)

freeze_ms = 0

def on_cmd(topic, msg):
    global freeze_ms
    parts = msg.decode().split()
    if parts and parts[0] == "freeze":
        freeze_ms = int(parts[1]) * 1000 if len(parts) > 1 else 20000

c = MQTTClient((TEAM + "-" + NODE).encode(), BROKER, keepalive=30)
c.set_callback(on_cmd)
c.connect()
c.subscribe(CMD_TOPIC)

seq = 0
next_hb = time.ticks_ms()
while True:
    c.check_msg()
    if freeze_ms:
        print("frozen for", freeze_ms, "ms")
        time.sleep_ms(freeze_ms)             # the hang
        freeze_ms = 0
        next_hb = time.ticks_ms()            # no burst of catch-up heartbeats
        print("resumed")
    if time.ticks_diff(time.ticks_ms(), next_hb) >= 0:
        c.publish(HB_TOPIC, str(seq).encode())
        seq += 1
        next_hb = time.ticks_add(next_hb, PERIOD_MS)
    time.sleep_ms(10)
```

**Why the sequence number:** it costs nothing, and it lets you tell *"the node stopped"* apart from *"some heartbeats were lost"* — two very different situations that a detector otherwise confuses. It also shows a reboot: the count starts again from 0.

**Keep every freeze under 30 seconds.** With `keepalive=30`, the broker closes a connection that is silent for 45 s (1.5 × 30 — this week's lecture explains why). A longer freeze makes the Pico lose its connection.

**Check before moving on** — on the Pi Zero:

```bash
mosquitto_sub -h localhost -t "team03/+/hb" -v
```

1. You see one line per second from **each** Pico, and each count goes up by one.
2. In a second terminal: `mosquitto_pub -h localhost -t "team03/pico-a/cmd" -m "freeze 5"`. Pico A's lines stop for about 5 s, then continue **with no number skipped**. Repeat for Pico B.

---

## 5 · Task B — The detector, on the Pi Zero

```python
# detector.py  —  run:  python3 detector.py <timeout_s> <label>
import paho.mqtt.client as mqtt
import sys, time, threading

TEAM      = "team03"                 # TODO: your team
TIMEOUT_S = float(sys.argv[1])       # the variable under study
LABEL     = sys.argv[2]              # e.g. short, long
CHECK_S   = 0.1                      # how often check() runs

last_seen, last_seq, suspected = {}, {}, set()
lock = threading.Lock()
logf = open("log_%s.txt" % LABEL, "a")

def log(event, node, extra=""):
    line = "%.3f %s %s %s" % (time.monotonic(), event, node, extra)
    print(time.strftime("%H:%M:%S"), line)
    logf.write(line + "\n")
    logf.flush()

def react(client, node, state):
    # TODO: your chosen reaction (§8). This example publishes an alert.
    client.publish("%s/status/%s" % (TEAM, node), state, retain=True)

def on_message(client, userdata, msg):
    node = msg.topic.split("/")[1]
    seq = int(msg.payload)
    with lock:
        if node in last_seq and seq != last_seq[node] + 1:
            log("GAP", node, "expected %d got %d" % (last_seq[node] + 1, seq))
        last_seq[node] = seq
        last_seen[node] = time.monotonic()

def check(client):
    now = time.monotonic()
    with lock:
        for node, t in list(last_seen.items()):
            late = now - t
            if late > TIMEOUT_S and node not in suspected:
                suspected.add(node)
                log("SUSPECT", node, "late=%.3f" % late)
                react(client, node, "suspected")
            elif late <= TIMEOUT_S and node in suspected:
                suspected.discard(node)
                log("REVIVE", node)
                react(client, node, "alive")

def keyboard(client):
    # type:  f pico-a 20   to freeze pico-a for 20 s
    while True:
        try:
            p = input().split()
        except EOFError:
            return
        if len(p) >= 2 and p[0] == "f":
            secs = p[2] if len(p) > 2 else "20"
            log("FREEZE", p[1], secs)
            client.publish("%s/%s/cmd" % (TEAM, p[1]), "freeze " + secs)

try:
    c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
    c = mqtt.Client()
c.on_message = on_message
c.connect("localhost", 1883, 60)
c.subscribe(TEAM + "/+/hb")
c.loop_start()
threading.Thread(target=keyboard, args=(c,), daemon=True).start()
log("START", "-", "timeout=%.2f" % TIMEOUT_S)
while True:
    check(c)
    time.sleep(CHECK_S)
```

Three things to notice:

- **`time.monotonic()`, not `time.time()`.** A wall clock can jump — for example when the Pi Zero corrects its time over the network. A monotonic clock never goes backwards, so elapsed times are always correct.
- **Both SUSPECT and REVIVE are logged.** A SUSPECT followed by a REVIVE, with no freeze in between, is a **false suspicion**: the node was alive all along, and the network was only slow. Without the REVIVE line you could not count them.
- **The freeze time is logged on the gateway's clock**, in the same file as the SUSPECT. That is what makes the detection time honest: both ends of the measurement come from one clock.

**Check before moving on:**

1. `python3 detector.py 3.0 test`. Leave it for one minute: there should be **no** SUSPECT lines.
2. Type `f pico-a 10` and press Enter. About 3 s later you see `SUSPECT pico-a`; about 10 s after the freeze, `REVIVE pico-a`.
3. In another terminal, `mosquitto_sub -h localhost -t "team03/status/#" -v` shows `suspected`, then `alive`.

---

## 6 · Task C — Experiment 1: detection time

Choose your two timeout values: one **aggressive** and one **conservative**. For a 1 s period, for example, **1.5 s** and **6 s**.

For each value:

1. Start `python3 detector.py 1.5 short` (then later `6 long`).
2. Type `f pico-a 20`. Wait for SUSPECT, then for REVIVE.
3. Repeat — **five trials** per timeout value. Alternate between `pico-a` and `pico-b` if you like. A single trial tells you almost nothing on a variable network.
4. Stop the detector with Ctrl-C.

Then use this small script to read the numbers out of a log:

```python
# analyze.py  —  run:  python3 analyze.py log_short.txt
import sys, statistics

pending, open_suspect = {}, {}
detect, false_alarms = [], 0
first = last = None

for line in open(sys.argv[1]):
    p = line.split()
    t, kind, node = float(p[0]), p[1], p[2]
    first = t if first is None else first
    last = t
    if kind == "FREEZE":
        pending[node] = t
    elif kind == "SUSPECT":
        if node in pending:
            detect.append(t - pending.pop(node))
        else:
            open_suspect[node] = t
    elif kind == "REVIVE" and node in open_suspect:
        del open_suspect[node]
        false_alarms += 1

if detect:
    print("detection: n=%d  min=%.2f  median=%.2f  max=%.2f s" % (
        len(detect), min(detect), statistics.median(detect), max(detect)))
hours = (last - first) / 3600
if hours > 0:
    print("false suspicions: %d in %.2f h  ->  %.1f per node per hour" % (
        false_alarms, hours, false_alarms / hours / 2))
```

**What to expect — and explain.** The detection time will **not** equal your timeout. It falls in a window of roughly *timeout − one heartbeat period* to *timeout + a little*. In your report, explain where your values fall and why. There are at least three causes, and all of them are in this week's code:

- When the freeze arrived, how long ago was the last heartbeat?
- How often does `check()` run?
- How long does the freeze command take to reach the Pico?

**Check before moving on:** `analyze.py` reports `n=5` for each log, and every detection time is within about one heartbeat period of the timeout. If not, see §11.

---

## 7 · Task D — Experiment 2: false suspicions

Now the opposite measurement. Leave everything **running and healthy**, and count how often the detector cries wolf.

1. `python3 detector.py 1.5 short_idle` — run for **10 minutes**, unattended. Do not freeze anything.
2. Then the same with `6 long_idle`.
3. Run `analyze.py` on each. It reports false suspicions **per node per hour**.

**Measure on a realistic network.** Run this while the network is in normal use — someone streaming video, a phone downloading an update, the microwave on if your router is near the kitchen. A false-alarm rate measured on an empty network describes a network you will never deploy on. **Write down the conditions** in your report.

If you get zero false alarms with both values, that is a real result — say so, and say what conditions you measured under. You may then try a shorter timeout (for example 1.1 × the period) to find where alarms begin.

---

## 8 · Task E — Do something on suspicion

A detector that only prints a message has not made your system fault-tolerant. **Choose and implement one reaction** in `react()`:

| Reaction | What it does |
|---|---|
| **Alert** | Publish a status message that a subscriber could act on *(the example code)* |
| **Mark stale** | Readings from that node stop being treated as current |
| **Degrade** | The gateway carries on with only the surviving node, and says so |

**Handle the REVIVE path too.** A node that comes back must be brought back in. Look at the GAP lines: a node that **rebooted** comes back with its count starting at 0 — as Week 6 showed, a node that restarts has forgotten things, and cannot simply carry on where it stopped.

---

## 9 · Task F — The comparison, and your choice

Fill in this table from your own runs:

| Timeout | Detection time (min / median / max) | False suspicions per node per hour |
|---|---|---|
| short: … s | … | … |
| long: … s | … | … |

Two points do not make a curve, but they show which way it runs — and that is the whole argument. Each is a detector you could ship. Neither is "correct".

**Then choose one for your system, and defend it.** Use:

- your **Week 4 delay measurements** — how variable is your link?
- **what your reaction costs.** If it is cheap and reversible — mark a reading stale — you can afford to suspect early. If it is expensive — fail over, discard data, wake someone up — you should demand more evidence.

A choice that is defended with your own numbers is a good answer, even if another team chose differently.

---

## 10 · Task G — Video evidence

This is one of the weeks that needs **video**. Record **one continuous clip, under 60 seconds**, showing:

1. the system running normally,
2. a Pico being **physically unplugged** — Pico B's 5 V wire, or Pico A's USB cable,
3. the detector reacting.

Frame it so the **hardware and the detector's output are visible together**. The point is the link between the unplug and the reaction — not a screen recording of a log that could have come from anywhere.

**File size:** record at 720p or 1080p, not 4K. GitHub refuses files over 100 MB; aim for under 50 MB. Put it in `week08/evidence/`.

---

## 11 · Optional — make suspicion continuous

Instead of one fixed threshold, keep the last 50 heartbeat intervals for each node, and compute how unusual the current silence is:

```python
suspicion = (late - mean_interval) / stdev_interval
```

Two reactions can then read the **same** detector at different thresholds — a cheap one triggering early, an expensive one waiting for stronger evidence. This is the idea behind the *accrual* detector from the lecture.

Only attempt this once the fixed-threshold version is measured and written up. A half-finished extension is worth less than a complete simple detector.

---

## 12 · What to submit

Everything goes in `week08/` — see the [Submission Guide](../SUBMISSION_GUIDE.md) and the [Report Guide](../REPORT_GUIDE.md).

- **Code** — heartbeat sender, detector, and your chosen reaction
- **Raw logs** — all trials, both timeout values, both false-alarm runs
- **The comparison table** — detection time and false suspicions, at both values
- **Video** — under 60 seconds, unplug and reaction in one shot
- **Report §3** — which timeout you chose for your system, and why, citing your own numbers
- **Model spec update** — your detection rule (period, timeout, reaction, and whether heartbeats are dedicated or piggybacked) is now part of your system's specification

---

## 13 · When it does not work

| Symptom | Most likely cause |
|---|---|
| A node is suspected constantly | Timeout shorter than the heartbeat period |
| A node is never suspected after a freeze | The freeze command never arrived — check the topic name with `mosquitto_sub -t "team03/#" -v` |
| Detection time is exactly the timeout, every trial | Suspicious — check you measure from the FREEZE line, not from the last heartbeat |
| `GAP expected 7 got 0` | That Pico rebooted — its count started again |
| The Pico prints `OSError` after a freeze | The freeze was longer than 45 s and the broker closed the connection — keep freezes under 30 s |
| Pico B never sends anything | It is not running `main.py` — Pico B has no Thonny to start it |
| A frozen Pico still shows as connected in the broker | Normal: the connection survives a hung program for up to 45 s. A live connection is not a live application — the Week 4 point about delivery versus processing |

---

## 14 · Next week

Your detector can now tell you that a node has gone. It cannot tell the remaining nodes **who should take over** — and if two of them decide that they should, things get worse, not better. Week 9 is about exactly that: elections, and why a majority keeps appearing in every answer. In the lab you will run an election and watch it go wrong.
