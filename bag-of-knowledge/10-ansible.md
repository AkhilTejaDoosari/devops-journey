[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Ansible

**Hire importance at $25–40/hr:** 2/10 — know what it is. Grows to 5/10 at $55/hr.

**One line:** Ansible configures servers automatically. You describe the desired state of a server. Ansible makes it so — on one server or a thousand.

> ⏳ Runbook notes not generated yet.
> Come back Day 42 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What configuration management is and why Bash scripts are not enough

A developer hands you a fresh EC2 instance. You need to install Docker, clone ShopStack, configure environment variables, and start the stack. You write a bash script. It works on the first run. On the second run it tries to install Docker again — apt throws warnings, the clone fails because the directory already exists, the stack restarts unexpectedly.

Bash scripts run commands. They do not check state. Running the same bash script twice has unpredictable results.

Ansible modules check state before acting. The `apt` module checks if Docker is already installed. If it is — nothing happens. If it is not — it installs it. Running the same Ansible playbook twice produces identical results. This property is called idempotency and it is the entire value of configuration management tools.

### Agentless — the developer's advantage

Most configuration management tools require an agent — a small process running on every server that receives instructions. Ansible does not. It uses SSH. You run Ansible on your laptop or a CI runner. It SSHes into the EC2, runs commands, reads output, and disconnects. Nothing is installed on the server. Nothing is running on the server permanently. Just SSH access and Python — which Ubuntu already has.

### The ShopStack scenario Ansible solves

```
Fresh EC2 — nothing installed
          ↓
ansible-playbook -i inventory.ini setup.yml
          ↓
Ansible SSHes into EC2
          ↓
Installs Docker (checks first — skips if already installed)
Adds ubuntu user to docker group
Clones shopstack repo
Creates .env file from template
Runs docker compose up -d
          ↓
ShopStack is running
Second run — nothing changes — all tasks report OK
```

### ShopStack files Ansible touches

```
shopstack/infra/ansible/inventory.ini    ← list of EC2 hosts
shopstack/infra/ansible/setup.yml        ← playbook — what to do on each host
shopstack/infra/ansible/templates/       ← Jinja2 templates for config files
shopstack/infra/ansible/vars/            ← variables used in the playbook
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- Idempotency — running a playbook twice produces the same result — the core concept
- Ansible is agentless — it uses SSH — nothing installed on the target server
- Inventory — the list of hosts Ansible manages — static INI file at minimum
- Playbook — a YAML file of plays — each play targets hosts and runs tasks
- Task — a single unit of work using a module — `apt`, `copy`, `service`, `git`
- Module — the tool that does the work — `apt` installs packages, `copy` copies files
- Variables — `{{ variable_name }}` — Jinja2 syntax — used in tasks and templates
- `--check` flag — dry run — shows what would change without changing anything
- Tags — run only specific tasks from a playbook — `--tags "docker"`
- The difference between `command` module and `shell` module — when to use each

**When you can explain all ten without hesitation — stop. Move to Bash.**

### What unlocks higher pay

At $55/hr you add: roles for reusable playbook structure, Ansible Vault for encrypting secrets, dynamic inventory from AWS APIs, handlers for conditional task execution, and Galaxy roles from the community. At $80/hr you architect the entire configuration management strategy for a fleet of servers across environments.

📚 Deep dive → Coming Day 42 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

Idempotency is the most important concept in Ansible. It means you can run a playbook ten times and the result is always the same. The first run changes things — installs packages, creates files, starts services. Every subsequent run checks state first and changes nothing if the desired state already exists. This is what separates Ansible from a bash script. A bash script does not check. Ansible always checks.

Ansible is agentless. It connects to servers over SSH. You need two things on the target — SSH access and Python. Ubuntu has both by default. When you run a playbook, Ansible transfers small Python scripts to the target, executes them, reads the output, and removes them. Nothing persists on the server after Ansible finishes.

An inventory file lists the hosts Ansible manages. A minimal inventory is an INI file with IP addresses or hostnames. You group hosts — `[web]`, `[db]` — and target groups in your playbook. ShopStack's inventory has one host — the EC2 instance.

A playbook is a YAML file. It contains one or more plays. Each play targets a group of hosts and runs a list of tasks. Each task calls one module with specific parameters. The `apt` module installs a package. The `copy` module copies a file. The `service` module starts or stops a service. The `git` module clones a repository.

The `command` module runs a command directly — no shell features, no pipes, no redirects. The `shell` module runs through `/bin/sh` — supports pipes, redirects, and shell syntax. Use `command` by default — it is safer and more predictable. Use `shell` only when you need shell features that `command` cannot provide.

Variables let you parameterise tasks. `{{ postgres_password }}` in a task or template is replaced with the variable's value at runtime. Variables live in `vars/` files, inline in the playbook, or passed on the command line with `-e`. Templates use Jinja2 — the same syntax — to generate config files from variables. A single template can produce different `.env` files for different environments by changing the variables.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- Task reports CHANGED every run → it is not idempotent — use the right module
- Task reports OK every run → desired state already exists — correct behaviour
- Need to test without changing → `--check` flag — dry run
- Need to run only the Docker tasks → `--tags "docker"`
- Need to pass a variable at runtime → `-e "var=value"`
- SSH connection refused → EC2 not running or key not configured in inventory

---

### Domain Awareness

- Ansible Vault — encrypt secrets in playbooks — `ansible-vault encrypt vars/secrets.yml`
- Ansible Galaxy — community roles — install and use instead of writing from scratch
- Dynamic inventory — query AWS API for hosts instead of a static INI file
- Handlers — tasks that run only when notified — restart Nginx only if config changed
- Roles — reusable, structured playbook components — the standard for large projects
- AWX / Ansible Tower — web UI and API for running Ansible at scale

---

### The one insight that separates copiers from understanders

Ansible modules are idempotent by design. You are not writing instructions. You are declaring desired state.

When you write `apt: name=docker-ce state=present`, you are not saying "install Docker." You are saying "Docker should be present." Ansible checks if it is. If yes — task reports OK, nothing changes. If no — task reports CHANGED, installs it. The module decides whether to act. You declare the outcome. This is why Ansible scales to a thousand servers — you describe what every server should look like and Ansible makes each one match, skipping anything already correct.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Use the prompt in `00-notes-pending.md` → Ansible section.
> Pull content from the generated notes section: `06-ansible-playbook-cli-reference`.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 42 evening.
> Fill in based on your own experience during the Ansible week.

### Preview — Learn this yourself no matter what

- Idempotency — the core concept — every interview question about Ansible leads here
- Agentless — SSH only — why this matters for adoption and security
- The difference between CHANGED and OK in playbook output — what each means
- `command` vs `shell` module — when to use each and why it matters
- `--check` flag — dry run — non-negotiable before running on production servers

### Preview — Use AI for this

- First draft of a playbook for a specific configuration task
- Jinja2 template syntax for a specific config file
- Ansible Vault commands for encrypting and decrypting secrets
- Dynamic inventory script for AWS EC2

📚 Deep dive → Coming Day 42 evening — see `00-notes-pending.md`
