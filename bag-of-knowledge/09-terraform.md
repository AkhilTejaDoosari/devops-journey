[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Terraform

**Hire importance at $25–40/hr:** 3/10 — know what it is and have used it. Grows to 7/10 at $55/hr.

**One line:** Terraform turns cloud infrastructure into code — reproducible, version-controlled, and reviewable like any other file in the repo.

> ⏳ Runbook notes not generated yet.
> Come back Day 35 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What the problem is that Terraform solves

Before infrastructure as code, a developer or DevOps engineer clicked through the AWS console to create an EC2 instance, a Security Group, a VPC. No record of what was clicked. No way to reproduce it exactly. Another engineer clicks through the same console for a staging environment and gets something slightly different. Six months later nobody knows what was created or why.

Terraform solves this by making infrastructure a text file. You write HCL — HashiCorp Configuration Language — that describes the infrastructure you want. `aws_instance` for an EC2. `aws_security_group` for a firewall. `aws_vpc` for a network. Terraform reads the file, compares it to what actually exists in AWS, and makes the changes needed to reach the desired state.

### The three-way comparison — the mental model that makes everything make sense

Terraform operates by comparing three things simultaneously:

```
HCL files         ← what you want
State file        ← what Terraform last created
Real AWS          ← what actually exists right now
```

`terraform plan` compares all three and shows you what would change. `terraform apply` makes those changes. The state file is Terraform's memory — it maps every resource in your HCL to the real resource ID in AWS. Without the state file, Terraform does not know what it created and cannot manage it.

This is why the state file is critical and must never be edited manually.

### Why ShopStack needs Terraform

ShopStack runs on EC2. Every time EC2 restarts, the IP changes. If you provision EC2 manually through the AWS console, you cannot reliably reproduce the exact configuration — same instance type, same Security Group rules, same key pair, same tags. With Terraform, the EC2 is defined in `main.tf`. Any engineer runs `terraform apply` and gets an identical environment.

### ShopStack files Terraform touches

```
shopstack/infra/terraform/main.tf          ← resource definitions
shopstack/infra/terraform/variables.tf     ← parameterised inputs
shopstack/infra/terraform/outputs.tf       ← values exposed after apply
shopstack/infra/terraform/terraform.tfstate ← never edit this manually
S3 bucket                                   ← where remote state lives
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- Infrastructure as code means the same file always produces the same infrastructure
- HCL — what a resource block looks like — `resource "aws_instance" "ec2" {}`
- Provider — what it is — AWS, GCP, Azure — Terraform talks to the cloud through it
- `terraform init` downloads the provider — must run before anything else
- `terraform plan` shows what would change — always run before apply
- `terraform apply` makes the changes — review the plan output before confirming
- State file — Terraform's memory — maps HCL resources to real cloud resource IDs
- Never edit `terraform.tfstate` manually — corrupt it and Terraform loses track of everything
- Remote state in S3 — why storing state locally is dangerous on a team
- `terraform destroy` deletes everything the config created — irreversible without a backup

**When you can explain all ten without hesitation — stop. Move to Ansible.**

### What unlocks higher pay

At $55/hr you add: writing reusable modules, remote state with S3 and DynamoDB locking, importing existing resources into state, workspace strategy for multiple environments, reading and fixing plan output for complex changes, and integrating Terraform into CI/CD pipelines. At $80/hr you design the module architecture and own the state management strategy for an entire organisation.

📚 Deep dive → Coming Day 35 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

Infrastructure as code means the infrastructure is reproducible. The same `terraform apply` on the same files in the same state produces the same result. You can destroy the entire ShopStack environment and rebuild it in minutes. You can review infrastructure changes in a pull request before they reach production. You can track who changed what and when in Git history.

A resource block is the building block. `resource "aws_instance" "shopstack_ec2"` declares an EC2 instance. Inside the block, you set the AMI, instance type, key name, and security groups. Terraform reads this and creates or updates the real EC2 to match.

`terraform plan` is the most important command. It compares your HCL, the state file, and real AWS, then shows you exactly what would be created, changed, or destroyed — before anything happens. Never skip plan. A plan that shows 47 resources being destroyed when you only changed an instance type is a sign something is wrong.

The state file is Terraform's memory. When you run `terraform apply` and create an EC2, Terraform writes the real EC2 instance ID to the state file — `i-0a1b2c3d4e5f`. Next time you run plan, Terraform knows `i-0a1b2c3d4e5f` is the real resource managed by your HCL. If you delete the state file, Terraform has no memory — it thinks nothing exists and would try to create everything again, creating duplicates.

Remote state in S3 means the state file lives in an S3 bucket instead of locally. This is required on a team because if two engineers apply at the same time from local state, they create conflicting versions. S3 with DynamoDB locking allows only one apply at a time — the second engineer's apply waits until the first finishes.

`terraform destroy` deletes everything the configuration created — EC2, Security Groups, VPC, everything. It reads the state file and destroys every resource listed in it. This is not reversible unless you have a backup of the state file and the resources can be recreated. Only run it when you genuinely want everything gone.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- Fresh machine — new project → `terraform init` first, always
- Want to see what would change → `terraform plan` before every apply
- Ready to make changes → `terraform apply` — review plan output, then confirm
- Resource exists in AWS but not in state → `terraform import` to bring it under management
- State file corrupted or lost → restore from S3 backup — this is an emergency
- Two engineers applying at the same time → DynamoDB locking prevents this if configured

---

### Domain Awareness

- Terraform Cloud — managed remote state and run execution — alternative to S3 backend
- Terragrunt — wrapper around Terraform for DRY module configuration
- Sentinel — policy as code for Terraform — enforce rules before apply
- OpenTofu — open-source fork of Terraform after licence change
- CDK for Terraform — write Terraform in TypeScript or Python instead of HCL
- Drift detection — when real infrastructure diverges from state without Terraform

---

### The one insight that separates copiers from understanders

Terraform state is the contract between your code and reality.

The HCL file says what you want. The state file says what Terraform last created. Real AWS says what actually exists. When all three agree — no drift, no surprises. When they diverge — something changed outside of Terraform, or the state was corrupted, or a resource was manually deleted. Every Terraform problem traces back to a disagreement between these three sources of truth. Understand the three-way comparison and every Terraform error makes sense.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 35 evening.
> Use the prompt in `00-notes-pending.md` → Terraform section.
> Pull content from the generated notes section: `06-terraform-cli-reference`.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 35 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 35 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 35 evening.
> Fill in based on your own experience during the Terraform week.

### Preview — Learn this yourself no matter what

- The three-way comparison — HCL, state file, real AWS — this is what every Terraform question tests
- Why state must never be edited manually — corrupt it and Terraform loses track of everything
- `terraform plan` before every apply — non-negotiable — what it shows and how to read it
- `terraform destroy` is irreversible — know what it deletes before you run it
- Remote state in S3 — why it is required on a team — locking prevents concurrent applies

### Preview — Use AI for this

- First draft of a Terraform resource block for a specific AWS service
- Variable and output block syntax
- Remote state backend configuration for S3 with DynamoDB locking
- Import command syntax for bringing existing resources into state

📚 Deep dive → Coming Day 35 evening — see `00-notes-pending.md`
