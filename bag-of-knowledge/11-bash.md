[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Bash

**Hire importance at $25–40/hr:** 6/10 — write working scripts without googling syntax. Grows to 7/10 at $55/hr.

**One line:** Bash glues every other tool together. Health checks, deploy scripts, CI steps — all Bash. You do not need to be a wizard. You need to write a working 30-line script without hesitation.

> ⏳ Runbook notes not generated yet.
> Come back Day 42 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What a developer does that makes Bash necessary

A developer finishes building the API. They hand you a Dockerfile and a docker-compose.yml. Now you need to: check if all 5 ShopStack containers are healthy, alert if one is down, build and push a new image after a code change, and set up the EC2 on a fresh machine. None of these are a single command. Each one is a sequence of commands with conditional logic.

Bash lets you write that sequence once, save it as a file, and run it reliably every time. `scripts/health-check.sh` checks all 5 services. `scripts/deploy.sh` builds, tags, and pushes. `scripts/setup.sh` installs Docker and starts the stack on a fresh EC2. These scripts are infrastructure — they run in CI, they run at 3am, they run by engineers who are not you.

### What makes a script safe vs fragile

A fragile script runs commands in sequence and ignores failures. If `docker build` fails, the script continues to `docker push` — pushing nothing, or worse, the old broken image. A deployment succeeds in the logs but a broken image is in production.

A safe script has `set -e` at the top — it exits the moment any command fails. It quotes every variable — `"$CONTAINER_NAME"` not `$CONTAINER_NAME` — so a name with a space does not break the command. It checks exit codes for critical operations. It logs what it is doing with timestamps so you can read the history at 3am.

### The exit code mental model — everything flows from this

Every command returns an exit code when it finishes. `0` means success. Anything non-zero means failure.

```
curl -sf http://localhost:8080/api/health
echo $?   ← 0 if healthy, non-zero if not

docker build -t api .
echo $?   ← 0 if build succeeded, 1 if failed
```

`if`, `&&`, `||`, and `set -e` all work on exit codes. Once you understand this, every conditional in Bash is logical — you are just asking "did the last thing succeed?"

```
docker build . && docker push image   ← push only if build succeeded
docker stop api || echo "already stopped"   ← continue even if stop fails
if ! docker ps | grep -q "api"; then alert; fi   ← if api not running, alert
```

### ShopStack scripts Bash writes

```
shopstack/scripts/health-check.sh    ← curl all 5 service endpoints, alert if any fail
shopstack/scripts/deploy.sh          ← build, tag with git SHA, push to Docker Hub
shopstack/scripts/setup.sh           ← install Docker, clone repo, start stack on EC2
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- `#!/bin/bash` and `set -e` at the top of every script — what each does and why
- Variables — `VAR=value`, `$VAR`, always quote `"$VAR"` — why quoting matters
- Exit codes — `0` is success, non-zero is failure — `$?` captures the last exit code
- `&&` runs the second command only if the first succeeded
- `||` runs the second command only if the first failed
- `if [ condition ]; then ... fi` — the basic conditional structure
- `$(command)` — command substitution — capture output into a variable
- `for item in list; do ... done` — loop over items
- `echo "text" >> file` — append to a file — vs `>` which overwrites
- `grep -q "pattern"` — silent grep — returns exit code only — no output

**When you can explain all ten without hesitation — you own Bash at the floor level.**

### What unlocks higher pay

At $55/hr you add: functions for reusable script logic, `trap` for cleanup on exit or error, argument parsing with `$1 $2 $@`, robust error messages with line numbers, heredocs for multi-line strings, and complex pipelines with `awk` and `sed` for log processing. At $80/hr you write production-grade scripts that handle edge cases, failures, and concurrent execution safely.

📚 Deep dive → Coming Day 42 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

`#!/bin/bash` tells the OS which interpreter to use when the script is executed directly. Without it, the OS guesses — sometimes wrong. `set -e` tells Bash to exit immediately if any command returns a non-zero exit code. Without `set -e`, a failing `docker build` does not stop the script — it continues to `docker push` and deploys nothing or something broken.

Variables are assigned without spaces — `IMAGE=shopstack-api` not `IMAGE = shopstack-api`. Referenced with a dollar sign — `$IMAGE`. Always quote them — `"$IMAGE"` — because if the value contains a space, an unquoted variable splits into multiple arguments. `docker stop $CONTAINER_NAME` where `CONTAINER_NAME="my container"` runs `docker stop my container` — two arguments. `docker stop "$CONTAINER_NAME"` keeps it as one.

Exit codes are the language Bash uses to communicate success and failure. Every command returns one. `0` means success. Any non-zero number means failure — the specific number often indicates the type of failure. `$?` is the variable that holds the last exit code. Check it immediately after the command — the next command replaces it.

Command substitution captures the output of a command into a variable. `IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)` runs the curl command and stores the result in `$IP`. The `$()` syntax is the modern form — always use this over the older backtick syntax.

`grep -q "pattern" file` searches silently — it returns exit code 0 if the pattern is found, non-zero if not. No output. This is how you use grep in conditionals — `if grep -q "ERROR" log.txt; then alert; fi` — without printing the matching lines.

`curl -sf http://url` makes a silent curl that fails on HTTP errors. `-s` suppresses progress output. `-f` returns a non-zero exit code for HTTP 4xx and 5xx responses. Without `-f`, `curl` returns 0 even if the server returns 500 — it got a response, so as far as curl is concerned it succeeded. With `-f`, a 500 response is a failure.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- Script continues after a failure → `set -e` missing at the top
- Variable with spaces breaking a command → missing quotes around `"$VAR"`
- Need to run command B only if command A succeeded → `A && B`
- Need to check if a service is running → `docker ps | grep -q "service-name"`
- Need to capture command output → `VAR=$(command)`
- Need to loop over all ShopStack services → `for SERVICE in frontend api worker db adminer; do`

---

### Domain Awareness

- `getopts` — argument parsing for scripts with flags like `-v` and `-h`
- `trap` — cleanup handler — runs a function when the script exits or errors
- Here documents — multi-line strings passed to a command
- Process substitution — `<(command)` — advanced pipeline patterns
- `awk` and `sed` — powerful text processing — always look up the exact syntax
- `jq` — JSON parser for Bash — query JSON output from APIs and Docker inspect

---

### The one insight that separates copiers from understanders

A Bash script is a sequence of commands with exit code awareness.

Once you understand that every command returns an exit code and that `set -e`, `&&`, `||`, and `if` all operate on exit codes — Bash stops being cryptic syntax and becomes logical. You are not memorising incantations. You are building conditional logic around "did this succeed or fail." That is the entire language. Everything else is syntax details you look up.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Use the prompt in `00-notes-pending.md` → Bash section.
> Pull content from the generated notes section: `06-the-shopstack-scripts`.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.
> The three ShopStack scripts — health-check.sh, deploy.sh, setup.sh — are the primary source.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Fill in based on your own experience writing ShopStack scripts.

### Preview — Learn this yourself no matter what

- Exit codes — 0 is success, non-zero is failure — everything in Bash flows from this
- `set -e` and quoting variables — the two rules that make scripts safe vs fragile
- `&&` and `||` — conditional chaining based on exit codes — must be cold
- `$(command)` — command substitution — how you capture output into a variable
- `grep -q` — silent match for conditionals — used constantly in health checks

### Preview — Use AI for this

- `awk` and `sed` one-liners for specific text transformations
- `getopts` argument parsing for a specific set of flags
- `trap` cleanup handler syntax — what to clean up on exit or error
- Complex `find` commands for log rotation or file management

📚 Deep dive → Coming Day 42 evening — see `00-notes-pending.md`
