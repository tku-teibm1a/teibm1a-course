# Week 05 — Three Nodes and a Broker

**In class:** 1 hour · **Outside class:** ~1 hour · **Covered by:** Checkpoint 1

Until now you had wires between pairs of devices. After today you have something that deserves to be called a system — with one component at its centre that everything depends on.

---

## 1 · What you will have at the end

- An MQTT broker running as a service on your Pi Zero
- Both Picos publishing to it, and the Pi Zero subscribing to both
- Three experiments that make this week's lecture visible

**Optional:** the I²C sensors (ambient light, pressure). Nothing above depends on them.

---

## 2 · Workload budget

| Task | When | Budget |
|---|---|---|
| Broker install and local test | In class | ~20 min |
| Both Picos publishing, Pi Zero subscribing | In class | ~40 min |
| Three experiments + write-up | After class | ~60 min |
| I²C sensors | Optional | ~45 min |

The three experiments are what is assessed. If time runs out, **skip the I²C sensors, not the experiments** — and say in your report that you did.

---

## 3 · ⚠️ Powering two Picos at once

This week **both Picos and the Pi Zero must run at the same time**. Your kit has one USB cable for the Picos, and the Pi Zero now needs its own power supply for the broker.

**Pico A:** USB cable from your laptop, as usual.

**Pico B:** power it from the **Pi Zero's 5 V header pin**:

| Pi Zero | Pico B |
|---|---|
| Pin 2 or 4 (5 V) | Pin 39 (VSYS) |
| Pin 6 (GND) | Pin 38 (GND) |

Pico B's `VSYS` pin accepts 1.8–5.5 V, and the Pi Zero's power supply has plenty to spare for it.

**Never plug Pico B's USB cable in while it is powered from the header.** That connects two power supplies together. To reprogram Pico B, remove the 5 V wire first.

*(A power bank or charger with a micro-USB cable also works, if you have one.)*

---

## 4 · Task A — The broker

### Install

```bash
sudo apt update
sudo apt install mosquitto mosquitto-clients
sudo systemctl enable mosquitto
sudo systemctl status mosquitto
```

`enable` makes the broker start automatically at boot, so your Pi Zero recovers by itself after a power cut. This is the first thing you have built that is meant to run unattended, indefinitely.

### The trap everyone hits: nobody can connect

Recent Mosquitto versions only accept connections from the Pi Zero itself. Your Picos will fail to connect — with no useful error on their side. Create `/etc/mosquitto/conf.d/local.conf`:

```
listener 1883 0.0.0.0
allow_anonymous true
```

Then `sudo systemctl restart mosquitto`.

`allow_anonymous true` means anyone on the network can read and write everything. That is acceptable on an isolated teaching network and indefensible anywhere else. Know that you are making that trade.

MQTT uses its standard port **1883**. Each team's broker runs on its own Pi Zero, so this does not clash between teams.

### Test the broker before involving the Picos

Two terminals on the Pi Zero:

```bash
# terminal 1
mosquitto_sub -h localhost -t "test/#" -v

# terminal 2
mosquitto_pub -h localhost -t "test/hello" -m "it works"
```

Always isolate the layer you are testing. If this fails, the problem is the broker. If it works and a Pico still cannot connect, the problem is the network or the client.

---

## 5 · Names: eight teams, one network

| | Rule | Example |
|---|---|---|
| Topic | team / node / quantity | `team03/pico-a/temperature` |
| Client ID | team-prefixed, unique | `team03-pico-a` |

⚠️ **Two clients with the same ID do not produce an error.** The broker silently disconnects the earlier one. Your node will drop off at random, and the cause will be another team's code.

With hierarchical topics, subscribing to `team03/#` gives you everything your team publishes.

---

## 6 · Task B — Publishing from the Picos

The MQTT library is not built into the firmware. Connect to Wi-Fi first (Week 4), then in the Shell:

```python
import mip
mip.install("umqtt.simple")
```

**Publisher:**

```python
from umqtt.simple import MQTTClient
import time

CLIENT_ID = b"team03-pico-a"          # TODO: yours, and unique
BROKER    = "192.168.1.50"            # TODO: your Pi Zero's address
TOPIC     = b"team03/pico-a/temperature"

c = MQTTClient(CLIENT_ID, BROKER, keepalive=30)
c.connect()

while True:
    value = read_sensor()             # TODO: your sensor function
    c.publish(TOPIC, ("%.2f" % value).encode())
    time.sleep(2)
```

Note `keepalive=30`. You have just configured a failure detector — see §8.

### Pico B's second reading

Your kit has **one** temperature sensor, so Pico A uses it. Pico B uses the temperature sensor **built into the RP2350 chip**:

```python
from machine import ADC

chip = ADC(4)                          # the chip's internal temperature sensor

def read_sensor():
    volts = chip.read_u16() * 3.3 / 65535
    return 27 - (volts - 0.706) / 0.001721
```

The conversion comes from the chip datasheet and is approximate. More importantly, it measures **the chip itself**, which runs warmer than the room. So Pico B will report a different temperature from Pico A — **and that is exactly what experiment 3 needs.**

---

## 7 · Task C — Subscribing on the Pi Zero

Install with `sudo apt install python3-paho-mqtt`.

```python
import paho.mqtt.client as mqtt

def on_message(client, userdata, msg):
    print(msg.topic, msg.payload.decode())

try:
    c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)   # paho-mqtt 2.x
except AttributeError:
    c = mqtt.Client()                                   # paho-mqtt 1.x

c.on_message = on_message
c.connect("localhost", 1883, 60)
c.subscribe("team03/#")              # TODO: your team number
c.loop_forever()
```

The `try` handles both versions of the library — they changed how a client is created, and which one you have depends on your OS release.

---

## 8 · Notice what you just built

MQTT has two features that are, in this course's vocabulary, a **failure detector**:

| Feature | What it does |
|---|---|
| keepalive | If the broker hears nothing from a client for too long, it declares the client gone |
| last will | A message the client registers *in advance*, which the broker publishes on its behalf if it disappears |

Set a last will now, even a trivial one. Then ask yourself the question Week 8 is about: **how did the broker decide your node was dead — and when would it be wrong?**

---

## 9 · Task D — Three experiments

### 1 · Stop the broker

With everything running, on the Pi Zero:

```bash
sudo systemctl stop mosquitto
```

Record what each part does. Does the publishing Pico raise an error, block, or carry on silently? How long until it notices? What happens to readings taken while the broker is down — queued, or gone? When you restart the broker, does the Pico reconnect by itself, or does it need a reset?

Your broker is not a replica, but every message passes through it — so it carries the same weakness as the primary in this week's lecture. You are watching that weakness happen.

### 2 · Retained messages

Publish with the retain flag set. The broker keeps the value and gives it immediately to any future subscriber:

```python
c.publish(TOPIC, ("%.2f" % value).encode(), retain=True)
```

Now **unplug the publishing Pico** and subscribe fresh. You immediately receive a temperature — from a node that is no longer running.

The value is real, correctly delivered, and possibly hours old. **Is it true?**

### 3 · Make your replicas disagree

Both Picos are now publishing a temperature — Pico A from the sensor, Pico B from its chip. Publish them to sibling topics (`team03/pico-a/temperature`, `team03/pico-b/temperature`) and answer:

- How far apart are the two readings, typically?
- A subscriber receives both. What rule should it use to produce *one* number?
- If one Pico stops publishing, how long before a subscriber could reasonably notice?

You have built a two-replica system with no rule for resolving disagreement. The lecture argued that this situation is unavoidable once copies exist. Decide what your system does about it.

Notice too that Pico B is punctual and well-formed — and measuring the wrong thing. That is the "silent wrongness" from Week 2, appearing in a sensor rather than a model.

---

## 10 · Choosing a delivery guarantee

In Week 4 you measured a real loss rate on your Wi-Fi link. **Use it.**

| Data | Suggested QoS | Why |
|---|---|---|
| Periodic readings | 0 | Another arrives in two seconds; one loss costs almost nothing |
| A detection event | 1 | Losing it matters; a duplicate does not, if your handler is idempotent |

State your choice and your measured loss rate together in your report. A decision with a number behind it is worth far more than a correct default.

---

## 11 · Optional — I²C sensors

Do this only once the core works. I²C devices share one pair of wires — **SDA** (data) and **SCL** (clock) — plus power and ground, and are told apart by **address**, not pin. It is your first shared medium at the hardware level: the same idea as Wi-Fi, three orders of magnitude smaller.

**Scan before writing any driver code:**

```python
from machine import I2C, Pin

i2c = I2C(0, scl=Pin(5), sda=Pin(4), freq=400000)   # TODO: your pins
print([hex(a) for a in i2c.scan()])
```

An empty list means wiring or power is wrong — no driver will help. Each address you see is one device answering; match it against your module's documentation.

Two pitfalls: the bus needs **pull-up resistors** (most boards include them; several boards in parallel can overload the lines), and two devices with the **same address** cannot share a bus. Each chip needs its own driver — finding a working MicroPython driver for your specific part is the exercise.

---

## 12 · What to submit

Everything goes in `week05/` of your team repository — see the [Submission Guide](../SUBMISSION_GUIDE.md).

- **Code** — broker config, both publishers, the Pi Zero subscriber
- **Logs** — a capture covering the broker being stopped and restarted
- **Three experiment write-ups** — what you observed, not what you expected
- **Report §3** — is a retained value true, and what would you add to a message so a subscriber could judge for itself?
- **Model spec update** — your broker is a single point everything depends on; record what that means

If you did the optional part, include your I²C scan output and sensor code. If not, nothing is deducted.

---

## 13 · When it does not work

| Symptom | Most likely cause |
|---|---|
| Local test works, Pico cannot connect | No `listener` line — broker still only accepts local connections |
| Pico connects, then drops repeatedly | Duplicate client ID, probably another team's |
| Connected, but no messages arrive | Topic mismatch — a typo or missing team prefix |
| Pico B will not start | 5 V and GND not both connected, or wired to the wrong pins |
| Pico B's temperature looks too high | Expected — it measures the chip, not the room |
| `mqtt.Client()` raises an error | Library version — use the `try` form in §7 |
| `i2c.scan()` returns empty | Wiring, power, or SDA and SCL swapped |
| Publisher freezes after a while | Broker unreachable and the client is blocking — see experiment 1 |

---

## 14 · Before Checkpoint 1

**Checkpoint 1 is two weeks away.** It covers Weeks 2 to 6, so after next week's lab you will have everything it reviews, and Week 7 is the checkpoint itself. Keep committing as you go rather than saving it all for the end — your commit history is part of the mark. Check on GitHub, not on your laptop, that Weeks 2 to 5 are already pushed.
