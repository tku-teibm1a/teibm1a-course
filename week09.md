# Week 09 — Elections, and Split Brain

**Time:** about 2 hours 20 min, in two sittings · **Where:** anywhere with a table and a power outlet — see [LAB_SETUP.md](../LAB_SETUP.md) · **Covered by:** Checkpoint 2

Two things this week:

1. **Run an election.** Implement Bully across your three nodes, stop the leader, and watch a new one emerge. Measure how long it takes.
2. **Break it, then fix it.** Simulate a network split so that *two* leaders appear. Then switch on the majority rule, and show that the same split produces **no** leader instead — and a leader again once the split heals.

Both halves are small. What matters is that you cause the failure deliberately, and can explain why the fix works.

---

## 1 · Workload budget

| Task | When | Budget |
|---|---|---|
| Addresses, wiring the switches, first election | Sitting 1 | ~40 min |
| Experiment A — stop the leader (5 trials) | Sitting 1 | ~20 min |
| Experiment B — force split brain | Sitting 2 | ~20 min |
| Experiment C — the same split, with the majority rule | Sitting 2 | ~20 min |
| Write-up and model spec update | Sitting 2 | ~40 min |

B and C use the same partition. You set it up once and run it twice: before and after switching the rule on. **No reprogramming is needed between experiments** — two jumper wires do the switching.

Each task ends with a **Check before moving on**.

---

## 2 · ⚠️ Before you start

**Power:** as before — Pico A on your laptop's USB, Pico B from the Pi Zero's 5 V header. **Never plug Pico B's USB in while it is powered from the header.**

**"Killing the gateway" means stopping its program — not unplugging it.** Pico B gets its power from the Pi Zero. If you unplug the Pi Zero, Pico B dies too, and you are no longer testing what you think. Stop the gateway with **Ctrl-C** in its terminal.

**Network:** all three boards on one 2.4 GHz network — see [LAB_SETUP.md](../LAB_SETUP.md) §3–5.

---

## 3 · Decide this first: the election must not use the broker

Your MQTT broker runs on the Pi Zero. If the gateway dies, the broker dies with it — and your two Picos would have no way to hold the election that its death is supposed to trigger.

So election messages go over **UDP, directly between nodes**, not through the broker. Sensor data can keep using MQTT.

The general rule is worth writing in your model spec:

> **A recovery mechanism must not depend on the component whose failure it handles.**

---

## 4 · Identifiers, addresses, and what the leader does

| id | Node | |
|---|---|---|
| 1 | `pico-a` | |
| 2 | `pico-b` | |
| 3 | `gateway` | highest — so it leads while it is alive |

The leader's one visible job: it sends `LEADER` to the others once a second, instead of the ordinary `HB` hello that everyone else sends. Those hellos are also how each node knows who it can currently reach.

**Port:** `9100 + team number` (team 03 uses 9103). This is deliberately different from your Week 4 port, so an old listener cannot interfere.

**Addresses:** every node needs the other two's IP addresses.
- Each Pico prints `my address: …` when it starts.
- On the Pi Zero, run `hostname -I`.

Home networks can hand out new addresses on another day — check them at the start of every session.

---

## 5 · Task A — The program: one file, all three nodes

The same `election.py` runs on both Picos (MicroPython) and on the Pi Zero (Python 3). Only `NODE_ID` changes. On Pico B, save it as `main.py` — together with `config.py` — so it starts without Thonny.

Three messages carry the election:

| Message | Sent when | Meaning |
|---|---|---|
| `ELECTION` | A node suspects the leader is gone | "I am starting an election" — sent **only to higher ids** |
| `OK` | A higher node receives ELECTION | "I am alive. Stand down; I will take over" |
| `COORDINATOR` | A node wins | "I am the leader" — sent to everyone |

```python
# election.py — the same file runs on all three nodes
import socket, sys, time

NODE_ID = 1                       # TODO: 1 = pico-a, 2 = pico-b, 3 = gateway
TEAM    = 3                       # TODO: your team number
ADDR    = {1: "192.168.1.21",     # TODO: pico-a's address
           2: "192.168.1.22",     # TODO: pico-b's address
           3: "192.168.1.50"}     # TODO: the Pi Zero's address
PORT    = 9100 + TEAM

HB_MS      = 1000     # every node says hello this often
TIMEOUT_MS = 3000     # leader silent this long -> election (your Week 8 choice)
T_WAIT_MS  = 1000     # wait this long for an OK
T_COORD_MS = 3000     # after an OK, wait this long for a COORDINATOR
MAJORITY   = 2        # of 3 nodes

# ---- the only part that differs between a Pico and the Pi Zero ----
try:
    import machine
    ON_PICO = True
except ImportError:
    ON_PICO = False

if ON_PICO:
    import network
    from machine import Pin
    from config import SSID, PASSWORD            # see LAB_SETUP.md section 4
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)
    for _ in range(30):
        if wlan.isconnected():
            break
        time.sleep(0.5)
    print("my address:", wlan.ifconfig()[0])
    _iso = Pin(15, Pin.IN, Pin.PULL_UP)          # GP15 to GND: cut off from the other Pico
    _maj = Pin(14, Pin.IN, Pin.PULL_UP)          # GP14 to GND: majority rule ON
    def isolated():    return _iso.value() == 0
    def majority_on(): return _maj.value() == 0
    def now():         return time.ticks_ms()
    def since(t):      return time.ticks_diff(time.ticks_ms(), t)
else:
    def isolated():    return False
    def majority_on(): return "--majority" in sys.argv
    def now():         return int(time.monotonic() * 1000)
    def since(t):      return now() - t

# ---- the election ----
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", PORT))
sock.settimeout(0.05)
PEERS = [p for p in ADDR if p != NODE_ID]

state  = "follower"      # follower | electing | waiting | leader | no-leader
leader = None
heard  = {}              # peer id -> when we last heard anything from it
wait_start, wait_ms = 0, 0

def log(msg):
    line = "%d %s  [state=%s leader=%s]" % (now(), msg, state, leader)
    print(line)
    with open("election_log.txt", "a") as f:
        f.write(line + "\n")

def cut_off(peer):       # the simulated partition between the two Picos
    return isolated() and peer in (1, 2) and NODE_ID in (1, 2)

def send(peer, kind):
    if cut_off(peer):
        return
    try:
        sock.sendto(("%s %d" % (kind, NODE_ID)).encode(), (ADDR[peer], PORT))
    except OSError:
        pass

def reachable():         # me, plus every peer heard from recently
    return 1 + len([p for p in heard if since(heard[p]) < TIMEOUT_MS])

def start_election():
    global state, wait_start, wait_ms
    state, wait_start, wait_ms = "electing", now(), T_WAIT_MS
    log("ELECTION started")
    higher = [p for p in PEERS if p > NODE_ID]
    if not higher:
        become_leader()
        return
    for p in higher:
        send(p, "ELECTION")

def become_leader():
    global state, leader
    if majority_on() and reachable() < MAJORITY:
        if state != "no-leader":
            state = "no-leader"
            log("NO MAJORITY - refusing to lead (reachable=%d)" % reachable())
        return
    old = leader
    state, leader = "leader", NODE_ID
    if old is not None and old != NODE_ID and old in heard:
        log("I AM LEADER - %d ms since last heard from old leader %d" % (since(heard[old]), old))
    else:
        log("I AM LEADER")
    for p in PEERS:
        send(p, "COORDINATOR")

def on_message(sender, kind):
    global state, leader, wait_start, wait_ms
    if kind == "ELECTION":
        log("ELECTION from %d - replying OK" % sender)
        send(sender, "OK")
        if state not in ("electing", "waiting"):
            start_election()
    elif kind == "OK" and state == "electing":
        state, wait_start, wait_ms = "waiting", now(), T_COORD_MS
        log("OK from %d - standing down" % sender)
    elif kind == "COORDINATOR":
        if sender < NODE_ID:
            log("COORDINATOR from lower id %d - challenging" % sender)
            start_election()
        else:
            old = leader
            state, leader = "follower", sender
            if old is not None and old != sender and old in heard:
                log("COORDINATOR from %d accepted - %d ms since last heard from old leader %d"
                    % (sender, since(heard[old]), old))
            else:
                log("COORDINATOR from %d accepted" % sender)
    elif kind == "LEADER" and state == "leader":
        log("ANOTHER LEADER: %d - starting election" % sender)
        start_election()

log("START  majority rule %s" % ("ON" if majority_on() else "off"))
start_election()
last_hb = now()

while True:
    try:
        data, _ = sock.recvfrom(64)
        kind, sender = data.decode().split()
        sender = int(sender)
        if sender in PEERS and not cut_off(sender):
            heard[sender] = now()
            on_message(sender, kind)
    except (OSError, ValueError):
        pass

    if since(last_hb) >= HB_MS:                  # hello, and the leader's visible job
        last_hb = now()
        for p in PEERS:
            send(p, "LEADER" if state == "leader" else "HB")
        if state == "leader" and majority_on() and reachable() < MAJORITY:
            state = "no-leader"
            log("LOST MAJORITY - stepping down")

    if state == "electing" and since(wait_start) > wait_ms:
        become_leader()                          # nobody higher answered in time
    elif state == "waiting" and since(wait_start) > wait_ms:
        log("no COORDINATOR arrived - restarting election")
        start_election()
    elif (state == "follower" and leader is not None
          and since(heard.get(leader, now())) > TIMEOUT_MS):
        log("leader %d silent for %d ms" % (leader, since(heard[leader])))
        start_election()
    elif state == "no-leader" and reachable() >= MAJORITY:
        start_election()
```

How to read it:

- **`start_election()`** sends ELECTION to every higher id. If there are none, the node wins at once.
- **An `OK` does not tell you who will lead** — only that it will not be you. You wait for a COORDINATOR; if none comes, you try again.
- **A follower whose leader is silent for `TIMEOUT_MS` starts an election.** This is your Week 8 detector, inside the election. Set `TIMEOUT_MS` to the value you chose last week.
- **`become_leader()` is where the majority rule lives** — five lines that only act when the GP14 switch is on (§8).
- **Every event is logged**, with the node's own clock, to the screen and to `election_log.txt`. That log is your evidence, and you cannot reconstruct it afterwards.

### Wiring the two switches

The Picos read two jumper wires **while running**, so you can change the experiment without touching the code:

| Wire | Pico A | Pico B | Effect when connected |
|---|---|---|---|
| **Partition** | GP15 (pin 20) → GND (pin 18) | — | The two Picos ignore each other — a simulated network split |
| **Majority rule** | GP14 (pin 19) → GND (pin 23) | GP14 (pin 19) → GND (pin 18) | A node refuses to lead without 2 of 3 nodes |

One partition wire is enough: Pico A then neither sends to nor listens to Pico B, so Pico B hears nothing from Pico A either. This is the same idea as the isolation wire in Week 6.

On the Pi Zero, the majority rule is a command-line flag: `python3 election.py --majority`.

**Check before moving on** — no wires connected:

1. Start the gateway first: `python3 election.py`. It logs `I AM LEADER`.
2. Start Pico A in Thonny, and power Pico B. Pico A's Shell shows `COORDINATOR from 3 accepted`.
3. Leave it for one minute. **Nothing else** should be logged — no repeated elections.

---

## 6 · Task B — Experiment A: stop the leader

1. Everything running, no wires. The gateway (3) is leader.
2. Press **Ctrl-C** on the gateway.
3. Watch Pico A's Shell. Expect `leader 3 silent…`, then `COORDINATOR from 2 accepted — … ms since last heard from old leader 3`. Pico B — the highest survivor — has won.
4. Restart the gateway. Being the highest id, it takes the leadership back — that is Bully living up to its name.
5. Repeat — **five trials**.

**The measurement** is the number in step 3: the time from the **last message heard from the old leader** to the **new leader's COORDINATOR**, both measured on **Pico A's own clock**. As in Week 8, both ends come from one clock, so the number is honest.

**What to expect, and explain.** Roughly `TIMEOUT_MS + T_WAIT_MS` — first the silence must last long enough to be suspected, then the winner waits to see whether anyone higher objects. Check that against your numbers, and explain any gap.

**This is where your Week 8 choice starts to cost something real.** A short timeout means fast failover — and occasional pointless elections. A long one means the system runs **leaderless** for longer. Say in your report whether you would still choose the same timeout.

**Check before moving on:** five trials, every one ending with Pico B as leader, and each time between about 3.5 and 5 s with the default settings.

---

## 7 · Task C — Experiment B: force split brain

1. Everything running, **no majority wires**. The gateway leads.
2. Connect the **partition wire** on Pico A.
3. Press **Ctrl-C** on the gateway.
4. Each Pico now sees only itself. Each times out — and each declares itself leader. **Two leaders.**
5. Pico A's Shell shows `I AM LEADER`. Wait 10 seconds, so both have led for a while.
6. **Remove the partition wire.** Pico A's Shell shows `ANOTHER LEADER: 2` — proof, on one clock, that both were leading at once — and then accepts Pico B.

**Read Pico B's log afterwards** for its side of the story: remove Pico B's 5 V wire, connect it by USB, and open `election_log.txt` in Thonny. You should see its own `I AM LEADER` during the same period.

Label this clearly as a **simulated partition** in your report. It is a faithful model of the effect on messages, not of a real radio failure.

**Check before moving on:** Pico A's log shows `I AM LEADER` while the wire was in, and `ANOTHER LEADER: 2` after you removed it.

---

## 8 · Task D — Experiment C: the same split, with the majority rule

Here is the fix — already in your code, inside `become_leader()`:

```python
    if majority_on() and reachable() < MAJORITY:
        if state != "no-leader":
            state = "no-leader"        # stop; do not lead
            log("NO MAJORITY - refusing to lead (reachable=%d)" % reachable())
        return
```

Five lines. A node that cannot see enough of the cluster **refuses to act** — which is the whole of this week's lecture, written as an early return.

1. Restart the gateway with `python3 election.py --majority`, and connect the **majority wire on both Picos**. The gateway leads.
2. Connect the **partition wire**, then press **Ctrl-C** on the gateway — exactly as in B.
3. Expect **no leader at all.** Each Pico can reach only itself: one node out of three. Each logs `NO MAJORITY - refusing to lead`. The system **stops rather than diverging.**
4. Wait 10 seconds. Then **remove the partition wire.** The two Picos can hear each other again — two out of three — and Pico B becomes leader **without the gateway**.

Record both moments. The change from "stopped" to "led" is the clearest demonstration in the whole course of what a majority actually does.

**Check before moving on:** during the partition, neither Pico logs `I AM LEADER`; after it heals, Pico A shows `COORDINATOR from 2 accepted`.

---

## 9 · What to record

| Experiment | Report |
|---|---|
| A | Five election times, and how they relate to your Week 8 detection time and `T_WAIT_MS` |
| B | Log extracts from **both** Picos showing each announcing itself leader during the same period |
| C | The same partition producing **no** leader, then a leader once the Picos reconnect |

For B, the strongest answer names a concrete **consequence**: what would your system have produced, or corrupted, while two leaders were both active? Think about the opening problem in the lecture — two aggregators, two different models, both under the same name.

---

## 10 · What to submit

Everything goes in `week09/` — see the [Submission Guide](../SUBMISSION_GUIDE.md) and the [Report Guide](../REPORT_GUIDE.md).

- **Code** — `election.py`, with the majority rule (it can be switched on and off, so one file covers both)
- **Logs** — all three experiments, from every node involved, with their timestamps
- **Election timings** — five trials, compared with your Week 8 detection time
- **Report §3** — what two simultaneous leaders would have done to your system
- **Model spec update** — your election rule, your majority size, what your system does when no majority is reachable, and the rule from §3: recovery must not depend on the component that failed

---

## 11 · When it does not work

| Symptom | Most likely cause |
|---|---|
| Elections repeat endlessly | `TIMEOUT_MS` shorter than `HB_MS`, or an address in `ADDR` is wrong so a peer is never heard |
| The lowest id wins | ELECTION being sent to all peers instead of only higher ids |
| Nothing happens when the gateway stops | Pico A and Pico B are not hearing each other — check the addresses and the port |
| Pico B stopped too when you "killed" the gateway | You unplugged the Pi Zero — stop the program with Ctrl-C instead (§2) |
| No leader even when all three are up, majority on | An address is wrong, so `reachable()` counts too few nodes |
| Split brain never appears in experiment B | The partition wire is on the wrong pin, or a majority wire is still connected |
| `OSError` when the program starts on a Pico | Wi-Fi did not connect — check `config.py` |

---

## 12 · Next week

Your nodes can now agree on **who** leads. Next week they have to agree on **when** things happened — closely enough that readings from two sensors can be fused into one input. You will synchronise the clocks you measured drifting in Week 3, and find out how much of the disagreement can actually be removed.

**Checkpoint 2** covers Weeks 8–10 and closes at **23:59 on Tuesday 1 December**.
