# Writing Your Weekly Report

**Distributed Systems & Edge Architectures (TEIBM1A)**

Your code and data show *what* your system did. Your report is where you show you understand *why* — and what you learned when things went wrong. At each checkpoint, your reports are the first thing that is read.

This guide explains what goes in each section, and what usually goes wrong.

---

## 1 · The short version

- **One report per week**, in that week's folder: `weekNN/report.md`. Each week is written and read **on its own**.
- **Six sections, in the same order** every week.
- **§2 Evidence** and **§3 Question of the week** are the heart of the report. Spend most of your time there.
- Every claim needs a number from **your own hardware**, or a pointer to a file that contains one.
- **§5 is for what went wrong.** Hardware fails in many small ways. Say what happened, how you found it, and what you did.
- **Use AI tools — and say how** in §6.
- About **one to two pages** per week is right. Longer is not better.

### Is the checkpoint report one big combined report?

**No.** At a checkpoint you do not write anything new. Your PDF is simply your weekly reports, one after another, plus `model-spec.md` — see [CHECKPOINT_SUBMISSION.md](CHECKPOINT_SUBMISSION.md). Each week is read separately.

The one document that pulls the weeks together is **`model-spec.md`**. That is where your understanding of the whole system grows from week to week.

---

## 2 · The template

Copy this into `weekNN/report.md` and fill it in. Keep the six sections, in this order.

```markdown
# Week N — <topic from the assignment guide>

## 1. What we built
<Two or three sentences.>

## 2. Evidence
<Table of your key numbers. Short log extract if useful.>
<Point to the full files: evidence/xxx.txt>

## 3. Question of the week
<Restate the question in one line, then answer it.>

## 4. Model spec changes
<What changed in model-spec.md, and why. One or two lines.>

## 5. Problems and reflection
<What went wrong, how you found the cause, what you did about it,
and what it taught you.>

## 6. Who did what, and how we used AI
- <Name> (<GitHub username>): ...
- <Name> (<GitHub username>): ...
- AI: <tool> — <what for> — <how you checked it>
```

---

## 3 · Section by section

### §1 · What we built — *short*

Two or three sentences. What exists now that did not exist last week?

> **Weak:** This week we learned about MQTT and did the lab.
>
> **Good:** An MQTT broker now runs on our Pi Zero. Both Picos publish temperature every 5 s, and a subscriber on the Pi Zero logs both streams to one file.

Do not explain the theory here, and do not describe problems here — they belong in §5.

---

### §2 · Evidence — *show that it worked*

Show the numbers that prove the week's task was done. A good §2 has three parts:

1. **A small table** of your key results.
2. **The conditions** that make the numbers meaningful — how long you measured, how many samples, which board, what the setup was.
3. **A pointer** to the full raw file in `evidence/`.

Here is the idea, using an example that is not from this course — timing how long a small Python script takes to start on a laptop:

> **Weak:**
> The script starts in about 40 ms. It is fast.
>
> **Good:**
>
> | Runs | Min | Median | Max | Over 100 ms |
> |---|---|---|---|---|
> | 200 | 31 ms | 38 ms | 212 ms | 3 |
>
> 200 runs, one every 2 s, on one laptop running only a terminal. The 212 ms value is the very first run; the other two slow runs happened while a browser update was downloading (`evidence/startup_200.txt`, lines 1, 87 and 88).

The second version lets the reader check you. It states the conditions, gives the spread and not only the average, and points to the file. The odd values are **kept and explained** — not deleted.

**Rules for §2:**

- **Your own data only.** Numbers from the slides, from another team, from the internet or from an AI tool are not evidence.
- **Keep the mess.** Real measurements have jitter and odd values. Perfectly round numbers look invented.
- **Always state the time base.** "Drift was 40 ms" means nothing. "40 ms over 25 minutes" is a measurement.
- **Point to files, do not paste them.** A 10-line extract is fine. A 500-line log belongs in `evidence/`.
- **Every experiment the guide asks for appears here,** including any that failed. Report what you *observed*, not what you expected.
- **Photos:** one clear photo of the wiring is enough, when the guide asks for it.

---

### §3 · Question of the week — *show that you understand*

Each guide's *What to submit* section sets one question (sometimes in parts). Find it there; the table in section 5 of this page lists them.

A strong answer has four parts, usually in this order:

1. **The answer**, in the first sentence. Yes, no, which one, how much.
2. **The evidence** from *your* §2 that supports it. Quote the number.
3. **The reasoning** — connect the evidence to the lecture's definition. Use the term correctly: *bounded*, *happened-before*, *perfect link*, *concurrent*.
4. **The limit** — what your data cannot tell you, or when your answer would stop being true.

Same example. Suppose the question was: *can this script be used in a loop that must run every 100 ms?*

> **Weak:**
> Yes, because it is very fast.
>
> **Strong:**
> **Not reliably.** 197 of our 200 runs started in under 100 ms, but 3 did not, and the worst took 212 ms (§2). A deadline is only useful if it is met *every* time, and ours was missed 3 times in 200 — so the 100 ms limit is not a bound we can rely on. The slow runs had causes outside the script (a cold first start, a download in the background), which means a cleaner measurement would not remove the problem; it would only hide it. We measured one idle laptop, so a busier machine would probably miss the deadline more often, not less.

**Rules for §3:**

- **Answer the question that was asked.** Restate it in one line so you do not drift.
- **Use your own numbers.** An answer that would be the same for every team is not an answer about your system.
- **Use the definition, not intuition.** When the question says "justify using the definition", the definition must appear in your answer.
- **Your own words.** Copying sentences from the slides — or from an AI tool — is not reasoning.
- **"It depends" is fine — if you say on what.** Say which condition changes the answer.
- About **150–400 words**. If it is much longer, you are probably not answering directly.

---

### §4 · Model spec changes — *one or two lines*

Say what you changed in `model-spec.md` this week, and the evidence that made you change it.

> **Good:** Timing section: changed "asynchronous" to "partially synchronous, loop bound ~12 ms under normal load", based on our Week 2 measurements. Removed the claim that a crashed Pico can be detected immediately.

Writing "no change" should be rare — almost every week gives you new evidence about your system. Make the change itself **in `model-spec.md`, as a commit that week**: its history is read at the checkpoint.

---

### §5 · Problems and reflection — *what went wrong, and what you learned*

Hardware goes wrong in many small ways: a loose jumper, a board that will not join Wi-Fi, a sensor that reads nonsense, a script that runs on the laptop but not on the Pico. This is normal. It is also where much of the real learning happens, so write it down.

For each real problem (one to three is typical), answer four things:

1. **Symptom** — what you saw. Be exact: the error message, the wrong number, the LED that did not blink.
2. **Cause** — what it turned out to be, and **how you found out**. The *how* is the most valuable part.
3. **Fix** — what you changed. If you never fixed it, say what you tried and where you got stuck.
4. **Lesson** — what you would do differently, or what it taught you about the system.

> **Weak:**
> We had some Wi-Fi problems but fixed them.
>
> **Good:**
> **Pico B could not join Wi-Fi.** Its script stopped at the first line with `ImportError: no module named 'network'`, while Pico A, running the same file, connected. Since the code was identical, we swapped the two USB cables and ports — the error followed the board, not the cable or the laptop. The start-up banner Thonny prints showed the cause: Pico B was running the firmware for the plain Pico 2, which has no Wi-Fi support. Flashing the Pico 2 W firmware fixed it.
> *Lesson:* "same code, different result" means look at the board first. We now read the firmware banner before debugging anything else.

Notice how often the best lesson is about the **method of finding the cause** — isolating one layer, swapping parts, comparing a working board with a broken one. That is exactly the skill distributed systems need, since in a real system you can never see everything at once.

**When a problem connects to the lecture, say so.** A board that looked dead but was still trying to connect *is* the Week 2 question "crashed, or just slow?" — met in real life.

If nothing went wrong, write one line saying so. That is rare.

---

### §6 · Who did what, and how we used AI

**People** — one line each, specific and honest. It is checked against your commits.

> - Wang Xiao-Ming (`xmwang`): wiring, Pico A firmware, ran experiment B
> - Chen Mei-Ling (`mlchen`): Pi Zero gateway script, analysis table, §3 draft

Both members should be able to explain the whole week. This line records how you split the work, not who understands it.

**AI** — see section 4 below. One or two lines per tool is enough.

---

## 4 · Using AI tools

**You are encouraged to use AI tools** — for debugging, for explaining a concept you did not follow, for understanding an error, for help with code, and for improving your English. *How* you use them matters, and it is part of what is read.

### Good use

- **Debugging with the real details.** Paste the exact error message, the code, and what you expected. Name the hardware: *Raspberry Pi Pico 2 W, MicroPython*, or *Pi Zero 2 W, Raspberry Pi OS*. A vague question gets a vague answer.
- **Asking it to explain, not only to fix.** "Why does this fix work?" teaches you something. A fix you do not understand will break again next week.
- **Checking its answer.** Run the code. Compare with the official documentation. Test on your own board.
- **Polishing your English** — after you have written the content yourself.

### Be careful

- **AI tools are often wrong about hardware.** Pin numbers, firmware versions and library functions are frequently invented or out of date — especially for a new board like the Pico 2 W. Always check against the documentation or the guide.
- **AI cannot produce your evidence.** Every number in §2 must come from your own boards. Invented or "example" data is not evidence.
- **AI cannot answer §3 for you.** It does not know your data. It can explain a concept; the answer must use your numbers and your words.
- **You must be able to explain every line you submit,** wherever it came from.

### What to write in §6

For each tool, one line: **which tool**, **what you used it for**, and **how you checked the result**. If it got something wrong, say what — that is useful, and it often belongs in §5 too.

> - AI: ChatGPT — explained the `OSError: [Errno 98]` message, and suggested setting `SO_REUSEADDR`; we tested it, and the error stopped.
> - AI: Claude — suggested pin GP26 for the sensor, which was wrong for our wiring; we checked the pinout diagram and used GP27.
> - AI: Copilot — autocompleted the CSV-writing loop in `gateway.py`; we reviewed it and added the missing `flush()`.

If you did not use any AI tool that week, write "AI: none".

---

## 5 · What each week's report must contain — Weeks 2–6 (Checkpoint 1)

Use this table as a checklist. The exact wording of each §3 question is in that week's guide, under *What to submit*.

| Week | §2 must show | §3 asks | §4 should touch |
|---|---|---|---|
| **2** — One node | Min, max and typical loop duration, **and how long you sampled** | Is your node's response time bounded in any useful way, and which timing model can you honestly assume? | Timing assumption, failure assumption, how a failed node would be detected |
| **3** — Two clocks | Both raw logs, your aligned comparison table, measured drift in **ppm** with the sampling duration | Three parts: drift in ppm vs the 20–50 ppm range; the minimum event separation your data can resolve; whether your two detections are ordered under **happened-before** | Timing — revised with your measured skew |
| **4** — First connection | Comparison table of **order, loss and delay** for serial and Wi-Fi, from your own runs | Which channel is closer to the "perfect link", and what exactly is still missing? | Channel — what your transport really guarantees |
| **5** — Broker | Logs covering the broker being stopped and restarted; **the three experiments**, written as what you observed | Is a retained value true, and what would you add to a message so a subscriber could judge for itself? | Architecture — the broker as a single point everything depends on |
| **6** — Concurrency | For experiments A, B and C: both vectors, the classification, and (for C) which update LWW kept and which it destroyed | Experiment C: what last-write-wins destroyed, and what consequence that would have had | Conflicts — node-to-index mapping and your conflict policy |

By Checkpoint 1, `model-spec.md` should cover all five areas in the last column.

### Reports you already wrote for Weeks 2–5

Sections 5 and 6 are new. **Add them to your Week 2–5 reports before Checkpoint 1** — you will remember those problems better now than later. This is the one allowed edit to an earlier week's folder: **add the two sections; do not change sections 1–4.** Commit it with a clear message, for example `week03: add reflection and AI sections`.

---

## 6 · When the main task did not work

This happens, and it is not a disaster. Write the report anyway, with all six sections. Put what you *did* get in §2, and the full story in §5:

```markdown
## 5. Problems and reflection
Experiment B did not produce a concurrent pair.

- Symptom: ... (evidence/expB_attempt1.txt)
- What we tried: 1) ...  2) ...  3) ...
- Where our understanding runs out: we think ..., but we could not confirm it because ...
```

Then answer §3 as far as your data allows, and say clearly which part of the answer is reasoning without data. A precise account of a failure is worth far more than a vague claim of success.

---

## 7 · Writing in English

What is read for is understanding, not polished English.

- **Short sentences.** One idea per sentence.
- **Numbers with units,** always: `12.4 ms`, `38 ppm`, `2 of 500`.
- **Name things exactly:** `pico-a`, `teamNN-gw`, `evidence/wifi_log.txt` — not "the board" or "the file".
- **Tables are welcome.** Most of §2 should be a table.
- Write `report.md` in **Markdown**, so it renders on GitHub and converts cleanly into your checkpoint PDF.

---

## 8 · Before each checkpoint — report check

For **every week** in the checkpoint:

- [ ] `weekNN/report.md` exists, with all six sections in order
- [ ] §2 has your own numbers, the conditions, and file paths that **really exist** in `evidence/`
- [ ] Every experiment in that week's guide appears in §2, including any that failed
- [ ] §3 answers the question in the first sentence, and quotes a number from §2
- [ ] §4 matches a real commit to `model-spec.md` that week
- [ ] §5 describes at least one real problem — symptom, cause, fix, lesson
- [ ] §6 names both members with their GitHub usernames, and says how AI was used (or "none")
- [ ] Open the file **on GitHub** and check it renders — broken tables and missing images show up there

Then follow [CHECKPOINT_SUBMISSION.md](CHECKPOINT_SUBMISSION.md) to tag and build the PDF.
