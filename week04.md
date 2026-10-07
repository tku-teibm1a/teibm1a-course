# Week 04 — The First Connection

**Before class:** ~45 min · **In class:** 1 hour · **Outside class:** ~1 hour · **Covered by:** Checkpoint 1

Your nodes finally talk. You will run the same exchange over a **wired serial link** and over **Wi-Fi**, and measure how differently the two behave.

**Bringing the Pi Zero up is homework, not part of the session.** It is mechanical, and doing it in class costs you the whole hour. Arrive with a Pi Zero you can log into.

---

## 1 · What you will have at the end

- A Pi Zero running Linux that you can reach without a monitor or keyboard
- A Pico sending numbered messages to the Pi Zero over a wire
- The same Pico sending numbered messages over Wi-Fi
- Your own measurements of order, loss and delay on both channels

---

## 2 · Workload budget

| Task | When | Budget |
|---|---|---|
| Pi Zero bring-up (§4) | Before class | ~45 min |
| Wired serial link (§6) | In class | ~30 min |
| Wi-Fi link (§7) | In class | ~30 min |
| Three experiments + write-up (§9) | After class | ~60 min |

If the core is not done, **stop and say so in your report** — where you got to, what you tried, where your understanding ran out. That is worth marks. Working silently until 2 a.m. is not.

---

## 3 · ⚠️ Read before wiring

- **The USB-to-TTL cable has a red 5 V wire. Leave it disconnected.** The Pi Zero has its own power supply; two sources feeding one power rail is how boards die.
- **Transmit goes to receive.** On every serial link this week, TX on one side connects to RX on the other. The two wires cross over.
- **Always connect GND.** It is the most commonly forgotten wire, and without it nothing works reliably.
- The Pico and the Pi Zero both use **3.3 V logic**, so they connect directly with no level shifting.

Cable colours vary between manufacturers. **Check yours** before trusting the table below.

---

## 4 · Before class — bring up the Pi Zero

### 4a · Prepare the SD card

1. Install **Raspberry Pi Imager** on your laptop.
2. Choose **Raspberry Pi OS Lite (64-bit)**. No desktop — the Zero has little to spare, and you will only use a terminal.
3. Before writing, open the Imager's **settings** and preconfigure:
   - **Hostname:** `teamNN-gw` (your team number — eight machines all called `raspberrypi` on one network is a problem you can avoid for free)
   - **Username and password**
   - **Wi-Fi:** your team's router, name and password
   - **Enable SSH**
4. Write the card, insert it, power the Zero.

You may have already done steps 1–4 during last week's lab. If so, go straight to 4b.

### 4b · Why you were given a USB-to-TTL cable

If the Wi-Fi details were wrong, SSH will never reach the board — and with no screen, you cannot find out why.

The serial console solves this. It is a direct wire into the boot process: you see kernel messages and get a login prompt **with no network involved at all**. Learn it now, while nothing is broken. It is the tool you will want at the exact moment nothing else works.

| USB-to-TTL wire | Pi Zero pin |
|---|---|
| GND (black) | any GND pin |
| TXD (green) | GPIO 15 / RXD |
| RXD (white) | GPIO 14 / TXD |
| 5 V (red) | **leave disconnected** |

Add `enable_uart=1` to `config.txt` on the SD card's boot partition. Then connect at **115200 baud**:

```bash
# Linux or macOS
screen /dev/ttyUSB0 115200

# Windows: PuTTY → Serial → 115200
```

You should see boot messages, then a login prompt.

### 4c · Get SSH working

Log in over the console, confirm the Pi Zero is on Wi-Fi (`hostname -I` shows its address), then from your laptop:

```bash
ssh <username>@teamNN-gw.local
```

**Do not continue until SSH works.** The next step disables the serial login, and if SSH is not working you will have locked yourself out. *(If that happens: put the SD card back in your laptop and re-enable it in `config.txt`.)*

---

## 5 · Freeing the serial port for the Pico

The Pico will use the **same serial port** the console just used. They cannot share it. Two things will stop your link working until you change them:

**1 · The login console owns the port.** Linux runs a login prompt on it. It will swallow your Pico's bytes and reply with garbage. Fix: run `sudo raspi-config` → *Interface Options* → *Serial Port* → **"login shell over serial?" No**, **"serial port hardware enabled?" Yes**. Reboot.

**2 · Bluetooth has the better serial port.** On this board, the more capable serial port is assigned to Bluetooth, leaving a simpler one on the GPIO pins whose timing depends on the processor clock. If your link is unreliable, add `dtoverlay=disable-bt` to `config.txt` and reboot. *(Check current Raspberry Pi documentation — this setting has changed between OS releases.)*

From now on you reach the Pi Zero by **SSH only**. Disconnect the USB-to-TTL cable.

---

## 6 · Task A — The wired link

| Pico | Pi Zero |
|---|---|
| GP0 (TX) | GPIO 15 (RXD) |
| GP1 (RX) | GPIO 14 (TXD) |
| GND | GND |

**Sender — on the Pico:**

```python
from machine import UART, Pin
import time

uart = UART(0, baudrate=115200, tx=Pin(0), rx=Pin(1))

seq = 0
while True:
    uart.write("%d\n" % seq)
    seq += 1
    time.sleep(0.1)
```

Every message carries a **sequence number**. This is not decoration — it is the only way you will detect loss or reordering later, and you cannot add it afterwards.

**Receiver — on the Pi Zero:** install with `sudo apt install python3-serial`, then:

```python
import serial

ser = serial.Serial("/dev/serial0", 115200, timeout=1)

while True:
    line = ser.readline().decode(errors="ignore").strip()
    if line:
        print(line)
```

`errors="ignore"` matters: during startup you will receive partial bytes, and a decode error should not end your program.

---

## 7 · Task B — The wireless link

**The Pico joins the network:**

```python
import network, time

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, PASSWORD)      # TODO: your team's router

for _ in range(30):
    if wlan.isconnected():
        break
    time.sleep(0.5)

print("ip:", wlan.ifconfig()[0])
```

Note the **bounded** wait. A loop that waits forever gives you a node that looks dead but is still trying — the exact ambiguity from Week 2, created by your own code.

**The Pico is 2.4 GHz only.** If your router broadcasts a 5 GHz-only network, the Pico will never connect and will not tell you why.

### Why UDP, not TCP

TCP retransmits lost packets automatically, so your log would show no loss at all — because **TCP turns loss into delay, and hides it from you**. Excellent engineering; useless for learning. UDP gives you the unimproved "fair-loss" link from the lecture, so you can see what the network actually does.

**Sender — on the Pico:**

```python
import socket, time

GATEWAY = ("192.168.1.50", 9003)   # TODO: your Pi Zero's address, your port
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

seq = 0
while True:
    msg = "%d %d" % (seq, time.ticks_ms())
    s.sendto(msg.encode(), GATEWAY)
    seq += 1
    time.sleep(0.1)
```

The local timestamp travels with each message. You know from Week 3 you cannot compare it with the Pi Zero's clock directly — but the **intervals** between messages are still meaningful.

**Receiver — on the Pi Zero:**

```python
import socket, time

srv = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
srv.bind(("0.0.0.0", 9003))        # TODO: your port

with open("wifi_log.txt", "w") as f:
    while True:
        data, addr = srv.recvfrom(1024)
        f.write("%s %.3f\n" % (data.decode().strip(), time.time()))
        f.flush()
```

Log to a **file**, not only to the screen. A terminal's scrollback is not data.

---

## 8 · Name things so eight teams can coexist

You share two routers with seven other teams.

| Thing | Rule |
|---|---|
| Hostname | `teamNN-gw` |
| UDP port | **9000 + your team number** — team 03 uses 9003 |
| Anything with an ID | Prefix it with your team number. This matters even more next week. |

Two teams using the same port on the same network will interfere in ways that look like network faults, and cost you an afternoon.

---

## 9 · Task C — Three experiments

Measure. Do not assume.

**1 · Is it FIFO?** Send several hundred numbered messages as fast as you can. Check whether any sequence number arrives after a larger one. **Both channels.**

**2 · What is lost?** Run for ten minutes, ideally while other teams are active. Count the gaps in your sequence numbers. **Both channels.**

**3 · What does the sender see when the receiver dies?** Unplug the Pi Zero mid-exchange. What does the Pico do — and does it notice at all?

If you have time, run experiment 1 over TCP as well. One channel shows you **gaps**; the other shows you **stalls**.

---

## 10 · What you should find

| | Serial | Wi-Fi (UDP) |
|---|---|---|
| Order | FIFO — it is a physical wire | Usually in order, not guaranteed |
| Loss | Near zero | Non-zero, worse when others are active |
| Delay | Stable | Variable, with occasional large spikes |
| Receiver dies | **Sender notices nothing** | **Sender notices nothing** |

Think hardest about the last row. On **both** channels the sender carries on sending into nothing. No channel tells it that nobody is listening. That is exactly why Week 8 exists.

---

## 11 · What to submit

Everything goes in `week04/` of your team repository — see the [Submission Guide](../SUBMISSION_GUIDE.md).

- **Code** — both sender variants and both receivers
- **Logs** — raw capture from each channel, in `evidence/`
- **A comparison table** — your own numbers for order, loss and delay on both channels
- **Report §3** — of the two channels, which is closer to the "perfect link" from the lecture, and what exactly is still missing?
- **Model spec update** — you now have real evidence about your channel. Revise that section properly.

A hint for §3: neither channel is a perfect link. Think about what each one can and cannot tell you about the **program** at the other end.

---

## 12 · When it does not work

| Symptom | Most likely cause |
|---|---|
| Console shows nothing | TX/RX not crossed, or `enable_uart=1` missing |
| Console shows unreadable characters | Wrong baud rate |
| Locked out after disabling serial login | SSH was not working first — re-enable in `config.txt` from your laptop |
| Serial link receives garbage | The login console is still running on the port |
| Serial works briefly, then corrupts | Timing instability — try `dtoverlay=disable-bt` |
| Pico never joins Wi-Fi | Wrong password, or a 5 GHz-only network |
| UDP sends fine, nothing arrives | Wrong IP, wrong port, or another team using your port |

Change one thing at a time, and predict the result before you test it.

---

## 13 · Before Week 5

Next week an MQTT broker runs on your Pi Zero, and both Picos publish to it. Nothing to prepare in advance — but **keep your Week 4 loss measurements**. You will use them to choose a delivery guarantee.
