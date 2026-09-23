# Assignment Submission Guide

**Distributed Systems & Edge Architectures (TEIBM1A)**
Department of Computer Science and Engineering, Tamkang University

---

## 1. How submission works

You work in a **team of two** and share one hardware cluster and **one GitHub repository** for the entire semester. There is no upload, no email, and no separate hand-in for individual weeks.

Everything you produce goes into your team repository as you go. Your work is reviewed at **four checkpoints** across the semester.

> Your repository is not a place to deliver finished work. It is where your system lives while you build it.

---

## 2. Where the instructions live

Two repositories are involved, and they do different things.

| Repository | Contains | Who edits it |
|---|---|---|
| **Course repository** | The assignment guide for each week, and this document | Instructor only |
| **Your team repository** | Your code, data and reports | Your team |

The weekly assignment guides are in [`assignments/`](assignments/) of the course repository. They are updated during the semester, so always read them there rather than keeping a copy.

---

## 3. Repository structure

Keep to this layout exactly. It is what makes review fast and fair.

```
teibm1a-team<NN>/
├── README.md              # team members, GitHub usernames, hardware serial numbers
├── model-spec.md          # living document — revised most weeks, never rewritten from scratch
├── week02/
│   ├── report.md
│   ├── code/
│   └── evidence/          # data files, photos
├── week03/
│   ├── report.md
│   ├── code/
│   └── evidence/
└── ...
```

Two rules that matter:

- **One folder per week.** Never edit a previous week's folder to fix something. If Week 3 revealed that your Week 2 code was wrong, say so in the Week 3 report and move on. The history is part of what is being assessed.
- **`model-spec.md` is the exception.** It is a single living document that you revise in place. Its git history shows how your understanding of your own system changed.

---

## 4. The weekly report

Every `report.md` uses these five sections, in this order, every week. Do not add sections, and do not reorder them.

```markdown
# Week N — <topic>

## 1. What we built
Two or three sentences. What exists now that did not exist last week?

## 2. Evidence
Your data. Numbers, a table, a short log extract, a photo of the wiring.
Point to files in evidence/ rather than pasting large outputs here.

## 3. Question of the week
The reasoning question set in that week's practical slides. Answer it directly.

## 4. Model spec changes
What you changed in model-spec.md this week, and why. One or two lines.
Write "no change" if nothing changed — but that should be rare.

## 5. Who did what
One line per team member.
```

Sections 2 and 3 carry most of the marks. Section 1 should be short.

---

## 5. Evidence: what counts

| Type | Use it for | Notes |
|---|---|---|
| **Data file** | The default for almost every week | Raw logs, measurements, timing records. Commit the actual file. |
| **Photo** | Confirming a physical setup | One photo is enough. Make the wiring visible. |
| **Video** | Only Weeks 8, 13, and the final showcase | Behaviour that unfolds over time — a node failing, a system degrading. Keep it under 60 seconds. |

**"It worked" is not evidence.** A claim with no data behind it scores zero on that criterion, even if the claim is true.

Real measurements are messy. A log with plausible jitter and the occasional anomaly is more convincing than clean round numbers, and it will be marked accordingly.

---

## 6. Checkpoints

Your repository is reviewed four times. Between checkpoints you are expected to keep working and committing normally.

| Checkpoint | Week | Covers | Weight |
|---|---|---|---|
| Health check | 4 | — | Not graded |
| **CP1** | 7 | Weeks 2–6 | 14% |
| **CP2** | 11 | Weeks 8–10 | 13% |
| **CP3** | 13 | Weeks 11–13 | 13% |
| Final | 16 | Whole semester + showcase | Final project (40%) |

The Week 4 health check exists to catch teams who are stuck or who have not started. It costs you nothing and may save you a great deal.

Each checkpoint closes at **23:59 on the Sunday** of that week. Commits after that point are visible but do not count toward that checkpoint.

---

## 7. How each week is scored

Within a checkpoint, every covered week is marked out of **3**:

| Criterion | 1 mark |
|---|---|
| **It works** | The system does what the week required, demonstrated by your evidence |
| **Evidence is specific** | Real data, real numbers, from your own hardware |
| **Reasoning is correct** | The question of the week is answered correctly and in your own terms |

Each checkpoint then adds up to **3 further marks** for **commit distribution** — see below.

---

## 8. Commit distribution is assessed

This is deliberate, and it is not a formality.

Your commit history is visible, timestamped, and attributed. Two teams submitting identical content will not receive identical marks if one worked steadily across five weeks and the other produced everything the night before the checkpoint.

| Pattern | Marks |
|---|---|
| Regular commits across the covered weeks, from both members | 3 |
| Uneven, or concentrated in a few sittings | 1–2 |
| Nearly all commits in the final 48 hours before the checkpoint | 0 |

**Both members must commit under their own GitHub account.** One person committing on behalf of the team is treated as a distribution failure, regardless of who did the work. If you pair-program on one machine, use `git commit --author` or co-author trailers so the record reflects reality.

---

## 9. Working as a team of two

You share one cluster, so you will often work side by side. That is expected and encouraged. What is not acceptable is a division where one member owns the hardware and the other owns the writing.

**Either member must be able to explain and operate any part of your system.** You may be asked to do so at any point during a practical session. Work may be divided; understanding may not.

---

## 10. Getting stuck

Being stuck is normal and is not penalised. Hiding it is costly.

If something does not work, commit what you have, and write in that week's report what you tried, what you observed, and where your understanding runs out. A clear account of an unresolved problem scores far better than silence, and it is often worth more marks than a working system with no explanation.

The third hour of each session is supervised working time. Use it to ask.
