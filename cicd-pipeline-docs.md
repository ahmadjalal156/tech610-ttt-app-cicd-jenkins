# CI/CD Pipeline: Tic Tac Toe App (Jenkins)

**Author:** Ahmad (tech610)
**Repo under pipeline:** [tech610-ttt-app-cicd-jenkins](https://github.com/ahmadjalal156/tech610-ttt-app-cicd-jenkins)
**Status:** Jobs 1 and 2 complete. Job 3 (deploy) in progress.

---

## Contents

1. [What this pipeline does](#what-this-pipeline-does)
2. [Pipeline diagram](#pipeline-diagram)
3. [Why we set it up this way](#why-we-set-it-up-this-way)
4. [Repo setup](#repo-setup)
5. [Authentication and security](#authentication-and-security)
6. [Webhook and how the pipeline is triggered](#webhook-and-how-the-pipeline-is-triggered)
7. [Job 1: CI test](#job-1-ci-test)
8. [Job 2: CI merge](#job-2-ci-merge)
9. [Other ways to make Job 2 work](#other-ways-to-make-job-2-work)
10. [Blockers hit and how they were solved](#blockers-hit-and-how-they-were-solved)
11. [Screenshots](#screenshots)

---

## What this pipeline does

In plain English: a developer pushes a change to the **dev** branch. That push automatically triggers Jenkins, which runs the app's tests. Only if those tests pass does the code get merged into **main**. Nobody merges to main by hand, and untested code never reaches it.

Job 3 will then take that merged code and deploy it to an EC2 instance, but this document covers Jobs 1 and 2 only.

---

## Pipeline diagram

```mermaid
flowchart LR
    A["Developer pushes to<br/>dev branch"] --> B["GitHub<br/>tech610-ttt-app-cicd-jenkins"]
    B -->|webhook fires| C["Job 1<br/>ahmad-ttt-job1-ci-test<br/>checks out dev, runs npm test"]
    C -->|tests fail| F["Pipeline stops.<br/>main untouched."]
    C -->|build stable| D["Job 2<br/>ahmad-ttt-job2-ci-merge<br/>merges dev into main"]
    D -->|Git Publisher pushes| B
    D -->|build stable| E["Job 3<br/>ahmad-job3-cd-deploy<br/>rsync to EC2, restart app"]
    E --> G["App running on EC2<br/>port 3000"]

    style C fill:#dbeafe
    style D fill:#dbeafe
    style E fill:#e5e7eb
    style F fill:#fee2e2
```

The key thing the diagram shows is the **gate between Job 1 and Job 2**. That arrow only fires on a stable build. A failing test run ends the pipeline there and main stays clean.

---

## Why we set it up this way

**Tests gate the merge, not a human.** The whole point of CI is that broken code is caught automatically and early. If merging to main depended on someone remembering to check the test results first, it would eventually get skipped. Chaining Job 2 to "trigger only if build is stable" makes that impossible.

**dev and main have different jobs.** dev is where work happens and where things are allowed to break. main is the branch that gets deployed, so it should only ever hold code that has passed its tests. Separating them means a developer can push freely without risking what is in production.

**Separate jobs rather than one big job.** Splitting test, merge, and deploy into three jobs means when something fails, the failure points at exactly which stage broke. A single job doing everything would give one red light and a long log to dig through.

### Benefits already seen

- **Fast feedback.** A push triggers the tests within seconds, so a mistake is caught while the change is still fresh rather than days later.
- **Failures are specific.** When Job 2 broke, the console output named the exact git command and argument that was wrong. Debugging took minutes.
- **No manual merging.** Once it worked, pushing to dev was the only action needed. main updated on its own.

### Benefits for an organisation

- **Fewer broken releases.** Untested code cannot reach the deployable branch, so the class of bug that comes from "someone merged without running the tests" disappears.
- **Faster, smaller releases.** Because each change is tested and integrated automatically, teams can ship many small changes a day rather than batching them into a risky monthly release.
- **Consistency.** The build and test steps run identically every time, on the same agent, regardless of who pushed. It removes the "works on my machine" problem.
- **Auditability.** Every change to main has a build number, a console log, and a timestamp behind it, so it is clear what was tested and when.
- **Developer time back.** Manual testing, merging, and deploying are repetitive and error-prone. Automating them frees developers for actual development.

---

## Repo setup

The repo holds **version 1.2** of the Tic Tac Toe app, with the app source uncompressed in an `app/` folder at the root.

```
tech610-ttt-app-cicd-jenkins/
├── app/            # the application code (v1.2)
├── .gitignore      # standard Node ignores (node_modules, logs)
└── README.md       # states the purpose of the repo
```

**Creating the dev branch:**

```bash
cd ~/github/tech610-ttt-app-cicd-jenkins
git checkout -b dev
git push -u origin dev
```

`-b` creates the branch and switches to it in one step. `-u` sets the upstream so later pushes are just `git push`.

---

## Authentication and security

**A dedicated deploy key, not my personal key.** I generated a separate SSH key pair purely for Jenkins:

```bash
ssh-keygen -t ed25519 -a 100 -C "jenkins@ttt-3job-cicd" -f ~/.ssh/ahmad-jenkins-2-gh-ttt-app
```

- `-t ed25519` is the modern recommended key type
- `-a 100` sets 100 KDF rounds, making the private key harder to brute force if stolen
- `-f` writes the files straight to `~/.ssh`, so there is no need to navigate into that folder at all

**Where each half goes:**

| Half | Goes to | Why |
|---|---|---|
| **Public key** (`.pub`) | GitHub repo > Settings > **Deploy keys** | GitHub needs it to recognise Jenkins |
| **Private key** | Jenkins > Credentials, kind **SSH Username with private key**, username `git` | Jenkins uses it to prove its identity |

**Why a deploy key rather than an account key.** A deploy key is scoped to a single repository. If it leaked, the damage is limited to that one repo. An account-level key would give access to everything I own on GitHub.

**Write access.** The deploy key has **Allow write access** ticked. Job 1 only needs to read, but Job 2 has to push the merge back to GitHub, so write is required. This is the minimum needed rather than a blanket permission.

**Security notes applied throughout:**

- SSH keys and the `.ssh` folder are never placed inside a Git repo
- The private key is never shared, printed into a shared channel, or committed
- Jenkins connects to GitHub over SSH (`git@github.com:...`), not HTTPS with a password

---

## Webhook and how the pipeline is triggered

**On GitHub:** repo > **Settings** > **Webhooks** > **Add webhook**

- **Payload URL:** `http://<jenkins-url>:8080/github-webhook/` (the trailing slash matters)
- **Content type:** `application/json`
- **Events:** just the push event

**On Jenkins:** in **Job 1** only, under **Build Triggers**, tick **GitHub hook trigger for GITScm polling**.

Jobs 2 and 3 have **no trigger of their own**. They are chained from the job before them. This is deliberate: it means the only way into the pipeline is a push to dev, and every later stage inherits the gate of the stage before it.

**The trigger chain:**

```
push to dev  →  webhook  →  Job 1  →  (if stable)  →  Job 2  →  (if stable)  →  Job 3
```

To confirm the webhook is reaching Jenkins, GitHub shows a **Recent Deliveries** tab on the webhook itself, with a green tick for each successful delivery.

---

## Job 1: CI test

**Name:** `ahmad-ttt-job1-ci-test`
**Type:** Freestyle project
**Purpose:** run the app's tests against the dev branch.

| Section | Setting |
|---|---|
| **Source Code Management** | Git, repo SSH URL, Jenkins deploy key credential |
| **Branches to build** | `*/dev` |
| **Build Triggers** | GitHub hook trigger for GITScm polling |
| **Build step** | Execute shell (see below) |
| **Post-build Actions** | Build other projects: `ahmad-ttt-job2-ci-merge`, trigger only if build is **stable** |

**Build step:**

```bash
cd app
npm install
npm test
```

`npm install` pulls the dependencies (which are not committed, thanks to `.gitignore`), then `npm test` runs the app's test suite. If any test fails, npm exits with a non-zero code, Jenkins marks the build red, and the chain stops there.

**Result:** a green build means the code on dev is safe to merge.

---

## Job 2: CI merge

**Name:** `ahmad-ttt-job2-ci-merge`
**Type:** Freestyle project
**Purpose:** merge the tested dev branch into main and push it to GitHub.

| Section | Setting |
|---|---|
| **Source Code Management** | Git, repo SSH URL, Jenkins deploy key credential |
| **Branches to build** | `*/dev` |
| **Additional Behaviours** | **Merge before build** — name of repository `origin`, branch to merge to `main` |
| **Build Triggers** | *(none — chained from Job 1)* |
| **Build steps** | *(none needed)* |
| **Post-build Actions** | **Git Publisher** — tick **Push Only If Build Succeeds** and **Merge Results** |

**How the two halves work together:**

- **Merge before build** checks out dev and merges it into main *locally, inside the Jenkins workspace*. At this point nothing has been sent back to GitHub.
- **Git Publisher** is what actually pushes that merge up. Because it is set to push only on success, a failure anywhere in the build means the merge is simply discarded and main is never touched.

That split is the safety mechanism. The merge is always attempted in a disposable workspace first, and only a clean result gets published.

**Result:** main receives the tested code automatically. Verified by switching the branch dropdown on GitHub to `main` and confirming the commit count increased and the change was present.

---

## Other ways to make Job 2 work

Git Publisher is the required approach for this task, but the same outcome can be reached other ways. Documented here for completeness.

**1. Execute shell step**

Skip the plugin and run git directly as a build step:

```bash
git checkout main
git pull origin main
git merge origin/dev
git push origin main
```

- **Pro:** transparent, every command is visible in the console output, and it works without the Git Publisher plugin installed.
- **Con:** you have to handle credentials yourself (usually by wrapping it in the **SSH Agent** build environment), and you must set `git config user.name` and `user.email` or the merge commit fails. More moving parts to get wrong.

**2. Declarative pipeline (Jenkinsfile)**

Define the whole pipeline as code in a `Jenkinsfile` committed to the repo:

```groovy
stage('Merge to main') {
    steps {
        sshagent(['ahmad-jenkins-2-gh-ttt-app']) {
            sh '''
                git checkout main
                git merge origin/dev
                git push origin main
            '''
        }
    }
}
```

- **Pro:** the pipeline is version-controlled alongside the app, reviewable in pull requests, and rebuilds identically if Jenkins is ever lost. This is the approach most organisations use in practice.
- **Con:** more upfront learning than clicking through the freestyle UI, and harder to debug when you are new to it.

**3. GitHub-side merge via pull request**

Open a PR from dev to main with a branch protection rule requiring the Jenkins check to pass before merging.

- **Pro:** adds human code review to the gate, which many teams want.
- **Con:** not fully automated. It needs someone to click merge, which is the opposite of what this task is demonstrating.

---

## Blockers hit and how they were solved

**Jenkins credential failed: `error in libcrypto`**

The Jenkins credential could not read the private key, so it fell back to no key and GitHub returned `Permission denied (publickey)`. The cause was how the key had been pasted. The fix was to paste the **entire** private key including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines, with a trailing newline after the END line. Missing that final newline is a common cause of this exact error.

**Job 2 failed: `ambiguous argument 'Origin/main^{commit}'`**

In the **Merge before build** config, the name of repository had been entered as `Origin` with a capital O. Git is case sensitive, so it looked for a remote called `Origin`, found nothing, and failed before the merge was even attempted. Changing it to lowercase `origin` fixed it and the next build went green.

Worth noting the build failed *safely*: because Git Publisher only pushes on success, the console log ended with a line confirming no push would occur. main was never at risk.

**Local `git push` failed: `Permission denied (publickey)`**

Unrelated to Jenkins. My personal GitHub key has a custom filename, and SSH only auto-tries default names like `id_ed25519`. The agent also does not survive closing Git Bash. Short-term fix was `ssh-add`; the permanent fix is a `~/.ssh/config` entry:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/tech610-ahmad-gh-key
```

**Timestamps appeared to be an hour out**

Jenkins runs on an EC2 instance set to **UTC**, while the local machine is on **BST (UTC+1)**. Builds looked like they belonged to someone else on the shared server until this was worked out. Confirmed ownership instead by checking the first line of the console output, which names the upstream job and build number that triggered it.

---

## Screenshots

*To be added.*

1. Job 1 build history showing a green build triggered by a GitHub push
2. Job 1 console output showing the tests passing
3. Job 2 config screen showing **Merge before build** and **Git Publisher**
4. Job 2 green build, with the console line confirming the merge
5. GitHub repo on the **main** branch showing the merged commit and updated commit count
6. GitHub webhook **Recent Deliveries** showing a successful delivery
