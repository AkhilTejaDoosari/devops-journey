[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# How To Use This System

Read this once. Then never again. It does not change.

---

## What This Folder Is

This is your compass for the entire 60-day sprint and beyond.

It is not a study guide. You do not read it to learn things. It is not a reference manual. You do not read it cover to cover.

It is a decision system. It tells you exactly which file to open and which section to read based on what moment you are in right now.

One trigger. One file. One section. Close it. Go back to the terminal.

---

## The Three Retention Levels

Every concept and command in every tool file is labelled with one of these three levels. Read this once. It explains everything.

---

**Own These Cold**

Concept in brain. Syntax in brain. You type it blind. No trigger needed — it just comes out.

Example: `ss -tulnp` — your hands type it before your brain finishes the thought. You do not open anything. You do not ask anyone. It just comes out.

---

**Know The Category**

Concept in brain. Situation rings the bell. You know exactly which command category to reach for. Flags and exact syntax → combat sheet or AI.

Example: disk full → your brain immediately says `df` then `du` → you open the combat sheet → you find the exact command → you run it.

---

**Domain Awareness**

No bell. No syntax. Just awareness that this tool or topic exists. When someone mentions it or a situation hints at it — your brain says "I know there is something for this, let me find it."

Example: someone says "rotate your logs" → you know log rotation is a thing → you ask AI how to set it up → you learn it in the moment → you use it.

---



You are always in one of four moments. Every file in your entire system belongs to exactly one of them.

---

### Moment 1 — Before Studying A Tool

> You are about to start a new tool week. You want to know your target before you open a single note.

**Open:** `bag-of-knowledge/[tool].md`

**Read:** Section 0 → Section 1 → Section 2

**Then:** Close it. Open `devops-journey/week-X/day-X/readme.md`. Read the day notes. Work the checklist on the terminal.

Do not read the combat sheet yet. Do not read the peace of mind questions yet. The bag gives you awareness — why this tool exists, what your target is, what must live in your brain. The day readme gives you the actual study content. The terminal is where the learning happens.

---

### Moment 2 — While Doing Hands-On Work

> ShopStack is running. You are on the terminal. You are working through the day checklist.

**Open:** `devops-journey/week-X/day-X/readme.md`

**Use:** The checklist — one task at a time

**Lab:** `shopstack/` — always running, always the real target

Do not open bag-of-knowledge during this moment. The terminal teaches you. The file reflects what the terminal taught. If you are reading instead of doing — stop. Go back to the terminal.

If a concept does not click → open `devops-runbook/` — go deep, get the aha, come back.

If something breaks → jump to Moment 3.

---

### Moment 3 — When Something Breaks

> A container is down. A command failed. You need to fix it now.

**Open:** `bag-of-knowledge/[tool].md`

**Go to:** Section 3 only — the combat sheet

**Format:** What breaks → Command → ShopStack example

If the combat sheet does not fix it: read the logs first — always. AI second — translate the error, not solve the problem. Never let AI diagnose before you read the raw output yourself.

---

### Moment 4 — Before An Interview

> Interview tomorrow. You want to find your gaps before they do.

**Open:** `bag-of-knowledge/[tool].md`
**Go to:** Section 5 only — peace of mind

Section 5 is your readiness test — not your study material. It does not teach you anything. It reveals what you own and what you do not.

**The rule:** Answer every question out loud before opening the toggle.

**If the answer comes out clean — no hesitation:** You own it. Move on to the next question.

**If you stumble — hesitated or got it wrong:** Do not re-read the whole tool. Find the specific weak spot.

```
I know the concept but forgot the detail
→ Go to Section 3 combat sheet or Section 4 ShopStack map
→ Do that specific task on the terminal
→ Come back and answer the question again

I don't actually understand this concept
→ Go to devops-runbook/ — read that topic only
→ Then terminal — do it on ShopStack
→ Come back and answer the question again
```

The runbook fills the gap that Section 5 reveals. Section 5 is the test. The runbook is the recovery. The terminal is where the gap actually closes.

**Use Section 5 every Sunday — not just before interviews.** Answer all 10 questions cold. Mark what you stumbled on. Fix only those weak spots during the week. By Day 60 every answer comes out clean.

---

## The Complete Folder Map

```
~/Repositories/
│
├── devops-runbook/              DEPTH ON DEMAND
│     notes/                    Open when concept does not click.
│       01. Linux     ✅         Go deep. Get the aha. Come back.
│       02. Git       ✅         Do not live here daily.
│       03. Networking ✅
│       04. Docker    ✅
│       05-11. ...    ❌ pending
│
├── devops-journey/              YOUR DAILY WORK
│     week-1/
│       day-X/
│         readme.md             Open during Moment 2
│         test.md               Open after session ends
│     bag-of-knowledge/         YOUR COMPASS — this folder
│       00-readme.md            Read once — you are here now
│       00-salary-ladder.md     Open once a week
│       00-notes-pending.md     Open when starting a new tool week
│       01-linux.md             Moments 1, 3, 4
│       02-git.md               Moments 1, 3, 4
│       03-networking.md        Moments 1, 3, 4
│       04-docker.md            Moments 1, 3, 4
│       05-kubernetes.md        Moments 1, 3, 4
│       06-github-actions.md    Moments 1, 3, 4
│       07-argocd.md            Moments 1, 3, 4
│       08-observability.md     Moments 1, 3, 4
│       09-terraform.md         Moments 1, 3, 4
│       10-ansible.md           Moments 1, 3, 4
│       11-bash.md              Moments 1, 3, 4
│
└── shopstack/                   THE LAB
      infra/                    Always running during hands-on work.
      services/                 The real target for every task.
```

---

## The One Rule

> The terminal teaches you. The file reflects what the terminal taught.

If you are reading a file to understand something — stop. Close the file. Open the terminal. Do the thing. Come back after.

This is the only thing standing between you and the infinite loop of organising instead of doing.

---

## The Three File Rules

**Rule 1 — One section at a time.** Never open a tool file and read the whole thing. One reason. One section. Close it.

**Rule 2 — Must know lives in your brain. Everything else lives in the file.** Section 2 must know → comes out in interviews and at 3am without any file open. Combat sheet → you look it up. That is not weakness. That is how seniors work.

**Rule 3 — One file change per week maximum.** If you are spending more than that maintaining this system you are avoiding the work.

---

## How To Know The System Is Working

You won today if you can answer yes to all three:

- Did I do something with my hands on ShopStack today?
- Can I explain what I did in 60 seconds without notes?
- Can I answer one peace of mind question I could not answer yesterday?

Three yes — you are moving. That is all that matters.

---

*Built once. Used every day. Never reorganised.*
