# Week 06 — Detecting Concurrency

**Time:** about 2 hours 10 min, in two sittings · **Where:** anywhere with a table and a power outlet — see [LAB_SETUP.md](../LAB_SETUP.md) · **Covered by:** Checkpoint 1

Two things this week, and the second matters more:

1. **Implement** a three-element vector clock on every update, and a gateway that classifies each pair of updates as *causally ordered* or *concurrent*.
2. **Break the alternative.** Create a genuinely concurrent pair, resolve it with wall-clock timestamps instead, and show exactly what gets destroyed.

Implementing a vector clock is a coding exercise. Demonstrating what it buys you is the deliverable.

---

## 1 · Workload budget

| Task | When | Budget |
|---|---|---|
| Clock, message format, comparison function | Sitting 1 | ~35 min |
| Gateway classifier, and experiment A | Sitting 1 | ~25 min |
| Experiment B, then C on the same data | Sitting 2 | ~40 min |
| Write-up and model spec update | Sitting 2 | ~30 min |

Each task ends with a **Check before moving on**. Do not start the next task until you see that result.
| Persisting the clock across reboot | Optional | ~25 min |

---

## 2 · ⚠️ Before you start

**Power:** as in Week 5, both Picos and the Pi Zero run at once. Pico A on your laptop's USB; Pico B from the Pi Zero's 5 V header (Pi Zero pin 2 → Pico B pin 39 `VSYS`, GND → GND). **Never plug Pico B's USB in while it is powered from the header.**

**Network:** all three boards and your laptop on one 2.4 GHz network. If you are working somewhere new, follow [LAB_SETUP.md](../LAB_SETUP.md) §3–5 first, and confirm the broker works with `mosquitto_sub` before touching the Picos.

**Reprogramming a Pico resets its vector clock to zero.** Finish all your code changes *before* starting the experiments, and do not re-upload in the middle of them. §9 explains why this matters.

---

## 3 · Setup — keep the shared state tiny

You need one small value that more than one node may update. Use a single integer, `threshold`, which either Pico may change. One number, three nodes.

The phenomenon has nothing to do with what the value means. A large payload only makes your logs harder to read.

All three nodes use one shared topic: `teamNN/threshold`.

### Decide this first: who is which index

| Index | Node |
|---|---|
| 0 | `pico-a` |
| 1 | `pico-b` |
| 2 | `gateway` |

Every node must agree on this mapping, permanently. If two nodes both think they are index 0, they increment the same slot, and the vectors become meaningless — **while still looking perfectly reasonable.**

The gateway only observes this week, so slot 2 stays at 0. It is reserved for when the gateway takes part, from Week 9.

Write the mapping into `model-spec.md`. It is now part of your system's specification.

---

## 4 · Task A — The clock, on both Picos

```python
ME = 0                      # TODO: 0 on Pico A, 1 on Pico B
vc = [0, 0, 0]

def tick():
    vc[ME] += 1

def merge(other):
    for i in range(len(vc)):
        if other[i] > vc[i]:
            vc[i] = other[i]
    vc[ME] += 1
```

Eleven lines. Compare that with what it lets you conclude.

**Check before moving on** — in Thonny's Shell on Pico B (`ME = 1`):

```python
>>> tick(); tick(); print(vc)
[0, 2, 0]
>>> merge([3, 0, 0]); print(vc)
[3, 3, 0]
```

If you see anything else, the clock is wrong — fix it now. Every later result depends on it. *(Reset the board afterwards so `vc` starts again from zero.)*

### What travels in a message

```python
import json, time

def make_update(value):
    tick()
    return json.dumps({
        "node":  ME,
        "vc":    list(vc),      # a copy, not a reference
        "value": value,
        "wall":  time.ticks_ms()
    })
```

`list(vc)` matters: sending the live list is a bug waiting to happen once you change it between building and sending.

The `wall` field is included **only** for experiment C. It plays no part in correctness — which is exactly the point being made.

---

## 5 · Task B — Sending and receiving on the Picos

Each Pico publishes when you **press its BOOTSEL button**, so you control exactly when updates happen. *(If `rp2.bootsel_button` is missing on your firmware, use a GPIO with a pull-up and touch a jumper wire to GND instead.)*

A jumper wire is your **"network cable" switch** for experiment B: while GP15 (pin 20) is connected to a GND pin (pin 18), the node ignores everything it receives.

```python
from umqtt.simple import MQTTClient
from machine import Pin
import rp2, json, time

CLIENT_ID = b"team03-pico-a"          # TODO: unique per node
BROKER    = "192.168.1.50"            # TODO: your Pi Zero
TOPIC     = b"team03/threshold"       # TODO: your team

# Connect to Wi-Fi first, exactly as in Week 4.
# Paste your clock code (Task A) here too.

isolate = Pin(15, Pin.IN, Pin.PULL_UP)   # wire to GND = cut off
threshold = 50

def on_update(topic, payload):
    global threshold
    if isolate.value() == 0:
        print("isolated - ignoring", payload)
        return
    m = json.loads(payload)
    if m["node"] == ME:
        return                        # our own message, echoed back
    merge(m["vc"])
    threshold = m["value"]
    print("received", m["vc"], "->", vc)

c = MQTTClient(CLIENT_ID, BROKER, keepalive=30)
c.set_callback(on_update)
c.connect()
c.subscribe(TOPIC)

while True:
    c.check_msg()                     # deliver any waiting message
    if rp2.bootsel_button():
        threshold += 1
        msg = make_update(threshold)
        c.publish(TOPIC, msg.encode())
        print("sent", msg)
        while rp2.bootsel_button():   # wait for release
            time.sleep_ms(20)
    time.sleep_ms(20)
```

**Why the echo check:** you are subscribed to the topic you publish on, so the broker sends your own messages back to you. Without the check, your clock advances twice for one event.

**Check before moving on** — start both Picos fresh, with no isolation wire, and watch both Shells. In a second terminal on the Pi Zero, run `mosquitto_sub -h localhost -t "teamNN/threshold" -v`.

1. Press BOOTSEL on Pico A **once**. Pico A prints `sent` exactly once, with `"vc": [1, 0, 0]`. `mosquitto_sub` shows the same message. Pico B prints `received [1, 0, 0] -> [1, 1, 0]`.
2. Press BOOTSEL on Pico B once. It sends `[1, 2, 0]`, and Pico A prints `received [1, 2, 0] -> [2, 2, 0]`.

Then restart both Picos so their clocks begin from zero for the experiments.

---

## 6 · Task C — The comparison, and the gateway

```python
def compare(a, b):
    le = all(x <= y for x, y in zip(a, b))
    ge = all(x >= y for x, y in zip(a, b))
    if le and ge:
        return "equal"
    if le:
        return "a happened-before b"
    if ge:
        return "b happened-before a"
    return "concurrent"
```

The final line is what a Lamport clock could never tell you (Week 3). "Neither one dominates" is a real, useful answer — not a failure to decide.

**On the Pi Zero**, keep the latest update from each node and compare each new one against it:

```python
import paho.mqtt.client as mqtt
import json, time

seen = {}                             # node index -> its latest update
log = open("vc_log.txt", "a")

def compare(a, b):
    le = all(x <= y for x, y in zip(a, b))
    ge = all(x >= y for x, y in zip(a, b))
    if le and ge: return "equal"
    if le: return "a happened-before b"
    if ge: return "b happened-before a"
    return "concurrent"

def on_message(client, userdata, msg):
    m = json.loads(msg.payload)
    for node, prev in seen.items():
        if node != m["node"]:
            rel = compare(prev["vc"], m["vc"])
            line = "%s  a=%s  b=%s  -> %s" % (time.strftime("%H:%M:%S"), prev["vc"], m["vc"], rel)
            print(line)
            log.write(line + "\n")
            log.flush()
    seen[m["node"]] = m

try:
    c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
    c = mqtt.Client()
c.on_message = on_message
c.connect("localhost", 1883, 60)
c.subscribe("team03/threshold")       # TODO: your team
c.loop_forever()
```

That log, with both vectors on every line, is your evidence file.

**Check before moving on** — test `compare()` on your laptop first, in plain Python:

| Call | Must return |
|---|---|
| `compare([2,1,0], [3,1,0])` | `a happened-before b` |
| `compare([3,1,0], [2,1,0])` | `b happened-before a` |
| `compare([2,0,0], [0,0,1])` | `concurrent` |
| `compare([1,2,0], [1,1,3])` | `concurrent` |
| `compare([2,1,0], [2,1,0])` | `equal` |

Then start the gateway. It prints nothing for the very first update — there is nothing to compare it with yet — and one line for every update after that.

---

## 7 · Task D — Three experiments

### A · Ordinary operation

1. Start all three nodes. Both isolation wires **disconnected**.
2. Press BOOTSEL on Pico A. Check Pico B prints `received`.
3. Press BOOTSEL on Pico B.
4. Read the gateway's classification.

**Expect `a happened-before b`.** Pico B saw A's update before producing its own, so a causal path exists. If you do not get this, **stop and fix it** — B and C are meaningless on a broken implementation.

### B · Manufacture genuine concurrency — *capture once, use twice*

1. Connect Pico B's isolation wire (GP15 to GND). Pico B is now cut off.
2. Press BOOTSEL on Pico A. Pico B should print `isolated - ignoring`.
3. Press BOOTSEL on Pico B. It produces an update **without having seen A's**.
4. Disconnect the isolation wire.

**Expect `concurrent`.** Neither update is in the other's causal past. Both are legitimate, and no order between them exists anywhere in the system.

This is a **simulated partition**: the radio is still on, but the node behaves as if the messages never arrived. You will use the same idea in Week 9. Say so in your report.

**Save both messages from this pair.** Experiment C uses them — there is nothing further to run.

### C · Now resolve the same pair with a clock — *analysis only*

Take the two messages from B, including their `wall` fields, and apply last-write-wins:

1. Compare the two `wall` values, and keep the larger.
2. Record which update was discarded.
3. State what information was lost, and whether anything in the system could have detected the loss afterwards.

Then the question that matters: **what are your `wall` values actually measuring?** Recall Week 3 — these two boards do not share a starting point. Is comparing them even meaningful?

---

## 8 · What to record

| Experiment | Report |
|---|---|
| A | Both vectors, the classification, and why it is correct |
| B | Both vectors, the classification, and how you know the isolation worked |
| C | Which update LWW kept, which it destroyed, and what that data was |

For C, the strongest answer names a **consequence**: something your system would now do differently, or report incorrectly, because of the discarded update.

---

## 9 · Optional — what happens to the clock on reboot?

A Pico that resets comes back with `vc = [0, 0, 0]`. Its next update will appear to come **before** everything — including updates it had already sent. That is Week 2's crash-recovery model, arriving with consequences: a node that forgets its place in logical time corrupts the record of every node it talks to.

Save the vector to flash after every change, and reload it at start-up:

```python
def save_vc():
    with open("vc.txt", "w") as f:
        f.write(",".join(str(x) for x in vc))

def load_vc():
    try:
        return [int(x) for x in open("vc.txt").read().split(",")]
    except OSError:
        return [0, 0, 0]
```

Then test it: reset a Pico mid-session and confirm the classifications stay sensible. Writing on every update is fine here — updates only happen when you press a button.

---

## 10 · What to submit

Everything goes in `week06/` of your team repository — see the [Submission Guide](../SUBMISSION_GUIDE.md).

- **Code** — clock, comparison function, publisher, and gateway classifier
- **Logs** — all three experiments, with both vectors visible for every classified pair
- **Report §3** — experiment C: what last-write-wins destroyed, and what consequence that would have had
- **Model spec update** — your node-to-index mapping and your conflict policy. You now have one; say what it is.

---

## 11 · When it does not work

| Symptom | Most likely cause |
|---|---|
| Everything classified as concurrent | Picos are not receiving each other's updates — check the subscription and `check_msg()` |
| Nothing ever classified as concurrent | The isolation wire is on the wrong pin, or not reaching GND |
| Vectors grow only in one slot | Two nodes sharing the same `ME` |
| One press advances the clock twice | Not filtering your own echoed message |
| Classifications reverse partway through | A Pico was reset or reprogrammed — see §9 |
| One press sends several updates | The release-wait loop is missing |

---

## 12 · Next week is Checkpoint 1

There is **no lab next week**. Week 7 is Checkpoint 1, covering Weeks 2 to 6. By **23:59 on Sunday of Week 7**:

1. Tag your repository `cp1` and push the tag.
2. Upload one PDF — cover page, every `report.md` for Weeks 2–6, and `model-spec.md` — to iClass.

The full procedure is in [CHECKPOINT_SUBMISSION.md](../CHECKPOINT_SUBMISSION.md). Budget fifteen minutes for the PDF, and check on GitHub, not on your laptop, that everything is pushed.
