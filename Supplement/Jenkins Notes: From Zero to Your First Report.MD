# Jenkins: From Zero to Your First Pipeline
---
## Table of Contents

1. [What problem does Jenkins solve?](#1-what-problem-does-jenkins-solve)
2. [Architecture: controller and agents](#2-architecture-controller-and-agents)
3. [The two job types](#3-the-two-job-types)
4. [Vocabulary](#4-vocabulary)
5. [Installing Jenkins with Docker](#5-installing-jenkins-with-docker)
6. [First login](#6-first-login)
7. [Your first Freestyle job](#7-your-first-freestyle-job)
8. [Environment variables and the workspace](#8-environment-variables-and-the-workspace)
9. [Inside the Jenkins filesystem](#9-inside-the-jenkins-filesystem)
10. [Freestyle: clone Git, run Python](#10-freestyle-clone-git-run-python)
11. [Agents and labels](#11-agents-and-labels)
12. [Automatic build triggers](#12-automatic-build-triggers)
13. [Anatomy of a Jenkinsfile](#13-anatomy-of-a-jenkinsfile)
14. [Your first declarative pipeline](#14-your-first-declarative-pipeline)
15. [Moving the pipeline into Git](#15-moving-the-pipeline-into-git)
16. [Multiple stages, and running them in parallel](#16-multiple-stages-and-running-them-in-parallel)
17. [A real Python CI pipeline](#17-a-real-python-ci-pipeline)
18. [Reading build output and reports](#18-reading-build-output-and-reports)
19. [Troubleshooting](#19-troubleshooting)
20. [Practices worth adopting](#20-practices-worth-adopting)

**Appendices**

- [Appendix A — End-to-end walkthrough](#appendix-a--end-to-end-walkthrough)
- [Appendix B — Command and URL reference](#appendix-b--command-and-url-reference)
- [Appendix C — The sample project](#appendix-c--the-sample-project)

---

## 1. What problem does Jenkins solve?

Five developers work on the same product. Everyone changes code daily and combines their work.
Without automation:

- Code works on one laptop and breaks the moment it meets everyone else's.
- Nobody notices a bug until days later, right before shipping.
- Every release means someone manually running a checklist — easy to skip a step.
- Feedback on "did my change break anything?" takes hours instead of minutes.

Concretely: to run a Python script on a server without Jenkins, someone has to SSH in, have the
right permissions, and run it by hand. With Jenkins you click one button, and you also get:

- **A full history** of every run
- **Complete logs** for each run
- **Automatic triggers** — webhook, SCM change, or a cron-like schedule
- **Access control** — the operator needs a Jenkins permission, not a shell account on the server

> **Term — "CI/CD"**
> **CI (Continuous Integration)**: every change is automatically merged and tested, so problems
> surface in minutes. **CD**: Continuous *Delivery* (packaged and ready, human clicks deploy) or
> Continuous *Deployment* (fully automatic release).
> Jenkins is a tool that *runs* the automation. CI/CD is the *practice*.

> **Why Jenkins when GitHub Actions exists?**
> Flexibility: self-hosted, extensible via plugins, and able to reach machines that will never
> touch the public internet. The trade-off is that *you* install, patch, and secure it. Teams with
> air-gapped labs, physical test hardware, or strict compliance requirements stay on Jenkins for
> exactly this reason.

---

## 2. Architecture: controller and agents

**Two pieces:**

1. **The controller** — hosts the web UI, stores configuration and build history, and schedules
   builds out to agents.
2. **The agents** — machines that actually run the build, inside their workspace.

```
Developer pushes a commit to Git
        ↓
Controller learns about it (webhook or polling)
        ↓
Controller triggers the pipeline
        ↓
Controller picks an agent BY LABEL
        ↓
Agent runs the build
        ↓
Test → build → package → publish
```

Label-based selection is the key mechanism: you tag agents with labels, and jobs request agents by
label. Hands-on in Section 11.

### Two kinds of agent

**Permanent (static)** — ordinary Linux or Windows servers, always on, configured once.

**Cloud (dynamic)** — Docker, Kubernetes, or EC2. Jenkins spins up an agent on demand from a
template, runs the build, and throws it away. No always-on fleet to patch, and every build starts
from a clean image.

### How an agent connects

The direction of the connection matters for firewalls:

| Launch method | Direction | When to use |
|---|---|---|
| **SSH** (*Launch agents via SSH*) | Controller → agent | Simplest for permanent Linux agents. Needs an SSH credential and Java on the agent. |
| **Inbound** (*Launch agent by connecting it to the controller*) | Agent → controller | When the controller can't reach the agent (NAT, firewall, DMZ). Transport is TCP on port 50000, or WebSocket over the normal HTTP port (no extra port needed). |

Cloud agents almost always use inbound — hence the image name `jenkins/inbound-agent`.

> **Notice — "building on the built-in node can be a security risk"**
> You'll see this warning on a fresh install. It means you have no agents, so pipelines run
> directly on the controller. Fine while learning; not fine for real work. Once you have agents,
> set the built-in node's **executors to 0** under Manage Jenkins → Nodes → Built-In Node →
> Configure.

---

## 3. The two job types

### Freestyle projects

The simplest starting point. **Think of them as shell scripts that run on a server**, triggerable
by events such as a Git push. Everything is configured through the UI. If you're comfortable with
the Linux command line, you're set.

Still perfectly reasonable for simple, single-step automation. They become painful as logic grows,
because none of it is version-controlled.

### Pipelines

Pipelines use a **`Jenkinsfile`, written in a Groovy-based DSL**, to define the build as code.
A common shape:

| Stage | What happens |
|---|---|
| **Checkout** | Pull code from Git, set up the environment on the agent |
| **Build** | Compile or assemble — a wheel, an executable, a container image |
| **Test** | Run tests against what you built |
| **Package** | Produce the deployable artifact |
| **Deploy** | Push to a registry or environment |

> **Use declarative syntax.** Jenkins supports two pipeline syntaxes: *declarative* (starts with
> `pipeline { ... }`) and *scripted* (starts with `node { ... }`). Declarative is the modern
> default and everything here uses it. When reading examples online, check which one you're looking
> at — mixing them is the most common beginner failure. If you need imperative logic inside
> declarative, wrap it in a `script { ... }` block.

### Other item types you'll see in "New Item"

| Type | Purpose |
|---|---|
| **Folder** | Groups items. Also a namespace for credentials. |
| **Multibranch Pipeline** | Automatically creates a pipeline for **every branch and PR** in one repository that contains a Jenkinsfile. This is where most teams end up. |
| **Multi-configuration project** | A "matrix" job: runs the same job once per combination of axes (e.g. 3 OS × 2 Python versions = 6 runs). For compatibility testing, not deployment. |

---

## 4. Vocabulary

| Term | Plain-English meaning |
|---|---|
| **Job / Project / Item** | A task you've told Jenkins to do. Like a saved recipe. |
| **Build / Run** | One execution of a job. Numbered #1, #2, #3… |
| **Controller** | The Jenkins brain — schedules work, stores config, hosts the UI |
| **Agent / Node** | The machine that actually runs the build |
| **Built-in node** | The controller acting as an agent. Should have 0 executors in production. |
| **Executor** | A slot on a node that runs one build at a time |
| **Workspace** | The per-job directory where Jenkins checks out code and works |
| **Label** | A tag on an agent; jobs use labels to request a kind of agent |
| **Pipeline** | A job defined as code in a Jenkinsfile |
| **Jenkinsfile** | The text file containing the pipeline definition |
| **Stage** | A labeled phase inside a pipeline |
| **Step** | One unit of work inside a stage (`sh`, `echo`, `junit`, …) |
| **Plugin** | An add-on extending Jenkins' abilities |
| **Artifact** | A file a build produces that's worth keeping |
| **Trigger** | What starts a build — button, schedule, webhook, or upstream job |
| **SCM** | Source Control Management (Git, Subversion, …) |
| **Credential** | A stored secret, referenced by ID and never by value |
| **Console Output** | The full text log of a build |
| **JENKINS_HOME** | The controller's data directory — config, jobs, build history, plugins |

---

## 5. Installing Jenkins with Docker

> **Term — "Docker" / "container"**
> Docker runs software inside an isolated bundle called a **container**: a lightweight, disposable
> mini-computer with everything pre-installed. Instead of installing Java and Jenkins on your real
> machine, you tell Docker to run the Jenkins image — and you can delete it cleanly afterward.

**Prerequisite:** Docker Desktop or Docker Engine.

### Create a network first

Later sections need containers to talk to each other by name. Create the network now so you don't
have to redo the install:

```bash
docker network create jenkins
```

### Run Jenkins

```bash
docker run -d --name jenkins \
  --network jenkins \
  --restart unless-stopped \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21
```

| Flag | Meaning |
|---|---|
| `-d` | **Detached** — run in the background, terminal stays free |
| `--name jenkins` | Nickname the container so you can refer to it later |
| `--network jenkins` | Attach to the network created above |
| `--restart unless-stopped` | Restart automatically after a reboot or crash |
| `-p 8080:8080` | **Port mapping** for the web UI |
| `-p 50000:50000` | Inbound-agent TCP port. Only needed if agents connect *to* the controller over TCP. Omit for WebSocket agents or no agents. |
| `-v jenkins_home:/var/jenkins_home` | **Volume** — persist all Jenkins data |
| `jenkins/jenkins:lts-jdk21` | Official LTS image, Java 21 |

> **Java version, as of July 2026.** The current LTS line requires **Java 21 or Java 25**. Java 17
> support was removed at **LTS 2.555.1 on April 15, 2026**, and the Java 17 controller images were
> dropped with it. `lts-jdk21` and `lts-jdk25` are both valid; `lts-jdk17` is not. If you inherit a
> Jenkins running on Java 17, plan the upgrade before the next LTS baseline.

> **Pin the tag for anything that matters.** `lts-jdk21` moves. For a reproducible instance, pin
> the full version (`jenkins/jenkins:2.568.1-jdk21`) or a digest, and upgrade deliberately.

> **Term — "port"**
> A numbered door on your machine that a program listens through. `-p 8080:8080` connects port 8080
> on your machine to port 8080 inside the container, so `http://localhost:8080` reaches Jenkins.

> **Term — "volume"**
> Containers are disposable; delete one and its contents vanish. A **volume** stores data outside
> the container so it survives recreation. Everything Jenkins cares about lives under
> `/var/jenkins_home`, so that one mount makes your instance durable.

**Verify:**

```bash
docker ps
docker logs -f jenkins   # watch startup; Ctrl-C to stop following
```

Then open `http://localhost:8080` (`localhost` means "this same computer" — not somewhere on the
internet).

---

## 6. First login

### Unlock Jenkins

The first screen asks for the initial administrator password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

(The same string is printed in `docker logs jenkins`, between rows of asterisks.)

### Install suggested plugins

Choose **Install suggested plugins**. You get Git, Pipeline, Credentials, Workspace Cleanup,
Timestamper, and other essentials. You don't need to understand each one yet.

> **Not in the suggested set**, and needed later: **Docker** (the cloud plugin, Section 11),
> **Docker Pipeline** (the `docker.build` step, Section 17), **Coverage** (Section 18), and
> **Pipeline Graph View** (Section 18). Install them when you reach those sections.

### Create your admin user

Fill in your details → **Save and Continue** → accept the default Jenkins URL → **Save and Finish**
→ **Start using Jenkins**.

> **Do create a real admin account.** Skipping this leaves the throwaway `admin` account with the
> initial password active — a bad idea on anything reachable by others.

### The UI, item by item

| Element | Purpose |
|---|---|
| **Breadcrumbs** (top) | Navigate the item hierarchy — you'll use these constantly |
| **New Item** | Create jobs, pipelines, and folders |
| **Build History** | Every run across all jobs |
| **Manage Jenkins** | All administration — nodes, clouds, plugins, security, credentials |
| **Build Executor Status** (bottom-left) | Live view of which node is running what |

Three pages under **Manage Jenkins** are worth knowing about now:

- **Nodes** (permanent agents) and **Clouds** (dynamic agent providers) — everything agent-related.
- **Credentials** — where SSH keys, API tokens, and passwords live. Far better than pasting secrets
  into job configuration.
- **Prepare for Shutdown** — stops new builds starting and lets in-flight builds finish. Use it
  before restarting a busy controller instead of just rebooting.

> **One page to be careful with:** **Script Console** runs arbitrary Groovy with full controller
> privileges. Treat access to it as equivalent to root on the controller.

---

## 7. Your first Freestyle job

**Dashboard → New Item.**

> **Don't use spaces in job names.** Jenkins creates a filesystem directory named after the job.
> Spaces break shell one-liners and `dir()` paths later. Use underscores or dashes.

Name it `my_first_job`, select **Freestyle project**, click **OK**.

### The categories of settings

1. **General** — description, discard old builds, parameters
2. **Source Code Management** — supply a Git URL and Jenkins clones it into the workspace
3. **Build Triggers** — what causes the job to run (Section 12)
4. **Build Environment** — notably "Delete workspace before build starts"
5. **Build Steps** — the actual work
6. **Post-build Actions** — publish test results, archive artifacts, notify

### Adding a build step

Under **Build Steps**, choose **Execute shell**. Whatever you write runs on the agent, inside the
job's workspace.

> **Term — "shell" / "command line"**
> A text-based way of telling a computer to do things instead of clicking icons. `echo "hello"`
> prints `hello`.

Start simple:

```bash
echo "hello world"
```

**Save**, then **Build Now**. Under **Build History**, click build `#1`, then **Console Output**.

> **Notice — the empty Git repository error**
> If you opened the Source Code Management section and left the repository URL blank, the first
> build fails immediately with a Git error. **Fix:** Configure → set SCM to **None** → Save.

Everything from here is the same loop: **configure → trigger → observe**, with more steps.

---

## 8. Environment variables and the workspace

### Built-in variables

Jenkins injects a set of variables into every build. The full list for your instance is at
`http://<your-jenkins>/env-vars.html`, linked next to the Execute shell box.

The ones you'll use constantly:

| Variable | Meaning |
|---|---|
| `BUILD_NUMBER` | The build number (1, 2, 3…). **Use this for versioning artifacts and image tags.** |
| `BUILD_URL` | Full URL of this build. Perfect for notifications. |
| `JOB_NAME` | Name of the job, including folder path |
| `WORKSPACE` | Absolute path to the workspace on the agent. **Use this instead of hardcoding paths.** |
| `NODE_NAME` | Name of the agent running the build (`built-in` for the controller) |
| `EXECUTOR_NUMBER` | Which executor slot on that node — handy for allocating per-build ports |
| `GIT_COMMIT` | Commit SHA that was checked out (with the Git plugin) |
| `BRANCH_NAME` | Branch being built — **Multibranch pipelines only.** Not set in a plain Pipeline job. |

Using them:

```bash
echo "build number: ${BUILD_NUMBER}"
echo "build url:    ${BUILD_URL}"
echo "running on:   ${NODE_NAME}"
echo "workspace:    ${WORKSPACE}"
```

> **Notice — "why is everything printed twice?"**
> In Console Output each command appears to repeat. The line beginning with `+` is **the command
> being executed** (the shell's trace output); the line after it is **that command's output**. Not
> a bug.

### Understanding the workspace

```bash
ls -ltr
echo "1234" > test.txt
ls -ltr
```

First run: the first `ls` shows nothing, then `test.txt` is created. View it via the **Workspace**
link on the job page.

> **Notice — workspaces persist between builds**
> Run it a second time and the first `ls` now shows `test.txt` already there. The workspace is
> **not** cleared between builds. Stale files are a classic source of "it passed but shouldn't
> have" — a build can pick up an artifact it didn't actually produce.
>
> **Fix:** in **Build Environment**, check **Delete workspace before build starts**. In pipelines,
> use `cleanWs()`. Both come from the **Workspace Cleanup** plugin, included in the suggested set.

---

## 9. Inside the Jenkins filesystem

```bash
docker exec -it jenkins bash
cd /var/jenkins_home
ls -ltra
```

> **Path warning — `/var/jenkins_home` vs `/var/lib/jenkins`**
> The **Docker image** uses `/var/jenkins_home`. The **Linux .deb/.rpm packages** use
> `/var/lib/jenkins`. **Agents** use whatever remote root you configured (often
> `/home/jenkins/agent`). Never hardcode a controller path into a step that runs on an agent — use
> `${WORKSPACE}`.

```bash
cd workspace && ls -ltra    # a folder named my_first_job — matching the job name exactly
cd my_first_job && cat test.txt   # → 1234
```

### Paths worth knowing

| Path (under `JENKINS_HOME`) | Contains |
|---|---|
| `workspace/` | Per-job working directories (controller / built-in node only) |
| `jobs/<name>/config.xml` | That job's configuration |
| `jobs/<name>/builds/` | Build history, console logs, per-build metadata |
| `plugins/` | Installed plugins |
| `secrets/` | Master encryption key and the initial admin password. **Never share this directory.** |
| `logs/` | Controller logs |

> **Backup, in one sentence:** back up `JENKINS_HOME` — especially `jobs/`, `secrets/`, `users/`,
> and the `*.xml` files — and restore `secrets/` **together with** the rest, because without the
> master key every stored credential becomes unreadable. `workspace/` is disposable.

**Why this section exists:** getting into the filesystem — `docker exec`, `kubectl exec`, or SSH —
is something you will do constantly when troubleshooting. It is frequently faster than reasoning
about the UI.

---

## 10. Freestyle: clone Git, run Python

**New Item** → `my_python_job` → **Freestyle project**.

### Source Code Management

Select **Git** and enter your repository URL.

> **Credentials:** a public repo needs none. A **private** repo needs a credential — add one under
> Manage Jenkins → Credentials (SSH key, or username + personal access token). Never put a token in
> the URL; it ends up in the console log.

### Probe before you assume

Check what's actually on the machine before writing the real build step:

```bash
python --version     # → command not found
python3 --version    # → works
```

So the real build step uses `python3`.

> **Notice — `python` vs `python3`**
> Most modern Linux images ship `python3` with **no `python` alias** (Debian splits it into a
> separate `python-is-python3` package; minimal images often omit it entirely). Assuming `python`
> exists is a classic build failure. **Run a one-line probe job first** — this habit generalizes to
> every tool your build depends on.

### A more realistic build step

Real Python jobs shouldn't install packages into the agent's system Python. Use a virtual
environment inside the workspace:

```bash
#!/usr/bin/env bash
set -euo pipefail

python3 -m venv "${WORKSPACE}/.venv"
source "${WORKSPACE}/.venv/bin/activate"

python -m pip install --upgrade pip
pip install -r requirements.txt

python -m pytest -q
```

> **Probe this too.** `python3 -m venv` is not guaranteed to work just because `python3` exists.
> On Debian/Ubuntu the `venv` module lives in a separate `python3-venv` package, and the failure
> message is `ensurepip is not available`. Before relying on it:
>
> ```bash
> python3 -m venv /tmp/probe && /tmp/probe/bin/pip --version && echo "venv OK"
> ```
>
> If that fails, install the right package in your agent image (Section 11) — don't work around it
> in the pipeline.

> **Shell gotcha — `set -euo pipefail`**
> Jenkins decides pass/fail from the script's **final exit code**. Without `set -e`, a command can
> fail in the middle and the build still goes green because the last line succeeded. Put
> `set -euo pipefail` at the top of every non-trivial shell step. This is one of the highest-value
> habits in this document.
>
> - `-e` exit on any error
> - `-u` error on undefined variables
> - `-o pipefail` a pipeline fails if *any* stage of it fails, not just the last

Save and build. The console shows Jenkins cloning the repository, then running your script.

---

## 11. Agents and labels

**Manage Jenkins → Nodes** for permanent agents; **Manage Jenkins → Clouds** for dynamic ones.

### New Node = permanent agent

| Field | Notes |
|---|---|
| **Number of executors** | Concurrent builds on this machine. Start at 1–2; don't exceed CPU count. |
| **Remote root directory** | A directory dedicated to Jenkins, e.g. `/home/jenkins/agent` |
| **Labels** | Space-separated tags jobs use to select this agent |
| **Usage** | "Use as much as possible" vs "Only build jobs with matching label expressions" — use the latter for special-purpose machines |
| **Launch method** | SSH or inbound. See the table in Section 2. |

### Clouds = dynamic agents

Install a cloud plugin first: **Manage Jenkins → Plugins → Available**, search `Docker` (or
`Kubernetes`), install, restart.

> **Two plugins with confusingly similar names.** **Docker** (`docker-plugin`) provides the
> *cloud* — it provisions containers as agents, and it's the one you need here. **Docker Pipeline**
> (`docker-workflow`) provides the *steps* `docker.build` and `agent { docker { ... } }`, used in
> Section 17. They are independent; most people end up wanting both.

Then **Manage Jenkins → Clouds → New cloud → Docker**. The critical field is the **Docker Host
URI**.

**The Docker Desktop connection problem:** Jenkins runs inside a container; the Docker daemon runs
on your host. The Jenkins container has no route to the host's socket unless you give it one. The
simplest fix is to mount the socket into the Jenkins container:

```
-v /var/run/docker.sock:/var/run/docker.sock
```

and set the URI to `unix:///var/run/docker.sock`. The socket is usually owned by `root:docker`, and
the `jenkins` user inside the container isn't in that group, so you often also need:

```
--group-add "$(getent group docker | cut -d: -f3)"
```

Then click **Test Connection**. A working connection prints the daemon's version string; anything
else, click **Show Details** and read the actual error.

> **🔒 Security — the Docker socket is root.**
> Anything that can reach `/var/run/docker.sock` can start a privileged container and take over the
> host. Never expose the daemon over plain, unauthenticated TCP (port 2375) on a shared machine.
> Acceptable on your own laptop for learning; on anything shared, use TLS on 2376 with client
> certificates, or Kubernetes agents.

### Creating a Docker agent template

**Docker Agent Templates → Add Docker Template.**

| Field | Value | Why |
|---|---|---|
| **Labels** | `docker-agent-python` | How jobs request this agent. **Describe the capability, not the machine.** |
| **Enabled** | ✅ | Otherwise it's never used |
| **Docker Image** | your own image — see below | |
| **Instance Capacity** | `2` | Max concurrent agents. **Keep it low at first** — a misconfiguration can otherwise spawn agents until the host falls over. |
| **Remote File System Root** | `/home/jenkins/agent` | Where the workspace is created inside the container |
| **Connect method** | *Attach Docker container* or *Connect with JNLP* | Attach uses `jenkins/agent:*`; JNLP/inbound uses `jenkins/inbound-agent:*` — the image must match the method |
| **Pull strategy** | **Never pull** for locally built images | Otherwise provisioning fails trying to pull an image that only exists on your host |

### Using a label

In a Freestyle job: **Configure** → check **Restrict where this project can be run** → enter
`docker-agent-python`. In a pipeline it's one line:

```groovy
agent { label 'docker-agent-python' }
```

Labels support expressions too: `linux && python3`, `hw-tester || simulator`.

> **Notice — the phantom "label doesn't exist" warning**
> Two causes: (1) a **trailing space** inserted by autocomplete — delete it; (2) cloud template
> labels aren't always in the label index until an agent has been provisioned once. Running the job
> clears it.

> **Notice — be patient.** Dynamic agents take time to provision: pull the image, start the
> container, connect back. A build sitting in the queue for 30 seconds is normal.

### The agent image doesn't have your tools

Point a Python job at a stock agent image and it **fails**: the code checks out fine, but
`python3: not found`. Stock Jenkins agent images contain a Java runtime and the agent remoting jar.
**Nothing else.**

**Fix: build your own agent image.** Debian-based (recommended — `apt` packaging of `venv` is
predictable):

```dockerfile
# Dockerfile.agent
FROM jenkins/inbound-agent:latest-jdk21

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip git curl \
 && rm -rf /var/lib/apt/lists/*
USER jenkins
```

Alpine-based (smaller; `jenkins/inbound-agent:latest-alpine-jdk21`) works too, but verify `venv`
before relying on it.

Build it, then **verify the toolchain before wiring it into Jenkins**:

```bash
docker build -t jenkins-agent-python:1.0 -f Dockerfile.agent .

# probe the image directly — two seconds, versus a four-minute build
docker run --rm --entrypoint sh jenkins-agent-python:1.0 -c \
  'python3 --version && python3 -m venv /tmp/probe && /tmp/probe/bin/pip --version && echo VENV_OK'
```

Then create a template pointing at that image with the label `docker-agent-python`.

> **Pin your image tags.** `:latest` means your builds change underneath you without warning. Use
> `:1.0`, `:2026-07`, or a digest.

**The lesson:** agents are just containers, and containers only contain what you put in them. This
is also exactly how you handle specialized toolchains or hardware — one label for agents with a
particular SDK, another for the two machines with physical test hardware attached. For
hardware-attached agents, use permanent agents with **1 executor** so two builds can't grab the
same device.

---

## 12. Automatic build triggers

### GitHub webhook (push-based, near-instant)

The repository tells Jenkins about a push, rather than Jenkins asking repeatedly.

1. Install the **GitHub** plugin (in the suggested set).
2. In the job: **Build Triggers → GitHub hook trigger for GITScm polling**.
3. In GitHub: **Settings → Webhooks → Add webhook**, Payload URL
   `http://<your-jenkins>/github-webhook/` (**the trailing slash matters**), content type
   `application/json`, event: *Just the push event*.

> **The job's Git URL must match the webhook's repository.** The trigger matches the pushed
> repository against the SCM URL in the job. A job cloning `git@github.com:me/x` and a webhook from
> `https://github.com/me/y` will silently never fire.

> **Firewalls break webhooks.** A webhook is GitHub making an **inbound** connection to your
> Jenkins. If Jenkins is on a private network, GitHub can't reach it — use a reverse proxy, an
> allow-listed ingress, or fall back to polling.

### Poll SCM (pull-based, works behind a firewall)

Jenkins reaches out to the repository periodically, so no inbound connection is needed:

```
H/5 * * * *
```

**Field order:** `MINUTE HOUR DAY-OF-MONTH MONTH DAY-OF-WEEK`

> **What `H` actually means.** `H` is **hash**, not "random". Jenkins hashes the *job name* to pick
> a fixed offset inside the interval, so every job gets a different but **stable** slot. Job A
> polls at :02, :07, :12…, job B at :04, :09, :14… — spreading load instead of every job hammering
> the SCM at :00.

| Expression | Meaning |
|---|---|
| `H/5 * * * *` | Every 5 minutes, at a job-specific offset |
| `H * * * *` | **Once per hour** — `H` alone picks one minute in the hour |
| `H H(0-6) * * *` | Once daily, at a job-specific time between 00:00 and 06:59 |
| `H H * * 1-5` | Once daily, weekdays only |
| `*/5 * * * *` | Every 5 minutes for *every* job at once — works, but this is what `H/5` exists to avoid |

**Build periodically** is a plain cron schedule that runs whether or not anything changed. Right
for nightly regressions; wrong for reacting to commits.

Also worth knowing: **Build after other projects are built** (chaining) and **Trigger builds
remotely** (`curl` a URL with a token — how external systems kick off builds).

### Confirming it worked

After a commit, the build starts on its own and the build page shows **"Started by an SCM change"**
— proof it was the commit and not a button press.

> **Poll latency is inherent.** With `H/5` the worst case is about five minutes. If you need fast
> feedback, use webhooks. Shortening the poll interval just hammers your SCM server, and on hosted
> Git it can get you rate-limited.

---

## 13. Anatomy of a Jenkinsfile

A keyword, then braces, then indented content. Every level nests the same way.

```groovy
pipeline {
    agent any

    stages {
        stage('Dev') {
            steps {
                echo 'hello world'
            }
        }
    }
}
```

### The building blocks

**1. `pipeline`** — the outermost block of every declarative pipeline.

**2. `agent`** — *where* it runs.

```groovy
agent any                              // any available agent
agent none                             // no global agent; each stage declares its own
agent { label 'docker-agent-python' }  // by label
agent { docker { image 'python:3.13-slim' } }  // run inside a container
```

With `agent any` and no agents configured, the build runs on the controller — see Section 2.

**3. `stages` / `stage`** — the phases of work. Names appear in the UI, so make them meaningful.

**4. `steps`** — the actual work: `sh`, `echo`, `junit`, `archiveArtifacts`, `withCredentials`.

### The blocks that make it production-grade

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    options {
        timeout(time: 30, unit: 'MINUTES')      // never let a hung build run forever
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()                            // requires the Timestamper plugin
        disableConcurrentBuilds()               // when builds share a resource
    }

    parameters {
        string(name: 'TARGET', defaultValue: 'dev', description: 'Deploy target')
        booleanParam(name: 'RUN_SLOW_TESTS', defaultValue: false, description: '')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        API_TOKEN = credentials('my-api-token')   // masked in the log
    }

    triggers {
        pollSCM('H/15 * * * *')
    }

    stages { /* ... */ }

    post {
        always  { junit allowEmptyResults: true, testResults: 'reports/junit*.xml' }
        success { echo "OK: ${env.BUILD_URL}" }
        failure { echo "FAILED: ${env.BUILD_URL}" }
        cleanup { cleanWs() }
    }
}
```

> **`post` is where reliability comes from.** Steps in `post { always { ... } }` run even when the
> stage failed. Test-report publishing belongs there — otherwise a failing test suite aborts the
> stage and you never get the report that tells you *which* test failed.
>
> `post` conditions run in a defined order with **`cleanup` always last**, which is why
> `cleanWs()` in `cleanup` doesn't destroy the workspace before `junit` and `archiveArtifacts` in
> `always` have read it.

### The interpolation trap you will hit

```groovy
sh "echo ${MY_VAR}"    // Groovy substitutes BEFORE the shell sees it
sh 'echo $MY_VAR'      // the SHELL expands it at runtime
```

Both often produce the same output, which is why it's confusing. The difference matters: with
**double quotes**, the value is baked into the command line, so a secret ends up in the console log
and in `ps` output, and a value containing spaces or quotes can break or inject into the command.

```groovy
// wrong — token ends up in the log
sh "curl -H 'Authorization: Bearer ${TOKEN}' https://api.example.com"

// right — the shell reads it from the environment, and Jenkins masks it
withCredentials([string(credentialsId: 'my-api-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
}
```

> **Masking is best-effort.** Jenkins replaces the literal value with `****`. It cannot mask a
> transformed value (base64-encoded, split, echoed one character at a time). Also, a
> **username/password** credential in `environment` binds *three* variables — `API_TOKEN`,
> `API_TOKEN_USR`, and `API_TOKEN_PSW` — which surprises people the first time.

> **Validate before pushing.** Name the file exactly `Jenkinsfile` so editors apply Groovy
> highlighting, and use the built-in linter:
>
> ```bash
> curl -X POST -u user:token -F "jenkinsfile=<Jenkinsfile" \
>   http://localhost:8080/pipeline-model-converter/validate
> ```

---

## 14. Your first declarative pipeline

**New Item** → `my_first_build_pipeline` → **Pipeline** → **OK**.

Two ways to supply the script:

1. **Pipeline script** — write it in the Jenkins UI. Fine for a first experiment.
2. **Pipeline script from SCM** — keep the Jenkinsfile in your repository. **This is the preferred
   method** and the whole point of pipeline-as-code.

### A starter template

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    python3 --version
                    echo "doing build stuff.."
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Testing..'
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    echo "doing test stuff.."
                '''
            }
        }

        stage('Deliver') {
            steps {
                echo 'Delivering..'
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    echo "doing delivery stuff.."
                '''
            }
        }
    }
}
```

Each stage has two kinds of step: `echo`, executed by Jenkins itself, and `sh`, executed by the
agent's shell.

> **Why triple single quotes?** `'''…'''` is a multi-line Groovy string with **no** interpolation —
> `$VAR` reaches the shell untouched. `"""…"""` interpolates in Groovy first. Default to `'''` for
> shell blocks.

> **The shebang inside `sh` is real.** Jenkins writes the script to a temporary file and executes
> it, so `#!/usr/bin/env bash` genuinely selects bash. It must be the **very first characters** of
> the string — `'''` immediately followed by `#!`, with no leading newline or spaces. Without it
> you get `/bin/sh`, where `set -o pipefail` may not exist and `${ARRAY[0]}` fails with
> "Bad substitution".

### Reading the Stage View

Save and build. The view is split into your three stages, each green or red, with per-stage timing.
Click any stage to see its logs.

> **Upstream failure cascades.** If Test contains an error, Deliver never runs — even though
> Deliver's own syntax is fine. That's the correct default. Where you genuinely want to continue,
> be explicit:
>
> ```groovy
> catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
>     sh 'ruff check .'
> }
> ```

---

## 15. Moving the pipeline into Git

Writing the pipeline in the UI works, but the point of pipeline-as-code is version control:
review, history, blame, rollback.

Save the pipeline script as a file named `Jenkinsfile` in your repository root and commit it. Then,
in the job's **Pipeline** section:

- **Definition:** *Pipeline script from SCM*
- **SCM:** Git
- **Repository URL:** your repo (add credentials if private)
- **Branch Specifier:** `*/main` (Jenkins still defaults to `*/master` in places — check it)
- **Script Path:** `Jenkinsfile` — any name and path works as long as this field matches

Save and build. The console confirms it **obtained the Jenkinsfile from Git**.

### Two things that will surprise you

1. **An extra stage appears** — `Declarative: Checkout SCM`, which you didn't write. With
   *Pipeline script from SCM*, Jenkins must clone the repo to read the Jenkinsfile, and that clone
   shows up as a stage. Expected.
2. **The Stage View history clears.** Adding a stage changes the pipeline's shape, so the
   visualization re-baselines. Nothing is lost — the builds still exist and their logs are intact.

> **The file in Git is the source of truth.** A classic waste of an afternoon: editing your local
> copy, or the old inline script in the UI, and wondering why nothing changes. Once the job is set
> to *from SCM*, **only the committed and pushed file matters.**

### Triggers as code

```groovy
triggers {
    pollSCM('H/15 * * * *')
}
```

> **Chicken-and-egg:** a `triggers` block only takes effect **after the pipeline has run at least
> once**, because Jenkins has to read the Jenkinsfile to learn about the trigger. Run it manually
> the first time.

### Where to go next: Multibranch

Once the Jenkinsfile is in the repo, switch from a Pipeline job to a **Multibranch Pipeline**.
Jenkins then discovers every branch and pull request containing a Jenkinsfile and builds each one
automatically — no per-branch job configuration. For most teams this is the destination. It also
unlocks `BRANCH_NAME` and `when { branch '...' }`, both of which do nothing in a plain Pipeline job
(see Section 16).

---

## 16. Multiple stages, and running them in parallel

One Jenkinsfile, multiple stages, progressing through environments in sequence:

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    stages {
        stage('Dev') {
            steps { sh 'echo "deploy to dev"' }
        }

        stage('UAT') {
            steps { sh 'echo "deploy to uat"' }
        }

        stage('Prod') {
            when { branch 'main' }        // Multibranch only — see below
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh 'echo "deploy to prod"'
            }
        }
    }
}
```

The valuable behavior: if Dev fails, you never reach UAT. This is the upstream/downstream cascade
from Section 14, used deliberately as a deployment gate.

> **⚠️ `when { branch 'main' }` only works in a Multibranch Pipeline.** It compares against
> `BRANCH_NAME`, which a plain Pipeline job never sets. In a plain Pipeline job the condition is
> never true and the stage is **silently skipped forever** — a nasty failure mode, because nothing
> errors. If you're on a plain Pipeline job, gate on something that exists:
>
> ```groovy
> when { expression { env.GIT_BRANCH ==~ /(origin\/)?main/ } }
> ```
>
> Or move to a Multibranch Pipeline.

> **`input` holds an executor** while it waits. For long waits, put it outside the agent block.

### Running independent work in parallel

```groovy
stage('Checks') {
    parallel {
        stage('Lint')       { steps { sh './scripts/lint.sh' } }
        stage('Unit tests') { steps { sh './scripts/test.sh' } }
        stage('Type check') { steps { sh './scripts/typecheck.sh' } }
    }
}
```

Each branch needs a free executor, so parallelism is bounded by your agent capacity.

> **Parallel branches share one workspace** unless each declares its own `agent`. That's usually
> what you want — the venv built in an earlier stage is right there — but it means two branches
> writing the same file will collide. Give each branch a distinct output path
> (`reports/junit-lint.xml` vs `reports/junit-tests.xml`), as Section 17 does.

---

## 17. A real Python CI pipeline

Everything below is Python and shell. The sample project it builds — every file, ready to commit —
is in [Appendix C](#appendix-c--the-sample-project).

### Prerequisite: there is no `tools` block for Python

Jenkins ships tool installers for JDK, Maven, Gradle, and Git. **There is no Python installer**, so
`tools { python 'Python3' }` fails with `Tool type "python" does not have an install of the
name...`. Choose one of:

1. **Bake Python into the agent image** (Section 11) — recommended, most reproducible
2. **Run the stage in a container**: `agent { docker { image 'python:3.13-slim' } }` (needs the
   Docker Pipeline plugin and a reachable daemon)
3. **Use the system Python + a venv** — what the examples below do, and the most portable

### Read this before you copy anything: red vs yellow

Three facts interact, and getting them wrong is the most common reason these pipelines behave
unexpectedly. All three were confirmed by running the sample project:

- `pytest` exits **non-zero** when tests fail, and exits **5** when it collects no tests at all.
- `ruff check` exits **1** when it finds violations.
- With `set -e`, either of those **fails the stage immediately**, which makes the build **FAILURE**.

The `junit` step's ability to mark a build **UNSTABLE** (yellow) only applies if the stage itself
did *not* fail. So:

- **Want a failing test to fail the build (red)?** Do nothing — that's the default below, and it's
  a perfectly reasonable policy.
- **Want failing tests to show as UNSTABLE (yellow)** so "tests failed" reads differently from "the
  pipeline broke"? Wrap the step:

  ```groovy
  catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
      sh '... pytest ...'
  }
  ```

Either policy is defensible. What is *not* defensible is assuming you have one when you've
configured the other.

### Stage 1 — CI only (lint + test)

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        PYTHONDONTWRITEBYTECODE       = '1'
        VENV                          = "${WORKSPACE}/.venv"
        PYTHONPATH                    = "${WORKSPACE}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Set up environment') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    python3 -m venv "$VENV"
                    "$VENV/bin/python" -m pip install --upgrade pip
                    "$VENV/bin/pip" install -r requirements.txt -r requirements-dev.txt

                    "$VENV/bin/python" --version
                    "$VENV/bin/pip" list
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    mkdir -p reports
                    "$VENV/bin/ruff" check . \
                        --output-format=junit \
                        --output-file=reports/junit-lint.xml
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    mkdir -p reports
                    "$VENV/bin/pytest" \
                        --junitxml=reports/junit-tests.xml \
                        --cov=app \
                        --cov-report=xml:reports/coverage.xml \
                        --cov-report=term
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'reports/junit-*.xml'
        }
        cleanup {
            cleanWs()
        }
    }
}
```

Key points:

- **`checkout scm`** re-uses the SCM configuration Jenkins already has for the job. Prefer it over
  a hardcoded `git url: '...'` step, which duplicates configuration and breaks on Multibranch.
- **`${WORKSPACE}`, not a hardcoded path.** The workspace differs between controller, Docker agent,
  and Kubernetes pod.
- **Each `sh` step is a separate shell process.** A venv `activate`d in one step is gone in the
  next. That's why the examples call `"$VENV/bin/..."` explicitly — more robust than re-activating
  everywhere. (Alternative: set `PATH = "${VENV}/bin:${PATH}"` in `environment`.)
- **`PYTHONPATH = "${WORKSPACE}"`** so `import app` resolves from a venv that never installed your
  project. **Nuance worth knowing:** if `tests/` contains an `__init__.py`, pytest inserts the repo
  root into `sys.path` on its own and the import works without `PYTHONPATH`. Delete that
  `__init__.py` and you get `ModuleNotFoundError: No module named 'app'`. Setting `PYTHONPATH`
  anyway costs nothing and removes the dependency on that subtlety. (The other option is
  `pip install -e .`, which needs a `pyproject.toml` and is slower.)
- **`--output-file` rather than `>` redirection.** `ruff ... > file` truncates the file the instant
  the shell opens it, so if ruff dies abnormally (exit 2 — bad config, internal error) you're left
  with a zero-byte XML that `junit` then reports as a malformed test report. `--output-file` writes
  only real output. Same reasoning applies to any tool with a native output-file flag.
- **`junit` in `post { always }`** — a failing test suite makes `pytest` exit non-zero and fails
  the stage. Publishing in `post/always` is what gets you the report anyway.
- **`allowEmptyResults: true`** stops the *first* run from failing on a missing report while you're
  wiring things up. Once stable, set it to `false` — otherwise a test stage that silently produced
  nothing looks identical to one that passed.

### Stage 2 — Package

```groovy
        stage('Package') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    "$VENV/bin/pip" install build
                    "$VENV/bin/python" -m build --wheel --outdir dist/
                    ls -l dist/
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/*.whl', fingerprint: true
                }
            }
        }
```

This requires a `pyproject.toml` in the repository root — `python -m build` has nothing to build
without one. Appendix C includes a working minimal example.

> **The packaging failure you'll hit first.** With both `app/` and `tests/` present and no explicit
> package configuration, setuptools finds two top-level packages and refuses to guess, failing with
> "Multiple top-level packages discovered". Fix it by declaring what to package:
>
> ```toml
> [tool.setuptools.packages.find]
> include = ["app*"]
> ```

> **`python -m build` needs network access by default.** It creates an isolated build environment
> and downloads the backends listed in `[build-system] requires` (setuptools, wheel). On an
> air-gapped agent, use `--no-isolation` with those packages pre-installed in the venv.

`archiveArtifacts` copies files back to the **controller** so they survive the agent being
destroyed. `fingerprint: true` records a checksum, letting Jenkins tell you later which build
produced a given file.

> Don't archive gigabytes. Archived artifacts live in `JENKINS_HOME` forever unless you set
> `buildDiscarder`. For anything large, push to a real artifact repository and archive only the
> metadata.

### Stage 3 (optional) — Build a Docker image

```groovy
        stage('Build image') {
            steps {
                script {
                    def image = docker.build("my-python-app:${env.BUILD_NUMBER}")
                    image.tag('latest')
                }
            }
        }
```

**Prerequisites:** the **Docker Pipeline** plugin (not the Docker *cloud* plugin), plus a working
`docker` CLI **on the agent** with permission to reach a daemon. Stock `jenkins/inbound-agent`
images do **not** contain a `docker` CLI. If you don't have both, delete this stage rather than
debugging it — nothing else depends on it.

Tag with `BUILD_NUMBER` (or the Git SHA), never only `latest`. `latest` tells you nothing about
which build produced a running container.

### Stage 4 — Smoke test

```groovy
        stage('Smoke test') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    PORT=$((8000 + ${EXECUTOR_NUMBER:-0}))
                    "$VENV/bin/python" -m app.main --port "$PORT" &
                    APP_PID=$!
                    trap 'kill "$APP_PID" 2>/dev/null || true' EXIT

                    for _ in $(seq 1 20); do
                        if curl -fsS "http://127.0.0.1:${PORT}/health"; then
                            echo
                            echo "app is up on port ${PORT}"
                            exit 0
                        fi
                        sleep 1
                    done

                    echo "app failed to start" >&2
                    exit 1
                '''
            }
        }
```

Start it, poll until healthy, and **kill it in the same step**. The `trap ... EXIT` guarantees
cleanup even when the step fails.

Two things worth understanding rather than copying blindly:

- **Use `if ... then ... fi`, not `curl ... && exit 0`.** Both happen to work under `set -e`, but
  only because of a subtle exemption for the left-hand side of `&&`. The `if` form needs no such
  reasoning. (The first curl attempt normally fails and prints a connection error to stderr — the
  loop is doing its job. That's expected, not a problem.)
- **A hardcoded port collides.** Two concurrent builds on the same agent both binding 8000 will
  fight. Derive it from `EXECUTOR_NUMBER`, or set the agent to 1 executor.

> **Why this deserves its own section:** the naive version — start the app in the background and
> move on — produces a build that mysteriously "passes" while the app never started, and leaves a
> stray process that Jenkins' ProcessTreeKiller then reaps at the end of the build, confusing
> everyone. Start, verify, and stop in one step.

### The complete pipeline

The full assembled Jenkinsfile, with lint and test running in parallel, is in
[Appendix C](#jenkinsfile). It's the file you should actually commit.

> **`skipDefaultCheckout(true)` + explicit `cleanWs()` + `checkout scm`** gives you full control
> over when the workspace is cleaned and when the code arrives. Without it, Jenkins checks out
> before your first stage and `cleanWs()` would delete the code it just fetched.

---

## 18. Reading build output and reports

### Console Output

The primary artifact of every build. Two links next to each entry in Build History: the **build
number** goes to the build page; the **status icon** goes straight to console output.

Remember from Section 8: lines starting with `+` are the commands, the following lines are their
output. That's why things appear "twice".

For long logs, `http://<jenkins>/job/<name>/<n>/consoleText` gives you the raw text — greppable,
and much faster than the rendered page.

### Workspace browser

The **Workspace** link lets you browse the files the build produced. Useful for confirming a build
actually generated what you expected. On ephemeral Docker agents the workspace is gone once the
container is destroyed — which is exactly why you `archiveArtifacts`.

### Test result reporting

**JUnit XML** is the lingua franca. Most Python tools emit it:

```bash
pytest --junitxml=reports/junit-tests.xml
ruff check . --output-format=junit --output-file=reports/junit-lint.xml
```

```groovy
post {
    always {
        junit allowEmptyResults: true, testResults: 'reports/junit-*.xml'
    }
}
```

Jenkins parses these into pass/fail tables, per-test history, and trend graphs. When it finds
failed tests it marks the build **UNSTABLE** (yellow) rather than FAILED — a distinction worth
preserving, because it separates "your code is broken" from "the pipeline is broken."

> **But only if the stage didn't already fail** — see the red/yellow discussion in Section 17. This
> surprises nearly everyone the first time.

### Coverage

With the **Coverage** plugin (`recordCoverage`; the old standalone Cobertura plugin is deprecated):

```bash
pytest --cov=app --cov-report=xml:reports/coverage.xml
```

```groovy
recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
```

`pytest-cov` writes Cobertura-format XML, which the plugin parses into per-file coverage and trend
graphs. Optional quality gates can mark the build unstable below a threshold:

```groovy
recordCoverage(
    tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']],
    qualityGates: [[threshold: 15.0, metric: 'LINE', baseline: 'PROJECT', unstable: true]]
)
```

> **⚠️ Pick a threshold that matches your project.** The Appendix C sample scores about **19%**,
> because `app/main.py` (the HTTP server) has no unit tests — only the smoke test exercises it. A
> `threshold: 60.0` would turn a perfectly correct first build yellow and send you hunting for a
> bug that isn't there. Start below your current number, then ratchet it up.

### Pipeline visualization

**Stage View** is built in (part of the suggested plugin set): stages as columns, runs as rows,
per-stage duration, hover for logs. Enough for most people.

**Pipeline Graph View** is the recommended addition — a modern graph rendering inside the normal
Jenkins UI that handles parallel and nested stages far better. Install via **Manage Jenkins →
Plugins → Available**.

> **Blue Ocean is not the answer any more.** Older tutorials — including most of the popular video
> ones — send you to Blue Ocean. It has been in maintenance-only status for years and is no longer
> recommended for new installations. Pipeline Graph View is its replacement.

### Notifications

For pipelines, notifications go in the `post` block. Always include `${BUILD_URL}` so the recipient
can click through. The built-in `mail` step needs SMTP configured under **Manage Jenkins →
System**; without it the step itself fails and turns an informative failure into a confusing one.

> **Notify on state change, not on every build.** A channel that gets a message for all 200 green
> builds a day is a channel nobody reads. Consider `post { unsuccessful { ... } }` plus a
> `post { fixed { ... } }` "recovered" notification.

---

## 19. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| First build fails immediately with a Git error | SCM section opened but repository URL left blank | Set **Source Code Management** to **None** |
| `python: command not found` | Image ships `python3`, no `python` alias | Use `python3`; probe with a one-line job first |
| `ensurepip is not available` from `python3 -m venv` | The `venv` module is packaged separately (Debian: `python3-venv`) | Install it in the agent image; probe the image directly |
| `pip: command not found` inside a venv step | Each `sh` step is a new shell — the previous `activate` is gone | Call `"$VENV/bin/pip"` explicitly, or set `PATH` in `environment` |
| `ModuleNotFoundError: No module named 'app'` | The venv never installed your project, so it isn't on `sys.path` | Set `PYTHONPATH = "${WORKSPACE}"`, add `tests/__init__.py`, or `pip install -e .` |
| `Multiple top-level packages discovered` | setuptools found both `app` and `tests` and won't guess | Add `[tool.setuptools.packages.find]` with `include = ["app*"]` |
| `pip install -r requirements.txt` fails, file not found | The file doesn't exist — an empty one is still required | Commit an empty `requirements.txt` with a comment in it |
| `python -m build` fails with a network error | Build isolation downloads setuptools/wheel | Allow network, or use `--no-isolation` with them pre-installed |
| Build is green but tested nothing | `pytest` exited 5 (no tests collected) and nothing checked | Add `set -euo pipefail`; verify the test path; set `allowEmptyResults: false` |
| Tests failed but the build is red, not yellow | The stage failed before `junit` could downgrade it | Wrap the test step in `catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE')` |
| Coverage gate fails on a correct build | Threshold set higher than the project's actual coverage | Lower the threshold to match reality, then ratchet up |
| `junit` reports a malformed/empty report | `tool > file` truncated the file before the tool crashed | Use the tool's native `--output-file` flag |
| Agent runs but a tool is missing | Stock agent images contain only Java and the remoting jar | Build a custom agent image with your toolchain baked in |
| `Tool type "python" does not have an install...` | There is no Python tool installer | Use an agent image, a Docker agent, or a venv |
| Docker cloud can't pull your locally built image | Default pull strategy tries the registry | Set the template's **Pull strategy** to **Never pull**, or push to a registry |
| Label "doesn't exist" warning despite existing | Trailing space from autocomplete, or label index not yet populated | Remove the space; run the job once |
| Build queued forever waiting for an agent | Dynamic agent still provisioning, or instance cap reached | **Wait.** Then check the cloud's instance cap and the Docker daemon |
| Docker cloud connection fails | Jenkins container can't reach the Docker daemon | Mount the socket into the Jenkins container |
| `permission denied` on `/var/run/docker.sock` | Agent user isn't in the docker group | `--group-add` the host's docker GID, or use a remote daemon over TLS |
| A stage is always skipped and never errors | `when { branch 'main' }` in a plain Pipeline job — `BRANCH_NAME` is unset | Use a Multibranch Pipeline, or gate on `env.GIT_BRANCH` |
| `Bad substitution` in a shell step | The step ran under `/bin/sh`, not bash | Put `#!/usr/bin/env bash` as the very first characters inside `'''` |
| Files from a previous build present in workspace | Workspaces persist by default | **Delete workspace before build starts** / `cleanWs()` |
| Two concurrent builds fight over a port or device | Shared agent, hardcoded resource | Derive from `EXECUTOR_NUMBER`, use `lock()`, or set the node to 1 executor |
| Downstream stages fail despite correct syntax | An upstream stage failed | Fix the upstream stage first |
| Console output appears duplicated | The `+` line is the command; the next is its output | Not a bug |
| Pipeline changes have no effect | Editing the wrong file when using Pipeline-from-SCM | The Jenkinsfile **in Git** is the source of truth — commit and push |
| Unexpected "Declarative: Checkout SCM" stage | Pipeline-from-SCM auto-adds the clone | Expected behavior |
| Stage View history disappeared | A stage was added, so the visualization re-baselines | Cosmetic; builds still exist |
| `triggers {}` in the Jenkinsfile never fires | Jenkins hasn't read the file yet | Run the job manually once |
| Webhook fires but no build starts | The job's Git URL doesn't match the pushed repository | Make the SCM URL and the webhook repository the same |
| Scheduled job runs far less often than expected | `H * * * *` is **hourly**, not per-minute | Use `H/5 * * * *` for every 5 minutes |
| A secret appeared in the console log | Groovy double-quote interpolation baked it into the command | Use `withCredentials` + single-quoted `sh` |
| Background process dies when the build ends | ProcessTreeKiller cleans up spawned processes | Start/verify/stop in one step |
| Build hangs forever | No timeout configured | `options { timeout(time: 30, unit: 'MINUTES') }` |
| Controller disk filling up | Unbounded build history and archived artifacts | `buildDiscarder(logRotator(...))` on every job |

### General technique

1. **Get into the filesystem.** `docker exec -it <container> bash` (or `kubectl exec`) and look.
   Often faster than reasoning through the UI.
2. **Read the error, then search it.** Click **Show Details** on connection failures, take the
   exact text, search it. This is genuinely how the work gets done.
3. **Reproduce the step by hand on the agent.** If `sh 'pytest ...'` fails, run that exact command
   in the agent's shell, in the workspace directory. Nine times out of ten the difference is an
   environment variable or `PATH`.
4. **Probe the image, not the pipeline.** `docker run --rm --entrypoint sh <image> -c '<command>'`
   answers "is the tool there?" in two seconds instead of a four-minute build.
5. **Check `Manage Jenkins → System Log`** for controller-side errors that never reach a build log.
6. **Bisect the pipeline.** Comment out stages until it passes, then add them back one at a time.

---

## 20. Practices worth adopting

**Pipeline design**

- **Keep the Jenkinsfile in the repository it builds.** Same review process, same history.
- **Keep it thin.** Jenkinsfiles should call scripts (`./scripts/test.sh`), not contain the logic.
  Then developers can run the exact same script locally, instead of debugging CI logic through a
  four-minute feedback loop.
- **Make builds reproducible.** Pin agent image tags; pin dependency versions with `==`.
- **Fail fast.** Put lint and unit tests before anything slow.
- **Set a timeout on every pipeline** and **`buildDiscarder` on every job.**
- **Publish test results in `post { always }`.**
- **Decide your red/yellow policy explicitly**, and write it down as a comment in the Jenkinsfile.

**Agents**

- **Never build on the controller.** Set the built-in node to 0 executors.
- **Maintain your own agent images** with your toolchain baked in, and version them.
- **One executor for agents with attached hardware**, so two builds can't fight over a device.
- **Prefer ephemeral agents.** A clean container per build eliminates an entire class of "works on
  agent 3 but not agent 5" problems.

**Security**

- **Store secrets in the credential store**, never in job config, `environment` blocks, or repos.
- **Use `withCredentials` and single-quoted shell** so secrets aren't interpolated into command
  lines and logged.
- **Scope credentials to folders** rather than globally.
- **Restrict Script Console access** — it's equivalent to root on the controller.
- **Keep plugins updated.** Most Jenkins CVEs are in plugins, not core.

**Naming**

- **No spaces in job names** — they become directory names.
- **Descriptive labels that name a capability** (`python3`, `hw-nvme-tester`, `windows-msvc`), not
  a machine (`server-04`). Machines get replaced; capabilities persist.

---

## Appendix A — End-to-end walkthrough

The shortest path from nothing to a green pipeline, with a verification step for each link.

### 1. Start Jenkins (Section 5)

```bash
docker network create jenkins

docker run -d --name jenkins \
  --network jenkins \
  --restart unless-stopped \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21

docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open `http://localhost:8080`, paste the password, install suggested plugins, create your admin user.

### 2. Confirm what the controller has (Section 10)

Create a Freestyle job with one **Execute shell** step and run it:

```bash
echo "--- toolchain probe ---"
java -version 2>&1
git --version
which python3 || echo "python3: NOT INSTALLED"
python3 -m venv /tmp/probe 2>&1 && echo "venv: OK" || echo "venv: NOT AVAILABLE"
```

On a stock `jenkins/jenkins:lts-jdk21` you get Java and Git, and **no Python**. That's expected —
the controller image is a Jenkins controller, not a build environment.

### 3. Pick how you'll get Python

| Approach | Effort | Use when |
|---|---|---|
| **A. Docker agent image** (Section 11) | Highest | You want the realistic setup |
| **B. `agent { docker { image 'python:3.13-slim' } }`** | Medium | You have the Docker Pipeline plugin and a reachable daemon |
| **C. Install into the controller** | Lowest | You just want to see the pipeline run today |

Option C, **for learning only** — this is lost whenever the container is recreated, because it
writes to the container layer rather than the `jenkins_home` volume:

```bash
docker exec -u root jenkins bash -c \
  'apt-get update && apt-get install -y --no-install-recommends python3 python3-venv python3-pip && rm -rf /var/lib/apt/lists/*'
```

Re-run the probe job to confirm. If you take this path, use `agent any` in place of
`agent { label 'docker-agent-python' }` everywhere.

### 4. Create the sample repository (Appendix C)

```bash
git init
git add .
git commit -m "Initial commit: sample Python project with Jenkinsfile"
git branch -M main
git remote add origin https://github.com/<you>/jenkins-sample.git
git push -u origin main
```

**Before pushing, run it locally.** Always confirm the project works outside Jenkins before blaming
Jenkins — this is far faster than a four-minute build cycle:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt

ruff check .
PYTHONPATH=. pytest --cov=app
python -m build --wheel --outdir dist/

PYTHONPATH=. python -m app.main --port 8000 &
curl -fsS http://127.0.0.1:8000/health   # → {"status": "ok", "version": "0.1.0"}
kill %1
```

If any of these fails locally, it will fail in Jenkins for the same reason.

### 5. Create the pipeline job (Section 15)

**New Item** → `jenkins-sample` → **Pipeline** → OK. Then:

- **Definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** your repository
- **Branch Specifier:** `*/main`
- **Script Path:** `Jenkinsfile`

Save → **Build Now**.

### 6. Verify each link in the chain

Don't just look for green. Confirm each capability independently:

| Check | Where | Passing looks like |
|---|---|---|
| Checkout worked | Console Output | `Obtained Jenkinsfile from git`, then the `git log -1` line |
| Environment built | Console Output | `pip list` shows pytest, ruff, coverage |
| Lint ran | **Test Result** | A `ruff` suite appears |
| Tests ran | **Test Result** | 3 tests, 3 passed |
| Coverage parsed | **Coverage** on the build page | ~19% and a per-file table |
| Artifact archived | **Build Artifacts** | `jenkins_sample-0.1.0-py3-none-any.whl` downloadable |
| Smoke test passed | Console Output | `app is up on port 8000` |
| Stage view renders | Job page | Checkout → Set up → Checks (parallel) → Package → Smoke test |

### 7. Prove the failure paths too

A pipeline that has only ever passed is untested. Verify it reports failure correctly:

```bash
# 1. Break a test → build should go red (or yellow, if you enabled catchError)
sed -i 's/assert add(2, 3) == 5/assert add(2, 3) == 6/' tests/test_calc.py
git commit -am "break a test" && git push

# 2. Break lint → Lint stage fails, Test still runs (parallel branches are independent)
echo "import os" >> app/calc.py
git commit -am "unused import" && git push

# 3. Restore
git revert --no-edit HEAD HEAD~1 && git push
```

Confirm each shows up where you'd expect: the failing test in **Test Result** with its assertion
message (`assert 5 == 6`), the lint violation as a `ruff` test case. **A CI system you haven't seen
fail is a CI system you don't yet trust.**

---

## Appendix B — Command and URL reference

### Docker lifecycle

```bash
docker ps                                    # running containers
docker logs -f jenkins                       # follow controller startup
docker exec -it jenkins bash                 # shell as the jenkins user
docker exec -it -u root jenkins bash         # shell as root (installing packages)
docker restart jenkins                       # restart
```

### Jenkins paths (Docker image)

| Path | Contains |
|---|---|
| `/var/jenkins_home/` | Everything Jenkins persists |
| `/var/jenkins_home/workspace/<job>/` | Per-job workspace (built-in node) |
| `/var/jenkins_home/jobs/<job>/config.xml` | Job configuration |
| `/var/jenkins_home/jobs/<job>/builds/` | Build history and console logs |
| `/var/jenkins_home/secrets/` | Master key + initial admin password |
| `/var/jenkins_home/plugins/` | Installed plugins |
| `/var/jenkins_home/logs/` | Controller logs |

> Native `.deb`/`.rpm` installs use `/var/lib/jenkins/`. Agents use their own remote root. Never
> hardcode a controller path into an agent step — use `${WORKSPACE}`.

### Back up the volume

```bash
docker run --rm \
  -v jenkins_home:/source:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/jenkins-backup-$(date +%F).tar.gz -C /source .
```

### Useful URLs

| URL | Purpose |
|---|---|
| `/env-vars.html` | Every environment variable your instance injects |
| `/job/<name>/<n>/consoleText` | Raw console log — greppable |
| `/job/<name>/<n>/api/json?pretty=true` | Build metadata as JSON |
| `/manage/systemInfo` | Versions, JVM, system properties |
| `/manage/log` | Controller-side errors that never reach a build log |
| `/pipeline-syntax/` | **Snippet Generator** — builds step syntax from a form |
| `/pipeline-model-converter/validate` | Validate a Jenkinsfile over HTTP |

> `/pipeline-syntax/` is the single most useful page for writing pipelines. It generates correct
> syntax for any installed step, including plugin steps whose parameters you'd otherwise guess at.

### Minimal Jenkinsfile

```groovy
pipeline {
    agent any
    options { timeout(time: 15, unit: 'MINUTES') }
    stages {
        stage('Build') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    echo "building"
                '''
            }
        }
    }
}
```

---

## Appendix C — The sample project

A complete, working Python project. Create these files, commit, push, and point a Pipeline job at
it. Deliberately small, but it exercises every mechanism in Section 17: dependencies, linting,
tests, coverage, packaging, and a health endpoint to smoke-test.

> **Verified July 24, 2026** on Python 3.12 with the pinned versions below: 3 tests pass, ruff
> reports no violations, coverage lands at ~19%, and `jenkins_sample-0.1.0-py3-none-any.whl`
> builds. The failure paths in Appendix A step 7 were also confirmed — ruff exits 1 on the unused
> import, pytest exits 1 on the broken assertion, and pytest exits 5 when it collects no tests.

```
jenkins-sample/
├── Jenkinsfile
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── .gitignore
├── app/
│   ├── __init__.py
│   ├── calc.py
│   └── main.py
└── tests/
    ├── __init__.py
    └── test_calc.py
```

### `app/__init__.py`

```python
"""Sample application package."""

__version__ = "0.1.0"
```

### `app/calc.py`

```python
"""Arithmetic helpers — deliberately trivial, so the tests are about CI, not math."""


def add(a: float, b: float) -> float:
    """Return the sum of a and b."""
    return a + b


def divide(a: float, b: float) -> float:
    """Return a divided by b.

    Raises:
        ZeroDivisionError: if b is zero.
    """
    if b == 0:
        raise ZeroDivisionError("division by zero")
    return a / b
```

### `app/main.py`

A standard-library HTTP server — no framework, so there is nothing extra to install.

```python
"""Minimal HTTP server exposing a health endpoint, for pipeline smoke tests."""

import argparse
import json
from http.server import BaseHTTPRequestHandler, HTTPServer

from app import __version__
from app.calc import add


class Handler(BaseHTTPRequestHandler):
    """Handles /health and /add."""

    def do_GET(self) -> None:  # noqa: N802  (name mandated by BaseHTTPRequestHandler)
        if self.path == "/health":
            self._send(200, {"status": "ok", "version": __version__})
        elif self.path.startswith("/add"):
            self._send(200, {"result": add(2, 3)})
        else:
            self._send(404, {"error": "not found"})

    def log_message(self, fmt: str, *args: object) -> None:
        """Silence per-request logging so build logs stay readable."""
        return

    def _send(self, code: int, payload: dict) -> None:
        body = json.dumps(payload).encode()
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--port", type=int, default=8000)
    args = parser.parse_args()

    server = HTTPServer(("127.0.0.1", args.port), Handler)
    print(f"listening on 127.0.0.1:{args.port}", flush=True)
    server.serve_forever()


if __name__ == "__main__":
    main()
```

> **`flush=True` matters.** Python buffers stdout when it isn't a terminal. Without it, your
> startup line may never appear in the build log, making a working app look like a hung one.

### `tests/__init__.py`

Empty file. It makes `tests` a package, which avoids import-path surprises — see the `PYTHONPATH`
note in Section 17.

### `tests/test_calc.py`

```python
"""Tests for app.calc."""

import pytest

from app.calc import add, divide


def test_add() -> None:
    assert add(2, 3) == 5


def test_divide() -> None:
    assert divide(10, 2) == 5


def test_divide_by_zero() -> None:
    with pytest.raises(ZeroDivisionError):
        divide(1, 0)
```

### `requirements.txt`

```
# Runtime dependencies. The sample app uses only the standard library,
# so this is intentionally empty — but the file must exist, because the
# pipeline installs from it.
```

> Keep the file even when empty. `pip install -r requirements.txt` fails on a missing file, and a
> stage that fails for that reason is a confusing first experience.

### `requirements-dev.txt`

```
pytest==9.1.1
pytest-cov==7.1.0
ruff==0.16.0
build==1.5.0
```

> **Pinned on purpose, and these four were installed and run together on 2026-07-24.** Unpinned dev
> dependencies mean a tool update can turn your build red overnight with no change from you. Pin,
> then upgrade deliberately.
>
> These versions require **Python 3.10 or newer** (pytest 9, `build`, and coverage.py all dropped
> 3.9). If your agent only has Python 3.9, pin the older lines instead: `pytest==8.3.5`,
> `pytest-cov==6.2.1`.

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "jenkins-sample"
version = "0.1.0"
description = "Sample project for a Jenkins CI pipeline"
requires-python = ">=3.10"
dependencies = []

[tool.setuptools.packages.find]
include = ["app*"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[tool.ruff]
line-length = 100
target-version = "py310"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]

[tool.coverage.run]
source = ["app"]
omit = ["*/tests/*"]
```

> `[tool.setuptools.packages.find]` with `include = ["app*"]` stops setuptools from also packaging
> `tests` into the wheel. Without it, `python -m build` fails with a "multiple top-level packages"
> error — a common and confusing first failure.

### `.gitignore`

```gitignore
__pycache__/
*.py[cod]
.venv/
venv/
dist/
build/
*.egg-info/
reports/
.coverage
.coverage.*
coverage.xml
.pytest_cache/
.ruff_cache/
```

> `reports/` and `dist/` are build outputs. Jenkins creates them per build and publishes them via
> `junit` and `archiveArtifacts` — they should never be committed. Listing `build/` also keeps
> `ruff check .` from linting the copy of your source that `python -m build` leaves behind, since
> ruff respects `.gitignore` inside a Git repository.

### `Jenkinsfile`

```groovy
pipeline {
    // Using a Docker agent (Section 11). If Python is on the controller
    // instead, change this to: agent any
    agent { label 'docker-agent-python' }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        PYTHONDONTWRITEBYTECODE       = '1'
        VENV                          = "${WORKSPACE}/.venv"
        // Belt and braces so `import app` resolves when pytest runs from the workspace root
        PYTHONPATH                    = "${WORKSPACE}"
    }

    // Policy: a failing lint or test fails the build (red). To make tests
    // report UNSTABLE (yellow) instead, wrap the pytest sh in catchError.

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Set up environment') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    python3 -m venv "$VENV"
                    "$VENV/bin/python" -m pip install --upgrade pip
                    "$VENV/bin/pip" install -r requirements.txt -r requirements-dev.txt

                    "$VENV/bin/python" --version
                    "$VENV/bin/pip" list
                '''
            }
        }

        stage('Checks') {
            parallel {
                stage('Lint') {
                    steps {
                        sh '''#!/usr/bin/env bash
                            set -euo pipefail
                            mkdir -p reports
                            "$VENV/bin/ruff" check . \
                                --output-format=junit \
                                --output-file=reports/junit-lint.xml
                        '''
                    }
                }
                stage('Test') {
                    steps {
                        sh '''#!/usr/bin/env bash
                            set -euo pipefail
                            mkdir -p reports
                            "$VENV/bin/pytest" \
                                --junitxml=reports/junit-tests.xml \
                                --cov=app \
                                --cov-report=xml:reports/coverage.xml \
                                --cov-report=term
                        '''
                    }
                }
            }
        }

        stage('Package') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    "$VENV/bin/pip" install build
                    "$VENV/bin/python" -m build --wheel --outdir dist/
                    ls -l dist/
                '''
            }
        }

        stage('Smoke test') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    PORT=$((8000 + ${EXECUTOR_NUMBER:-0}))
                    "$VENV/bin/python" -m app.main --port "$PORT" &
                    APP_PID=$!
                    trap 'kill "$APP_PID" 2>/dev/null || true' EXIT

                    for _ in $(seq 1 20); do
                        if curl -fsS "http://127.0.0.1:${PORT}/health"; then
                            echo
                            echo "app is up on port ${PORT}"
                            exit 0
                        fi
                        sleep 1
                    done

                    echo "app failed to start" >&2
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'reports/junit-*.xml'
            archiveArtifacts artifacts: 'dist/*.whl', allowEmptyArchive: true, fingerprint: true
            // Requires the Coverage plugin; delete this line if not installed.
            // No quality gate here on purpose — this project sits at ~19%.
            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
        }
        failure {
            echo "Build failed: ${env.BUILD_URL}"
        }
        cleanup {
            cleanWs()
        }
    }
}
```

### Adapting it

| If you… | Change |
|---|---|
| Have Python on the controller, no agents | `agent { label '...' }` → `agent any` |
| Don't have the Coverage plugin | Delete the `recordCoverage` line |
| Want failing tests yellow, not red | Wrap the pytest `sh` in `catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE')` |
| Want to add a Docker image build | See Section 17, Stage 3 — needs the Docker Pipeline plugin and a `docker` CLI on the agent |
| Use `main` vs `master` | Set **Branch Specifier** to match |
| Are on Python 3.9 | Pin `pytest==8.3.5`, `pytest-cov==6.2.1`, and set `requires-python = ">=3.9"` |

---
