# CI/CD Pipeline: Tic Tac Toe App (Jenkins)

**Author:** Ahmad (tech610)
**Repo under pipeline:** [tech610-ttt-app-cicd-jenkins](https://github.com/ahmadjalal156/tech610-ttt-app-cicd-jenkins)
**Status:** Complete. All three jobs working, tested end to end.

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
10. [Job 3: CD deploy](#job-3-cd-deploy)
11. [The EC2 instance](#the-ec2-instance)
12. [Blockers hit and how they were solved](#blockers-hit-and-how-they-were-solved)
13. [Testing the pipeline](#testing-the-pipeline)
14. [Screenshots](#screenshots)

---

## What this pipeline does

In plain English: a developer pushes a change to the **dev** branch. That push automatically triggers Jenkins, which runs the app's tests. Only if those tests pass does the code get merged into **main**. Once it is on main, Jenkins deploys it to an EC2 instance, and the change appears on the live site. Nobody merges or deploys by hand, and untested code never reaches production.

The whole flow in one line:

```
push to dev  ->  test  ->  merge to main  ->  deploy to EC2  ->  live site updated
```

---

## Pipeline diagram

```mermaid
flowchart LR
    A["Developer pushes to<br/>dev branch"] --> B["GitHub<br/>tech610-ttt-app-cicd-jenkins"]
    B -->|webhook fires| C["Job 1<br/>ahmad-ttt-job1-ci-test<br/>checks out dev, runs npm test"]
    C -->|tests fail| F["Pipeline stops.<br/>main untouched."]
    C -->|build stable| D["Job 2<br/>ahmad-ttt-job2-ci-merge<br/>merges dev into main"]
    D -->|Git Publisher pushes| B
    D -->|build stable| E["Job 3<br/>ahmad-job3-cd-deploy<br/>rsync to EC2, restart with pm2"]
    E -->|SSH + rsync| G["EC2 instance<br/>app live on port 3000"]

    style C fill:#dbeafe
    style D fill:#dbeafe
    style E fill:#dcfce7
    style G fill:#dcfce7
    style F fill:#fee2e2
```

Two things the diagram shows:

- **The gate between each job.** The arrows from Job 1 to Job 2 and Job 2 to Job 3 only fire on a **stable** build. A failing test run ends the pipeline early and main stays clean.
- **Code reaches EC2 by rsync over SSH, not by git.** The production box is never asked to clone the repo. Jenkins pushes the tested files to it.

---

## Why we set it up this way

**Tests gate the merge, not a human.** The whole point of CI is that broken code is caught automatically and early. If merging to main depended on someone remembering to check the test results first, it would eventually get skipped. Chaining each job to "trigger only if build is stable" makes that impossible.

**dev and main have different jobs.** dev is where work happens and where things are allowed to break. main is the branch that gets deployed, so it should only ever hold code that has passed its tests. Separating them means a developer can push freely without risking what is in production.

**Separate jobs rather than one big job.** Splitting test, merge, and deploy into three jobs means when something fails, the failure points at exactly which stage broke. A single job doing everything would give one red light and a long log to dig through. In practice this paid off: a broken test showed up as a red Job 1, a bad merge config as a red Job 2, and both were obvious at a glance.

**Deploy with rsync, not git clone on the server.** Production should not need git installed, should not hold repo credentials, and should not be able to pull whatever happens to be on main at any moment. Jenkins copies the exact tested files across with rsync and restarts the app. The server stays a dumb target that only ever receives code.

### Benefits already seen

- **Fast feedback.** A push triggers the tests within seconds, so a mistake is caught while the change is still fresh rather than days later.
- **Failures are specific.** When Job 2 broke, the console output named the exact git command and argument that was wrong. Debugging took minutes.
- **The gate genuinely works.** A frontpage change that broke a unit test was caught by Job 1, the pipeline stopped, and neither main nor the live site was affected (see [Blockers](#blockers-hit-and-how-they-were-solved)).
- **No manual steps.** Once it worked, pushing to dev was the only action needed. Test, merge and deploy all happened on their own.

### Benefits for an organisation

- **Fewer broken releases.** Untested code cannot reach the deployable branch, so the class of bug that comes from "someone merged without running the tests" disappears.
- **Faster, smaller releases.** Because each change is tested, integrated and deployed automatically, teams can ship many small changes a day rather than batching them into a risky monthly release.
- **Consistency.** The build, test and deploy steps run identically every time, on the same agent, regardless of who pushed. It removes the "works on my machine" problem.
- **Auditability.** Every change has a build number, a console log, and a timestamp behind it, so it is clear what was tested, merged and deployed, and when.
- **Developer time back.** Manual testing, merging, and deploying are repetitive and error-prone. Automating them frees developers for actual development.

---

## Repo setup

The repo holds **version 1.2** of the Tic Tac Toe app, with the app source uncompressed in an `app/` folder at the root.

```
tech610-ttt-app-cicd-jenkins/
|-- app/            # the application code (v1.2)
|-- .gitignore      # standard Node ignores (node_modules, logs)
`-- README.md       # states the purpose of the repo
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

Two separate keys are used, one for each destination Jenkins talks to.

### Key 1: GitHub deploy key (Jenkins to GitHub)

Generated purely for Jenkins, separate from my personal GitHub key:

```bash
ssh-keygen -t ed25519 -a 100 -C "jenkins@ttt-3job-cicd" -f ~/.ssh/ahmad-jenkins-2-gh-ttt-app
```

- `-t ed25519` is the modern recommended key type
- `-a 100` sets 100 KDF rounds, making the private key harder to brute force if stolen
- `-f` writes the files straight to `~/.ssh`, so there is no need to navigate into that folder at all

| Half | Goes to | Why |
|---|---|---|
| **Public key** (`.pub`) | GitHub repo > Settings > **Deploy keys** | GitHub needs it to recognise Jenkins |
| **Private key** | Jenkins > Credentials, kind **SSH Username with private key**, username `git` | Jenkins uses it to prove its identity |

**Why a deploy key rather than an account key.** A deploy key is scoped to a single repository. If it leaked, the damage is limited to that one repo. An account-level key would give access to everything I own on GitHub.

**Write access.** The deploy key has **Allow write access** ticked. Job 1 only needs to read, but Job 2 has to push the merge back to GitHub, so write is required. This is the minimum needed rather than a blanket permission.

### Key 2: EC2 key pair (Jenkins to EC2)

The `.pem` key created with the EC2 instance. When the instance launched, AWS placed the **public** key in `~/.ssh/authorized_keys` on the box. The **private** key (the `.pem`) was added to Jenkins as a second credential, username `ubuntu`, used by Job 3's SSH Agent.

**The general rule followed throughout:** the private key stays with whoever is connecting (Jenkins), and the public key sits on whatever is being connected to (GitHub, EC2).

**Security notes applied throughout:**

- SSH keys and the `.ssh` folder are never placed inside a Git repo
- Private keys are never shared, printed into a shared channel, or committed
- Jenkins connects to GitHub over SSH (`git@github.com:...`), not HTTPS with a password
- Production (EC2) never holds git or repo credentials, because code arrives by rsync

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
push to dev  ->  webhook  ->  Job 1  ->  (if stable)  ->  Job 2  ->  (if stable)  ->  Job 3
```

A webhook is a **push, not a pull**: GitHub tells Jenkins the moment a change lands, rather than Jenkins repeatedly polling to ask if anything changed. To confirm the webhook is reaching Jenkins, GitHub shows a **Recent Deliveries** tab on the webhook itself, with a green tick for each successful delivery.

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
| **Additional Behaviours** | **Merge before build** - name of repository `origin`, branch to merge to `main` |
| **Build Triggers** | *(none - chained from Job 1)* |
| **Build steps** | *(none needed)* |
| **Post-build Actions** | **Git Publisher** - tick **Push Only If Build Succeeds** and **Merge Results** |

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

## Job 3: CD deploy

**Name:** `ahmad-job3-cd-deploy`
**Type:** Freestyle project
**Purpose:** deploy the merged, tested code from main to the EC2 instance and restart the app.

| Section | Setting |
|---|---|
| **Source Code Management** | Git, repo SSH URL, Jenkins deploy key credential |
| **Branches to build** | `*/main` (the merged, tested code) |
| **Build Environment** | **SSH Agent**, using the EC2 key credential (username `ubuntu`) |
| **Build Triggers** | *(none - chained from Job 2)* |
| **Build step** | Execute shell (see below) |
| **Post-build Actions** | *(none - end of the chain)* |

**Why SSH Agent.** Ticking SSH Agent and selecting the EC2 credential loads that private key into the agent for the duration of the build. The rsync and ssh commands can then authenticate without a key file being written to disk anywhere.

**Build step:**

```bash
rsync -avz --delete -e "ssh -o StrictHostKeyChecking=no" \
  app/ ubuntu@<EC2-IP>:/home/ubuntu/app/

ssh -o StrictHostKeyChecking=no ubuntu@<EC2-IP> << 'EOF'
  cd /home/ubuntu/app
  npm install
  pm2 delete ttt-app || true
  pm2 start index.js --name ttt-app
  pm2 save
EOF
```

What each part does:

- **`rsync -avz --delete`** copies only what has changed. `--delete` removes files on EC2 that no longer exist in the repo, so old code cannot linger.
- **The trailing slash on `app/`** copies the *contents* of the folder into the target, rather than nesting another `app` folder inside.
- **`-e "ssh -o StrictHostKeyChecking=no"`** tells rsync to transport over SSH and skips the interactive "unknown host, continue?" prompt that would otherwise freeze the build.
- **The heredoc (`<< 'EOF' ... EOF`)** runs that block of commands on the EC2 box over a single SSH connection.
- **`pm2 delete ttt-app || true`** removes the old running process. The `|| true` stops the build failing on the very first run when nothing exists yet.
- **`pm2 start index.js --name ttt-app`** starts the app. `index.js` is the app's entry point and it listens on port 3000.
- **`pm2 save`** persists the process list so the app comes back up if the instance reboots.

**Chaining.** Job 2's post-build action **Build other projects** points at `ahmad-job3-cd-deploy`, trigger only if stable.

**Why rsync and not git clone (the key rule for this job).** The task explicitly forbids cloning main onto production. Copying with rsync means the EC2 instance never needs git installed, never holds repo credentials, and only ever receives the exact files Jenkins has already tested. It cannot accidentally pull an untested commit.

**Result:** the app restarts with the new code and the change is live at `http://<EC2-IP>:3000`.

---

## The EC2 instance

**Image:** Ubuntu 24.04 LTS (the task allows 22.04 or 24.04 for a Node 20 app).

**Dependencies installed** so the app can run:

```bash
# Node 20 via the NodeSource repo (Ubuntu's own repo ships an older version)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# pm2 process manager, keeps the app running and restartable
sudo npm install -g pm2
```

**Security group inbound rules:**

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | Jenkins worker nodes (or anywhere to start) | lets Jenkins rsync and SSH in to deploy |
| Custom TCP | 3000 | Anywhere | lets users view the app in a browser |

**Important detail:** the SSH connection in Job 3 comes from the **Jenkins agent node**, not the controller. The builds ran on `jenkins-node-2204-java17v5`, a separate machine from the controller. So the security group must allow SSH from the agents, not just the controller's IP. Allowing from anywhere gets it working initially; tightening to the Jenkins worker security group is the more secure option.

**One gotcha:** an instance's public IP changes if it is stopped and started, and that IP is hardcoded in Job 3's deploy script. An **Elastic IP** pins the address so the pipeline does not silently break after a stop/start.

---

## Blockers hit and how they were solved

**Jenkins credential failed: `error in libcrypto`**

The Jenkins credential could not read the private key, so it fell back to no key and GitHub returned `Permission denied (publickey)`. The cause was how the key had been pasted. The fix was to paste the **entire** private key including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines, with a trailing newline after the END line. Missing that final newline is a common cause of this exact error.

**Job 2 failed: `ambiguous argument 'Origin/main^{commit}'`**

In the **Merge before build** config, the name of repository had been entered as `Origin` with a capital O. Git is case sensitive, so it looked for a remote called `Origin`, found nothing, and failed before the merge was even attempted. Changing it to lowercase `origin` fixed it and the next build went green. Worth noting the build failed *safely*: because Git Publisher only pushes on success, the console log ended with a line confirming no push would occur.

**Job 1 went red after a frontpage change (the gate working as intended)**

Changing the `<h1>` heading to include a timestamp broke a unit test that asserted the heading was exactly "Tic Tac Toe". `npm test` failed, Job 1 went red, and the pipeline stopped. main and the live site were untouched. This is the single clearest demonstration of why the merge is gated on tests: a change that broke expected behaviour was caught automatically and never shipped. The cleaner fix, suggested by the trainer, was to use the app's built-in `APP_FOOTER_TIMESTAMP` support (line 100 of `server.js`) so the frontpage could show a timestamp without breaking any test.

**Node kept installing as version 18, not 20**

On the EC2 box, `apt install nodejs` gave Node 18. The NodeSource `curl` command only *adds the repository*, it does not install anything, so Ubuntu's own older package was being picked up. The fix was to remove the distro package first (`sudo apt remove nodejs`), then add the NodeSource repo and install from it. Confirmed with `node -v` showing `v20.x`.

**Nearly selected another student's EC2 credential**

Jenkins displays credentials in the dropdown as **username plus description**. Because every EC2 credential on the shared server uses the username `ubuntu`, several entries looked almost identical. The fix was to give my credential a distinctive description including my name, so it is unambiguous in the list.

**Local `git push` failed: `Permission denied (publickey)`**

Unrelated to Jenkins. My personal GitHub key has a custom filename, and SSH only auto-tries default names like `id_ed25519`. The agent also does not survive closing Git Bash. The permanent fix is a `~/.ssh/config` entry:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/tech610-ahmad-gh-key
```

**Timestamps appeared to be an hour out**

Jenkins runs on an EC2 instance set to **UTC**, while the local machine is on **BST (UTC+1)**. Builds looked like they belonged to someone else on the shared server until this was worked out. Confirmed ownership instead by checking the first line of the console output, which names the upstream job and build number that triggered it.

---

## Testing the pipeline

The pipeline was tested by repeatedly changing the frontpage timestamp on the dev branch and confirming the change appeared on the live site each time:

1. Edit the timestamp in `server.js` on the dev branch
2. `git add . && git commit && git push`
3. Watch Job 1, Job 2 and Job 3 run green in sequence
4. Hard refresh `http://<EC2-IP>:3000` and confirm the new timestamp is showing

Running this several times confirmed the pipeline redeploys reliably, which is the point: a developer can push a change and trust it reaches production automatically, without any manual steps.

---

## Screenshots

*To be added.*

1. Job 1 build history showing a green build triggered by a GitHub push
2. Job 1 console output showing the tests passing
3. Job 2 config screen showing **Merge before build** and **Git Publisher**
4. GitHub repo on the **main** branch showing the merged commit
5. Job 3 config showing **SSH Agent** and the Execute shell rsync step
6. Job 3 console output showing the rsync transfer and the pm2 restart
7. All three jobs green on the Jenkins dashboard, in sequence
8. The frontpage on EC2 showing the updated timestamp (change 1)
9. The frontpage on EC2 showing a later timestamp (change 2), proving repeated reliable deploys
10. GitHub webhook **Recent Deliveries** showing a successful delivery
