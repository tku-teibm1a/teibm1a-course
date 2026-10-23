# Distributed Systems & Edge Architectures

**TEIBM1A** · Department of Computer Science and Engineering, Tamkang University
Instructor: Prof. Sheng-Zhi Huang (黃勝志)

This repository holds everything you need for the practical side of the course: the assignment guide for each week, and the rules for how work is submitted and marked.

**Bookmark this page.** It is updated during the semester, and it is always the current version.

---

## Start here

1. Read the **[Submission Guide](SUBMISSION_GUIDE.md)** and the **[Checkpoint Submission guide](CHECKPOINT_SUBMISSION.md)** once, properly. They explain how your repository is structured and how marks are awarded. Most lost marks come from not having read them.
2. Read the **[Report Guide](REPORT_GUIDE.md)**. It explains what goes in each section of your weekly report, and how to write an optional **integrated report** that brings the weeks together, for example with the help of an AI tool.
3. Read **[Doing the Labs Outside Class](LAB_SETUP.md)**. The hands-on work is done where you have space, power and a network you control; this page covers the setup.
4. **One member** creates your team repository inside the course organisation from the `teibm1a-starter` template, named `teibm1a-teamNN` and set to **Private**, then adds the teammate as a collaborator with **Write** access. It is one shared repository for the whole semester.
5. Work through the assignment guide for the current week.

---

## Assignment guides

| Week | Topic | Guide |
|---|---|---|
| 2 | One node, no network | [week02.md](assignments/week02.md) |
| 3 | Two clocks, no agreement | [week03.md](assignments/week03.md) |
| 4 | The first connection | [week04.md](assignments/week04.md) |
| 5 | Three nodes and a broker | [week05.md](assignments/week05.md) |
| 6 | Detecting concurrency | [week06.md](assignments/week06.md) |
| 8 | Building a failure detector | *published in Week 8* |
| 9 | Elections | *published in Week 9* |
| 10 | Aligning the clocks | *published in Week 10* |
| 11–13 | Edge AI and resilience | *published later* |

Week 7 is Checkpoint 1 — a review of Weeks 2 to 6, with no new lab.

---

## Checkpoints

Your repository is reviewed four times. Weekly work is never uploaded anywhere — you just push to GitHub. At **CP1, CP2 and CP3** you also make one formal submission to **iClass**, where marks are recorded: see the **[Checkpoint Submission guide](CHECKPOINT_SUBMISSION.md)**.

| Checkpoint | Week | Covers | Weight |
|---|---|---|---|
| Health check | 4 | — | Not graded |
| **CP1** | 7 | Weeks 2–6 | 14% |
| **CP2** | 11 | Weeks 8–10 | 13% |
| **CP3** | 13 | Weeks 11–13 | 13% |
| Final project | 16 | Showcase | 40% |

Each checkpoint closes at **23:59 on the Tuesday** after that week — the day before the next class.

---

## The three rules that matter most

**Your commit history is part of your grade.** Two teams submitting identical content will not receive identical marks if one worked steadily and the other did not. Both members must commit under their own account.

**Never edit a previous week's folder.** If Week 5 revealed that your Week 3 work was wrong, say so in the Week 5 report and correct `model-spec.md`. The history is part of what is assessed. *(The one exception is adding the new report sections 5 and 6 to Weeks 2–5 before Checkpoint 1.)*

**Being stuck is not penalised. Hiding it is.** Commit what you have, and write down what you tried and where your understanding ran out — that is what report §5 is for. An empty folder tells us nothing.

**Use AI tools, and say how.** You are encouraged to use them. Report §6 records what you used them for and how you checked the result — see the [Report Guide](REPORT_GUIDE.md) §4.

---

## Getting help

Bring questions to class — you do not need to bring your hardware. Bring the exact error message, the log, or the commit that fails; see [LAB_SETUP.md](LAB_SETUP.md) §8.

If something in these guides is wrong, unclear, or does not match your hardware, tell me. Corrections are made here and everyone benefits immediately.
