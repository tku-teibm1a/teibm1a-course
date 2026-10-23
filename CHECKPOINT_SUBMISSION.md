# Checkpoint Submission — iClass

**Distributed Systems & Edge Architectures (TEIBM1A)**

Your weekly work lives in GitHub. **Marks are recorded in iClass.** At each checkpoint you connect the two with one formal submission.

---

## 1 · The rule in one line

| When | Where | What |
|---|---|---|
| Every week | **GitHub** | Commit and push as you work. Nothing goes to iClass. |
| At CP1, CP2, CP3 | **GitHub** + **iClass** | Tag your repository, then upload one PDF to iClass. |

Nothing else is submitted through iClass. The final project is submitted separately; details are announced in Week 14.

---

## 2 · When

| Checkpoint | Week | Covers | Deadline |
|---|---|---|---|
| **CP1** | 7 | Weeks 2–6 | 23:59, Tuesday 3 November |
| **CP2** | 11 | Weeks 8–10 | 23:59, Tuesday 1 December |
| **CP3** | 13 | Weeks 11–13 | 23:59, Tuesday 15 December |

The iClass submission closes at the same moment as the checkpoint. A late iClass upload is a late checkpoint.

**One member submits for the team** — iClass has no team groups, so only one upload per team. Every member's **name and student ID** must be on the cover page, or that member's submission cannot be recorded.

---

## 3 · Step 1 — Tag your repository

A tag freezes exactly what you are submitting. Later commits do not change it.

If you wrote an integrated report, **commit and push it first**, so the tag includes it.

```bash
git tag cp1
git push origin cp1
```

Use `cp1`, `cp2` or `cp3` as appropriate. Confirm the tag appears on GitHub under **Releases → Tags** before continuing.

If you need to re-tag before the deadline — you fixed something — move it:

```bash
git tag -f cp1
git push -f origin cp1
```

---

## 4 · Step 2 — Build one PDF

A single PDF, containing in this order:

1. **Cover page** — team number, **every member's name, student ID and GitHub username**, the checkpoint (e.g. *CP1*), your repository URL, and the tag name. For example, `cover.md`:

   ```markdown
   # TEIBM1A — Checkpoint 1 — Team 03

   | Name | Student ID | GitHub |
   |---|---|---|
   | Wang Xiao-Ming | xxxxxxxxx | xmwang |
   | Chen Mei-Ling | xxxxxxxxx | mlchen |

   Repository: https://github.com/<org>/teibm1a-team03
   Tag: cp1
   ```
2. **Your integrated report**, `cp1-report.md` — *only if you wrote one.* It is optional; see the [Report Guide](REPORT_GUIDE.md) §9.
3. **Every week's `report.md`** in the checkpoint's range, in week order.
4. **`model-spec.md`**, as it stands at the tag.

That is all. **Do not** paste code, raw logs or data into the PDF — those are assessed from the repository. Where a report refers to evidence, the file path is enough.

Name the file:

```
TEIBM1A_Team03_CP1.pdf
```

### Two ways to make it

**With pandoc** (quickest, if installed):

```bash
pandoc cover.md week02/report.md week03/report.md week04/report.md \
       week05/report.md week06/report.md model-spec.md \
       -o TEIBM1A_Team03_CP1.pdf
```

If you wrote an integrated report, add `cp1-report.md` straight after `cover.md`.

**Without any tools:** open each `report.md` on GitHub, use the browser's *Print → Save as PDF*, then merge the PDFs with any PDF tool.

Building the PDF takes about **15 minutes**. Do not let formatting consume more time than that — plain and readable is exactly what is wanted.

---

## 5 · Step 3 — Upload to iClass

Upload the PDF to the matching checkpoint assignment in iClass, and paste into the text box:

```
Team 03: Wang Xiao-Ming (xxxxxxxxx), Chen Mei-Ling (xxxxxxxxx)
Repository: https://github.com/<org>/teibm1a-team03
Tag: cp1
```

---

## 6 · How the two are used

| Assessed from | What |
|---|---|
| **The PDF in iClass** | Your weekly reports (and integrated report, if any), and your model specification |
| **Your repository at the tag** | Code, evidence files, video, and whether the system worked |
| **Your repository's full history** | Commit distribution — who committed, and when |

The PDF is generated from your repository, so the two should always agree. **If they ever differ, the repository at the tag is what counts.**

Commit distribution is read from GitHub, not from the PDF. It cannot be improved by the upload, and it cannot be harmed by it.

---

## 7 · Common mistakes

| Mistake | Consequence |
|---|---|
| Forgetting to push the tag | The tag exists only on your laptop, and the reviewed state is whatever was last pushed |
| Tagging after 23:59 | Commits after the deadline do not count toward that checkpoint |
| A report missing from the PDF | That week's reasoning mark cannot be awarded |
| Pasting code into the PDF | Not penalised, but unnecessary — it makes the PDF harder to read |
| Both members uploading | Only one upload per team is marked; agree who submits |
| A member missing from the cover | Their submission cannot be recorded |

---

## 8 · Before you upload

1. The tag is visible on GitHub, not just on your laptop.
2. The PDF contains every weekly report in the checkpoint's range (and your integrated report, if you wrote one), and `model-spec.md`.
3. The cover page has every member's name and student ID, the repository URL and the tag.
4. The file is named `TEIBM1A_TeamNN_CPn.pdf`.

Before building the PDF, go through the report check in the [Report Guide](REPORT_GUIDE.md) §8 for every week.

If something in a week never worked, the report should already say so — see the Submission Guide. The PDF is not the place to explain it for the first time.
