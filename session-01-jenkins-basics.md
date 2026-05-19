# Session 01 — Jenkins Basics

**Goal:** Install Jenkins on EC2, wire Docker as the build agent, run a pipeline that builds and pushes the ShopStack API image to Docker Hub automatically.

---

## Who Does What

Understanding who owns what is more important than memorizing commands.

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Jenkins Exists

Manual builds break — someone skips a step, uses the wrong version, forgets to push. Jenkins automates the exact same checklist every time. Consistent, repeatable, no humans in the loop.

**In ShopStack terms:**
- Before Jenkins: you SSH into EC2, manually run `docker build`, `docker push`. Every time. By hand.
- After Jenkins: you push code to GitHub. Jenkins does the rest automatically.

---

## Visual Map

```
EC2 (t3.small)
├── Jenkins Master (port 8080)
│   ├── Job: shopstack-api
│   │   └── Jenkinsfile ← pulled from GitHub on every build  🚀 YOU OWN THIS
│   └── Credentials Store
│       └── dockerhub-credentials (username + access token)  🚀 YOU OWN THIS
└── Docker
    └── Agent container (docker:27-dind)
        ├── Spun up fresh per job
        ├── Runs: docker build → docker push
        └── Destroyed after job completes
```

---

## Jenkins Architecture

```
git push origin main                     🧑‍💻 Dev triggers this
        ↓
GitHub sends webhook to Jenkins          🚀 You wired this connection
        ↓
Jenkins Master
- Reads Jenkinsfile from GitHub          🚀 You wrote this file
- Schedules the job
- Injects credentials safely             🚀 You stored these
        ↓
Docker Agent Container
- Fresh container, clean environment
- Runs: docker build → docker login → docker push
- Container destroyed after
        ↓
Docker Hub
- shopstack-api:COMMIT_SHA pushed        🚀 Outcome of your pipeline
- shopstack-api:latest pushed
```

**Master** — schedules jobs, stores config, serves UI on port 8080. Does not run builds itself.

**Agent** — runs the actual build. In your setup: a Docker container spun up on demand and destroyed after the job. No permanent worker EC2 needed.

---

## Jenkinsfile — What It Is and Who Owns It

🚀 **DevOps Work** — The Jenkinsfile is the pipeline definition. You write it, you own it, you version control it. This is what separates a DevOps engineer from someone who just clicks buttons.

```groovy
pipeline {
    agent { docker { image '...' } }    // WHERE to run

    environment {
        KEY = 'value'                   // VARIABLES — reused across stages
    }

    stages {
        stage('Stage Name') {           // ONE logical step — name shows in UI
            steps {
                sh 'shell command'      // WHAT to run inside the agent
            }
        }
    }

    post {
        always {
            sh 'cleanup command'        // ALWAYS runs — success or failure
        }
    }
}
```

| Block | What it does |
|---|---|
| `agent` | Which container to use as the build machine |
| `environment` | Variables reused across all stages |
| `stage('name')` | One logical unit — name shows in Jenkins UI |
| `sh '...'` | Run a shell command inside the agent container |
| `withCredentials` | Pull secrets from Jenkins store — never hardcoded |
| `post { always }` | Runs whether build passes or fails — cleanup |

---

## The ShopStack Jenkinsfile

🚀 **DevOps Work** — Save this at `~/shopstack/Jenkinsfile` and commit it to GitHub. This file IS the pipeline.

```groovy
pipeline {
    agent {
        docker {
            image 'docker:27-dind'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_IMAGE = 'akhiltejadoosari/shopstack-api'
        DOCKER_TAG = "${GIT_COMMIT[0..7]}"           // commit SHA — traces image back to exact code
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm                         // clone repo onto agent
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ./services/api'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',   // ID stored in Jenkins
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push ${DOCKER_IMAGE}:${DOCKER_TAG}'
                    sh 'docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest'
                    sh 'docker push ${DOCKER_IMAGE}:latest'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
            sh 'docker system prune -f'              // clean up after every build
        }
    }
}
```

---

## The Dockerfile — Not Your Job

🧑‍💻 **Dev Work** — `./services/api/Dockerfile` was written by the developer. You don't touch it. You just need to know it exists at that path so your pipeline can find it.

---

## Installation — Exact Order

### 🧪 Lab Setup — Free up disk space
> Not a DevOps skill. EC2 ships with bloat. One-time cleanup.

```bash
sudo systemctl stop snapd
sudo apt-get remove --purge -y snapd
sudo rm -rf /var/lib/snapd /snap
sudo apt-get autoremove -y
sudo apt-get clean
df -h /
# need at least 1.5G free before continuing
```

---

### 🧪 Lab Setup — Resize EC2 volume if needed
> Not a DevOps skill. Infrastructure sizing is done before you arrive.
> If disk is under 15G, resize in AWS Console:
> EC2 → Volumes → select volume → Modify → set size to 20 → Save
> Then on EC2:

```bash
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
# should now show ~19G
```

---

### 🧪 Lab Setup — Install Java 21
> Not a DevOps skill. Jenkins needs Java to run. In a real company Jenkins already exists.
> Jenkins 2.555+ requires Java 21 minimum. Java 17 installs fine but Jenkins silently fails to start.

```bash
sudo apt-get update -y
sudo apt-get install -y openjdk-21-jdk
java -version
# expected: openjdk version "21.x.x"
```

---

### 🧪 Lab Setup — Install Jenkins
> Not a DevOps skill. Jenkins is pre-installed in real companies.

> ⚠️ The standard Jenkins key URL (jenkins.io-2023.key) is EXPIRED as of early 2026.
> Do NOT use the wget method. Use the keyserver method below.

```bash
# Fetch current key directly from keyserver by ID
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys 7198F4B714ABFC68
sudo gpg --export 7198F4B714ABFC68 | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg

# Add Jenkins repo
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install
sudo apt-get update -y
sudo apt-get install -y jenkins
```

---

### 🧪 Lab Setup — Start Jenkins
> Not a DevOps skill. Starting services is ops work. In a real company Jenkins runs 24/7.

```bash
sudo systemctl reset-failed jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
# expected: Active: active (running)
```

---

### ⚙️ Ops Work — Wire Docker to Jenkins
> Giving Jenkins permission to run Docker commands.
> In a real company the ops team handles this during Jenkins provisioning.
> Worth understanding because you'll hit the permission error if it's missing.

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
groups jenkins
# expected: jenkins : jenkins docker
```

---

### 🧪 Lab Setup — Open port 8080
> Not a DevOps skill. Network/firewall rules are managed by cloud/infra team.
> AWS Console → EC2 → Security Groups → your instance SG → Inbound rules → Add rule
> Type: Custom TCP | Port: 8080 | Source: 0.0.0.0/0

---

### 🧪 Lab Setup — First-time Jenkins UI setup
> Not a DevOps skill. One-time wizard, done once when Jenkins is first installed.

```bash
# Get the unlock password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

1. Open `http://YOUR_IP:8080`
2. Paste the password → Continue
3. Install suggested plugins → wait 2-3 minutes
4. Create admin user → Save and Continue → Save and Finish → Start using Jenkins

---

### 🧪 Lab Setup — Install Docker Pipeline plugin
> Not a DevOps skill. Plugin management is ops/admin work.
> Manage Jenkins → Plugins → Available plugins → search `Docker Pipeline` → Install
> Check "Restart when no jobs running"

---

### 🚀 DevOps Work — Add Docker Hub credentials to Jenkins
> THIS is your job. You own the secrets that allow Jenkins to push images.
> Never hardcode credentials in the Jenkinsfile. Always store in Jenkins credential store.

**Step 1 — Create Docker Hub access token:**
Docker Hub → Account Settings → Personal access tokens → Generate new token
- Name: `jenkins`
- Expiration: None
- Access: Read & Write
- Copy immediately — shown only once

**Step 2 — Add to Jenkins:**
Manage Jenkins → Credentials → System → Global credentials → Add credentials
- Kind: Username with password
- Username: `akhiltejadoosari`
- Password: paste the access token
- ID: `dockerhub-credentials`
- Description: `Docker Hub`
→ Create

> The ID `dockerhub-credentials` must match exactly what's in the Jenkinsfile `credentialsId` field.

---

### 🚀 DevOps Work — Create the Jenkins pipeline job
> THIS is your job. Connecting the repo to Jenkins is DevOps work.

1. New Item → name: `shopstack-api` → Pipeline → OK
2. Scroll to Pipeline section
3. Definition: `Pipeline script from SCM`
4. SCM: Git
5. Repository URL: `https://github.com/AkhilTejaDoosari/shopstack.git`
6. Credentials: none (public repo)
7. Branch: `*/main`
8. Script Path: `Jenkinsfile`
9. Save → Build Now

---

### 🚀 DevOps Work — Wire GitHub webhook
> THIS is your job. Auto-triggering builds on code push is the whole point of CI.

**In Jenkins job:**
Configure → Triggers → check **GitHub hook trigger for GITScm polling** → Save

**In GitHub repo:**
Settings → Webhooks → Add webhook
- Payload URL: `http://YOUR_EC2_IP:8080/github-webhook/`
- Content type: `application/json`
- Events: Just the push event
- Active: ✅
→ Add webhook

> Every `git push origin main` now automatically triggers a Jenkins build. No manual clicks.

> ⚠️ EC2 IP changes on every restart. When that happens:
> 1. `curl -s http://checkip.amazonaws.com` → get new IP
> 2. GitHub → repo Settings → Webhooks → update Payload URL

---

## Expected Build Output

```
Started by GitHub push
Obtained Jenkinsfile from git https://github.com/AkhilTejaDoosari/shopstack.git
docker pull docker:27-dind
docker run ... docker:27-dind           ← agent container starts
docker build -t shopstack-api:a3f9c12 ./services/api
Login Succeeded
docker push shopstack-api:a3f9c12       ← pushed with commit SHA tag
docker push shopstack-api:latest        ← also tagged latest
docker logout
docker system prune -f                  ← auto-cleanup
docker stop ... && docker rm ...        ← agent destroyed
Finished: SUCCESS
```

> ℹ️ You will see this warning — it is NOT a failure, ignore it:
> `The container started but didn't run the expected command`
> Jenkins falls back to the host Docker socket. Build runs fine.

---

## Commit SHA Tagging — Why It Matters

```
# Bad — build number tells you nothing about the code
shopstack-api:4

# Good — commit SHA traces directly to the code
shopstack-api:a3f9c12

# Verify: what code is in that image?
git show a3f9c12
# → shows the exact commit, author, and diff
```

In production, if something breaks you run `git show COMMIT_SHA` and see exactly what changed.

---

## Bugs You Will Hit

**Jenkins won't start — wrong Java version**
```
Running with Java 17 which is older than minimum required version (Java 21)
```
Fix:
```bash
sudo apt-get install -y openjdk-21-jdk
sudo systemctl reset-failed jenkins
sudo systemctl start jenkins
```

---

**Jenkins repo not trusted — GPG error**
```
NO_PUBKEY 7198F4B714ABFC68
The repository is not signed
```
Cause: The 2023 key file is expired. The wget method does not work.
Fix:
```bash
sudo rm -f /usr/share/keyrings/jenkins-keyring.gpg
sudo rm -f /etc/apt/sources.list.d/jenkins.list
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys 7198F4B714ABFC68
sudo gpg --export 7198F4B714ABFC68 | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install -y jenkins
```

---

**Docker permission denied**
```
permission denied while trying to connect to the Docker daemon socket
```
Fix:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

**Push fails — unauthorized**
```
error: unauthorized: incorrect username or password
```
Cause: Credential ID mismatch or expired token.
Fix: Manage Jenkins → Credentials → verify ID is exactly `dockerhub-credentials` → update token.

---

**Disk full — build fails mid-way**
```
no space left on device
```
Fix:
```bash
docker image prune -a -f
df -h /
# if still under 2G free, resize the EC2 volume (see Lab Setup above)
```

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Jenkins won't start | Java 17, needs 21 | Install openjdk-21-jdk, reset-failed, start |
| apt install jenkins fails | Expired GPG key | Fetch key from keyserver by ID |
| Docker permission denied | jenkins not in docker group | usermod -aG docker jenkins + restart |
| Dockerfile not found | Wrong context path in Jenkinsfile | Verify `./services/api` exists in repo |
| Push unauthorized | Wrong credential ID or bad token | Check ID = `dockerhub-credentials`, regenerate token |
| Jenkins UI unreachable | Port 8080 blocked | AWS SG → add inbound TCP 8080 |
| Build fails mid-way | Disk full during Go/large build | docker image prune -a -f, resize volume |
| Webhook not triggering | EC2 IP changed after restart | Update Payload URL in GitHub webhook settings |

---

## Quick Reference

| What | Value / Command |
|---|---|
| Jenkins UI | `http://YOUR_IP:8080` |
| Jenkins version | 2.555.2 |
| Java required | 21+ |
| Jenkins status | `sudo systemctl status jenkins` |
| Jenkins logs | `journalctl -u jenkins.service --no-pager \| tail -30` |
| Initial password | `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` |
| Get EC2 IP | `curl -s http://checkip.amazonaws.com` |
| Restart Jenkins | `sudo systemctl restart jenkins` |
| Check docker group | `groups jenkins` |
| Credential ID | `dockerhub-credentials` |
| Jenkinsfile location | `~/shopstack/Jenkinsfile` |
| Agent image | `docker:27-dind` |
| Docker Hub result | `akhiltejadoosari/shopstack-api:COMMIT_SHA` and `:latest` |
| Clean Docker disk | `docker image prune -a -f` |
| Verify commit SHA | `git show COMMIT_SHA` |
