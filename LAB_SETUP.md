# Doing the Labs Outside Class

**Distributed Systems & Edge Architectures (TEIBM1A)**

The classroom has little table space and very few power outlets, so the hands-on part of each week is done **where you work best** — at home, in a lab, in a study room. Class time is for the theory, and for your questions.

Each week's assignment guide is written so you can follow it on your own, step by step. This page covers the setup that is the same every week: power, network, and a routine for starting a session.

---

## 1 · What you need

| Item | Why |
|---|---|
| A table with room for three boards and a laptop | You will be moving wires while things are powered |
| **One power outlet** for the Pi Zero's power supply | A power strip makes life easier |
| Your laptop, charged or plugged in | It powers Pico A over USB, and it is how you reach everything else |
| A Wi-Fi network that meets the rules in §2 | All three boards and your laptop must reach each other |
| Your kit: both Picos, Pi Zero, cables, breadboard, sensors, jumper wires | Check you have everything before you leave class |

From Week 5 on, all three boards run at once. Pico A is powered from your laptop's USB, and **Pico B from the Pi Zero's 5 V header** — see the Week 5 guide, §3. That way you only need one outlet.

> ⚠️ **Never plug Pico B's USB cable in while it is powered from the header.** To reprogram Pico B, remove the 5 V wire first.

---

## 2 · Choosing a network

Your boards need a network that is:

- **2.4 GHz.** The Pico cannot see 5 GHz networks at all.
- **Password only** (WPA2-Personal). No login page in a browser, and no username.
- **Not isolating devices.** Your devices must be able to reach *each other*, not only the internet.

| Network | Works? |
|---|---|
| Home router | Usually yes — use the 2.4 GHz network if it has two |
| **Phone hotspot** | **Yes — the most reliable choice.** See below |
| Dormitory or campus Wi-Fi with a login page | No — the Pico cannot log in through a web page |
| Guest networks, café Wi-Fi | Often no — many block device-to-device traffic |

**Phone hotspot settings:**

- **iPhone:** Settings → Personal Hotspot → turn on **Maximize Compatibility** (this switches it to 2.4 GHz).
- **Android:** Hotspot settings → set the band to **2.4 GHz**.

Your laptop must join the **same** network as the boards.

---

## 3 · Moving the Pi Zero to a new network

The Pi Zero only joins networks it already knows. In Week 4 you gave it one network in Raspberry Pi Imager. To use a different one, choose **one** of these:

### Option A — Add the new network while you can still reach the Pi Zero *(recommended)*

Do this while you are still on a network the Pi Zero knows, for example before leaving class. SSH in, then:

```bash
sudo nmcli connection add type wifi con-name home ifname wlan0 \
     ssid "YOUR_NETWORK_NAME" \
     wifi-sec.key-mgmt wpa-psk wifi-sec.psk "YOUR_PASSWORD"
nmcli connection show
```

The second command should list `home` next to your existing network. The Pi Zero now remembers both, and joins whichever is in range when it starts. Repeat with another `con-name` for each extra network.

*(If `nmcli` is not found, your SD card has an older Raspberry Pi OS. Use option B or C.)*

### Option B — Make a hotspot that looks like the old network

Set your phone hotspot's **name and password to exactly what you entered in Raspberry Pi Imager** in Week 4. The Pi Zero joins it with no changes at all. Useful if you are already at home and cannot reach the board.

### Option C — Re-flash the SD card *(last resort)*

Write a fresh image with Raspberry Pi Imager, entering the new network, exactly as in Week 4 §4. You then need to redo the installs from later weeks (for example, the MQTT broker from Week 5). Your code is safe in GitHub.

---

## 4 · The Picos on a new network

Two things change on a new network:

1. **`SSID` and `PASSWORD`** in your Pico code.
2. **The Pi Zero's address** (`BROKER` in your code). A different network gives it a different address, and even the same network may give it a new one on another day.

Tip: keep these in one small file, `config.py`, and import it:

```python
# config.py — do NOT commit this file
SSID     = "YOUR_NETWORK_NAME"
PASSWORD = "YOUR_PASSWORD"
BROKER   = "192.168.1.50"
```

```python
from config import SSID, PASSWORD, BROKER
```

Add `config.py` to your repository's `.gitignore`, and commit a `config_example.py` with the passwords blanked out instead. Your Wi-Fi password should not be in Git, even in a private repository.

---

## 5 · Starting a session — the same routine every time

Most "it stopped working" problems at home are one of these steps, skipped.

1. **Power the Pi Zero.** Wait about a minute for it to boot and join Wi-Fi.
2. **Log in from your laptop:**
   ```bash
   ssh <username>@teamNN-gw.local
   ```
   If `.local` does not resolve, find the Pi Zero's address in your phone hotspot's or router's list of connected devices, and use that instead.
3. **Get its address:** `hostname -I`. If it changed since last time, update `BROKER` in `config.py` on both Picos.
4. **From Week 5 on, check the broker is running:**
   ```bash
   sudo systemctl status mosquitto
   ```
5. **Connect Pico A** to your laptop, then **Pico B** from the header. Start each one and watch for its first message.
6. **Test one layer at a time.** If something fails, find out whether it is the network, the broker, or your code before changing anything — as in Week 5 §4.

When you finish: **commit and push before you unplug anything.**

---

## 6 · Following the weekly guide on your own

- **Read the whole guide once before you touch the hardware.** Each one starts with a ⚠️ section; read it first.
- **Do the tasks in order.** Each task assumes the one before it works.
- **Use the checks.** Where a guide says *Check before moving on*, do not move on until you see that result. Problems found early are small.
- **Budget the time in the guide's workload table.** If a task takes more than twice its budget, stop, commit what you have, and write down where you got to. That is the correct thing to do, and it is worth marks — see the [Report Guide](REPORT_GUIDE.md) §6.
- **Record conditions as you go.** Which network, how far from the router, how long you measured. You will need them for §2 of your report, and you will not remember them later.
- **Take your wiring photo before you take the circuit apart.**

---

## 7 · Working as a team of two

You have one kit between you, so usually one of you holds the hardware. That is fine, but:

- **Both members operate the hardware at least once each week.** Either of you must be able to explain and run any part of the system — see the [Submission Guide](SUBMISSION_GUIDE.md) §9.
- **Both members commit, from their own GitHub accounts.** If you work together on one laptop, use co-author trailers so the record is correct.
- **Hand the kit over in a working state,** with the latest code pushed.

---

## 8 · Getting help

Bring your questions to class. **You do not need to bring the hardware.** Bring what lets someone help you:

- the exact error message, copied — not a description of it
- the log file or the last 20 lines of output
- the commit with the code that fails

A question with a log attached can usually be answered in two minutes. "It doesn't work" cannot.
