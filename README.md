# DevOps Journey

A 60-day hands-on DevOps learning journal — building, breaking, and deploying a real application across every tool in the modern DevOps stack.

---

## The Three Repos

| Repo | What it is | Link |
|---|---|---|
| `devops-runbook` | Personal notes — every tool, every concept | [github.com/AkhilTejaDoosari/devops-runbook](https://github.com/AkhilTejaDoosari/devops-runbook) |
| `shopstack` | The app — 3-tier polyglot containerised store | [github.com/AkhilTejaDoosari/shopstack](https://github.com/AkhilTejaDoosari/shopstack) |
| `devops-journey` | This repo — daily sessions, tests, notes | You are here |

---

## Target

```
Role:       Junior DevOps Engineer
Pay:        $30-40/hour
Day 60:     First application sent
Day 90-120: First offer
```

---

## Your Day — Every Single Day

```
6:00 AM   Wake up
6:30 AM   Read notes — one section only (45 min)
7:15 AM   Breakfast
7:30 AM   Hands on terminal — apply what you read (2 hours)
9:30 AM   Break (30 min)
10:00 AM  Hands on continued OR interview questions out loud (1.5 hours)
11:30 AM  Resume and LinkedIn (30 min)
12:00 PM  Lunch — no screens
1:00 PM   Watch one DevOps engineer on YouTube doing what you studied (45 min)
1:45 PM   Write one thing you learned in your own words (15 min)
2:00 PM   Done
```

Total: 6 hours focused. Not 12 hours half-attention.

---

## 60 Day Roadmap

| Week | Days | Tool | Infrastructure | Cost | Resume line earned |
|---|---|---|---|---|---|
| 1 | 1-7 | Docker | EC2 t3.small | $0 free tier | "Containerised 3-tier polyglot app, published images to DockerHub" |
| 2 | 8-14 | Kubernetes | k3s on same EC2 | $0 free tier | "Deployed and managed app on Kubernetes on AWS" |
| 3 | 15-21 | CI/CD | Same EC2 + GitHub Actions | $0 | "Built CI/CD pipeline, automated deployments with GitOps" |
| 4 | 22-28 | Observability | Same EC2 | $0 | "Implemented monitoring, dashboards, and alerting" |
| 5 | 29-35 | AWS | Real EKS 2 days | ~$5 | "Deployed production workload on AWS EKS" |
| 6 | 36-42 | Terraform | Terraform creates EKS | ~$5 | "Provisioned cloud infrastructure with Terraform" |
| 7 | 43-49 | Ansible + Bash | EC2 free tier | $0 | "Automated server configuration with Ansible" |
| 8 | 50-60 | Resume + Interview | Nothing new | $0 | Sending applications every day |

Total AWS cost: ~$10

---

## Structure

```
devops-journey/
├── README.md              ← you are here — 60-day plan and roadmap
├── week1/
│   ├── README.md          ← week summary and sessions table
│   ├── day1/
│   │   ├── README.md      ← checklist, commands, what you did
│   │   ├── TEST.md        ← questions only — use for revision
│   │   └── TEST_ANSWERS.md← answers — check after attempting
│   ├── day2/
│   └── ...
├── week2/
├── week3/
├── week4/
├── week5/
├── week6/
├── week7/
├── week8/
├── interview/
│   ├── README.md          ← all 30 questions with answers
│   ├── docker.md
│   ├── kubernetes.md
│   ├── cicd.md
│   ├── observability.md
│   ├── aws.md
│   └── terraform.md
└── resume/
    └── akhilteja-devops-resume.md
```

---

## How to Use This Repo

**Daily:** After each session create or update the day folder. Write what you did, what commands you used, what you learned.

**Day 3 revision:** Open TEST.md from day 1. Answer every question out loud. Check TEST_ANSWERS.md. Do not look at README.md until you have answered.

**Day 7 revision:** Same. Faster this time.

**Day 15 and 30:** Same. If it is second nature move on. If not repeat.

**Interview prep:** Open `interview/docker.md`. Answer every question in 30 seconds. No notes.

---

## Resume — Built Week by Week

Every week you complete adds one bullet. By week 8 the resume is done.

See current resume: [resume/akhilteja-devops-resume.md](./resume/akhilteja-devops-resume.md)
