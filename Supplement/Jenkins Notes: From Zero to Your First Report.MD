# Jenkins: From Zero to Your First Pipeline

---

## Table of Contents

1. [What Problem Does Jenkins Actually Solve?](#1-what-problem-does-jenkins-actually-solve)
2. [Jenkins Architecture: Controller and Agents](#2-jenkins-architecture-controller-and-agents)
3. [The Two Main Job Types](#3-the-two-main-job-types)
4. [Vocabulary Reference](#4-vocabulary-reference)
5. [Installing Jenkins with Docker](#5-installing-jenkins-with-docker)
6. [First Login and the Web GUI Tour](#6-first-login-and-the-web-gui-tour)
7. [Manage Jenkins: What Admins Actually Use](#7-manage-jenkins-what-admins-actually-use)
8. [Your First Freestyle Job](#8-your-first-freestyle-job)
9. [Environment Variables and the Workspace](#9-environment-variables-and-the-workspace)
10. [Exploring the Jenkins Filesystem](#10-exploring-the-jenkins-filesystem)
11. [Freestyle Job: Cloning Git and Running Python](#11-freestyle-job-cloning-git-and-running-python)
12. [Setting Up Agents](#12-setting-up-agents)
13. [Using Labels to Route Jobs to Agents](#13-using-labels-to-route-jobs-to-agents)
14. [Automatic Build Triggers](#14-automatic-build-triggers)
15. [Anatomy of a Jenkinsfile](#15-anatomy-of-a-jenkinsfile)
16. [Your First Declarative Pipeline](#16-your-first-declarative-pipeline)
17. [Moving the Pipeline into Source Control](#17-moving-the-pipeline-into-source-control)
18. [Multi-Stage Pipelines](#18-multi-stage-pipelines)
19. [A Real Python CI Pipeline](#19-a-real-python-ci-pipeline)
20. [Pipeline Visualization](#20-pipeline-visualization)
21. [Reading Build Output and Reports](#21-reading-build-output-and-reports)
22. [Troubleshooting Guide](#22-troubleshooting-guide)
23. [Best Practices](#23-best-practices)

**Appendices**

- [Appendix A — End-to-end walkthrough](#appendix-a--end-to-end-walkthrough)
- [Appendix B — Command reference](#appendix-b--command-reference)
- [Appendix C — The sample project](#appendix-c--the-sample-project)

---

## 1. What Problem Does Jenkins Actually Solve?

Imagine you're on a team of five developers, all writing code for the same product. Every day, everyone makes changes and combines their work.

Without automation, here's what usually goes wrong:
- Someone's code works on their own laptop, but breaks the moment it's combined with everyone else's.
- Nobody notices a bug until days later, right before the product is supposed to ship.
- Every release involves someone manually running a checklist of commands — easy to forget a step.
- Feedback ("did my change break anything?") takes hours or days instead of minutes.

### Why this beats doing it manually

Consider running a Python script on a server. Without Jenkins, someone has to SSH into the box, have the right permissions, and run it by hand. With Jenkins you click one button — and you additionally get:
- **A full history** of every time that script ran
- **Complete log output** for each run
- **Automatic triggers** (webhook, SCM change, or cron-like schedule)
- **Access control** — the operator needs Jenkins permission, not a shell account on the server

> **Term Explained — "CI/CD"**
> - **CI = Continuous Integration**: every code change is automatically combined with everyone else's and tested, so problems surface in minutes.
> - **CD = Continuous Delivery** (packaged and ready, human clicks deploy) or **Continuous
>   Deployment** (fully automatic release).
>
> Jenkins is the tool that *runs* the automation; CI/CD is the *practice*.

> **Extra Knowledge — Why Jenkins, when GitHub Actions exists?**
> Jenkins' advantage is flexibility — self-hosted, endlessly extensible via plugins, and able to
> reach machines that will never touch the public internet. The trade-off is that *you* install,
> maintain, patch, and secure it. Teams with air-gapped labs, physical test hardware, or strict
> compliance requirements often stay on Jenkins for exactly this reason.

---

## 2. Jenkins Architecture: Controller and Agents

**Two pieces:**
1. **The controller** — hosts the web UI, stores configuration and build history, schedules builds out to agents.
2. **The agents** — run the actual build inside their workspace.

### A typical workflow

```
Developer pushes a commit to a Git repository
        ↓
Jenkins controller becomes aware of the commit (webhook or polling)
        ↓
Controller triggers the appropriate pipeline
        ↓
Controller selects an agent BASED ON LABELS
        ↓
Agent runs the build (usually a set of shell commands)
        ↓
Test, build, package, publish the result
```

Label-based selection is the key mechanism — you tag agents with labels, and jobs request agents by label. We set this up hands-on in Section 13.

### The two categories of agent

**Permanent (static) agents**

Think of these as standalone Linux or Windows servers configured to run Jenkins jobs — ordinary everyday servers. 

**Cloud (dynamic) agents**

More common in modern environments. Examples: **Docker, Kubernetes, AWS EC2 Fleet**.
Jenkins **spins up an agent on demand** from a template you configure, runs the build, and
disposes of it. No fleet of always-on servers to patch, and every build starts from a clean image.

### How an agent connects to the controller

Whichever category it is, the agent process and the controller have to establish a connection, and
the direction of that connection matters for firewalls. Two launch methods:

| Launch method | Direction | Notes |
|---|---|---|
| **SSH** (*Launch agents via SSH*) | Controller → agent | Controller SSHes in and starts the agent process. Needs an SSH credential and Java on the agent. Simplest for permanent Linux agents. |
| **Inbound** (*Launch agent by connecting it to the controller*) | Agent → controller | The agent dials home. Use when the controller cannot reach the agent (NAT, firewall, DMZ). Transport is either **TCP** on the inbound port (default 50000, hence `-p 50000:50000` in Section 5) or **WebSocket** over the normal HTTP port, which needs no extra port at all. |

Cloud agents almost always use the inbound method, which is why the standard cloud agent image is
called `jenkins/inbound-agent`.

> **Notice — Building on the built-in node**
> When you first install Jenkins, you'll see a warning that building on the built-in node can be a security risk. This means you have no agents configured, so any pipeline would run directly on the Jenkins controller — **which is not recommended.** It's safe to ignore during initial learning, but set up agents before doing anything real.
>
> Before anything real: create agents, then set the built-in node's **executors to 0** (Manage Jenkins → Nodes → Built-In Node → Configure).

---

## 3. The Two Main Job Types

### Freestyle projects

The simplest way to get started. **Think of them as shell scripts that run on a server**,
triggerable by events such as a push to a Git repository.

Everything is configured through the UI, and most Freestyle projects just execute a shell script. If you're comfortable with the Linux command line, you're set.

Freestyle projects are still perfectly reasonable for simple, single-step automation. They become painful when logic grows, because none of it is version-controlled.

### Pipelines

Pipelines use a **`Jenkinsfile`, written in a Groovy-based DSL**, to define the build as code.
They're broken into stages. A common five-stage shape:

| Stage        | What happens                                                    |
| ------------ | --------------------------------------------------------------- |
| **Checkout** | Pull code from Git, set up the environment on the agent         |
| **Build**    | Compile or assemble — a wheel, an executable, a container image |
| **Test**     | Run tests against what you just built                           |
| **Package**  | Produce the deployable artifact                                 |
| **Deploy**   | Push artifacts to a registry or environment                     |

> **Declarative is what you want.** Jenkins supports two pipeline syntaxes: *declarative* (starts
> with `pipeline { ... }`) and *scripted* (starts with `node { ... }`). Declarative is the modern
> default and everything in this document uses it.

### Other item types in the "New Item" menu

| Type | Purpose |
| --- | --- |
| **Freestyle project** | UI-configured single job. Combines any SCM with any build system; also usable for non-software tasks. |
| **Pipeline** | A single pipeline defined by a Jenkinsfile. Where you'll spend most of your time. |
| **Multi-configuration project** | Also called a **matrix** project. Runs *the same* job once per combination of configuration axes — e.g. 3 OS values × 2 Python versions = 6 runs — and shows the results as a grid. Use it for compatibility matrices, not for deployment stages. (In pipelines, the modern equivalent is the declarative `matrix` directive.) |
| **Folder** | A container for grouping and filtering items. Also a namespace for credentials. |
| **Multibranch Pipeline** | Automatically creates a pipeline for **every branch and PR** in *one* repository that contains a Jenkinsfile. |
| **Organization Folder** | Scans an **entire GitHub/GitLab/Bitbucket organization**, and creates a Multibranch Pipeline for every repository that contains a Jenkinsfile. |

---

## 4. Vocabulary Reference

| Term                     | Plain-English meaning                                                       |
| ------------------------ | --------------------------------------------------------------------------- |
| **Job / Project / Item** | A task you've told Jenkins to do. Like a saved recipe.                      |
| **Build / Run**          | One execution of a job. Numbered #1, #2, #3…                                |
| **Controller**           | The Jenkins brain — schedules work, stores config, hosts the UI             |
| **Agent / Node**         | The machine that actually runs the build                                    |
| **Built-in node**        | The controller acting as an agent. Should have 0 executors in production.   |
| **Executor**             | A slot on a node that runs one build at a time                              |
| **Workspace**            | The per-job directory where Jenkins checks out code and works               |
| **Label**                | A tag on an agent; jobs use labels to request a kind of agent               |
| **Pipeline**             | A job defined as code in a Jenkinsfile                                      |
| **Jenkinsfile**          | The text file containing the pipeline definition                            |
| **Stage**                | A labeled phase inside a pipeline                                           |
| **Step**                 | A single unit of work inside a stage (`sh`, `echo`, `junit`, …)             |
| **Plugin**               | An add-on extending Jenkins' abilities                                      |
| **Artifact**             | A file a build produces that's worth keeping                                |
| **Trigger**              | What starts a build — button, schedule, webhook, or upstream job            |
| **SCM**                  | Source Control Management (Git, Subversion, …)                              |
| **Credential**           | A stored secret (token, SSH key, password) referenced by ID, never by value |
| **Console Output**       | The full text log of a build                                                |
| **JENKINS_HOME**         | The controller's data directory — config, jobs, build history, plugins      |

---

## 5. Installing Jenkins with Docker

> **Term Explained — "Docker" / "container"**
> Docker runs software inside an isolated bundle called a **container** — a lightweight,
> disposable mini-computer with everything pre-installed. Rather than installing Java, Jenkins, and dependencies on your real machine, you tell Docker to run the Jenkins image. Easy to delete cleanly afterward.

**Prerequisite:** Docker Desktop or Docker Engine.

### Create a network first

Later sections need the Jenkins container and a helper container to talk to each other by name. Create a user-defined bridge network now so you don't have to redo the install:

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
| `-d` | **Detached**: run in the background, terminal stays free |
| `--name jenkins` | Nickname the container so you can refer to it later |
| `--network jenkins` | Attach to the user-defined network created above |
| `--restart unless-stopped` | Restart automatically after a reboot or crash |
| `-p 8080:8080` | **Port mapping** for the web UI |
| `-p 50000:50000` | Inbound-agent TCP port. Only needed if agents connect *to* the controller over TCP. Omit it if you use WebSocket agents or no agents. |
| `-v jenkins_home:/var/jenkins_home` | **Volume**: persist all Jenkins data |
| `jenkins/jenkins:lts-jdk21` | Official Long Term Support image, Java 21 |

> **Java version, as of July 2026.** The current LTS line (2.555.x) **requires Java 21 or Java 25**;
> Java 17 support was removed at LTS 2.555.1 in May 2026, and the Java 17 controller images were
> dropped with it. `lts-jdk21` and `lts-jdk25` are both valid; `lts-jdk17` is not. If you inherited
> a Jenkins on Java 17, plan the upgrade before the next LTS baseline.

> **Pin the tag for anything that matters.** `lts-jdk21` moves. For a reproducible instance, pin the
> full version (`jenkins/jenkins:2.555.3-jdk21`) or a digest, and upgrade deliberately.

> **Term Explained — "port"**
> A numbered door on your machine that a program listens through. `-p 8080:8080` connects port 8080 on your real machine to port 8080 inside the container, so `http://localhost:8080` reaches Jenkins.

> **Term Explained — "volume"**
> Containers are disposable — delete one and its contents vanish. A **volume** stores data outside the container so it survives recreation. Everything Jenkins cares about lives under
> `/var/jenkins_home`, so that one mount is enough to make your instance durable.

**Verify:**

```bash
docker ps
docker logs -f jenkins   # watch startup; Ctrl-C to stop following
```

Then open `http://localhost:8080`.

> **Term Explained — "localhost"**
> A special address meaning "this same computer." `http://localhost:8080` is the service on port 8080 of your own machine — not somewhere on the internet.

---

## 6. First Login and the Web GUI Tour

### Unlocking Jenkins

The first screen is **Unlock Jenkins**, asking for the initial administrator password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

(The same string is also printed in `docker logs jenkins`, between rows of asterisks.)

Paste it into the browser.

### Install suggested plugins

Choose **Install suggested plugins**. This gives you Git, Pipeline, Credentials, Workspace Cleanup, Timestamper, and other essentials. You don't need to understand each one yet.

> **Not included in the suggested set**, and needed later in this document: **Docker** (cloud
> plugin, Section 12), **Docker Pipeline** (the `docker.build` step, Section 19), **Coverage**
> (Section 21), and **Pipeline Graph View** (Section 20). Install them when you reach those
> sections, not before.

### Create your admin user

Fill in your details → **Save and Continue** → accept the default Jenkins URL → **Save and Finish**
→ **Start using Jenkins**.

> **Do create a real admin account.** Skipping this leaves the throwaway `admin` account with the initial password active, which is a genuinely bad idea on anything reachable by others.

### The Jenkins UI, item by item

| UI element | Purpose |
|---|---|
| **Breadcrumbs** (top) | Navigate up and down the item hierarchy — you'll use these constantly |
| **New Item** | Create jobs, pipelines, and folders |
| **People** | User accounts |
| **Build History** | Every run across all jobs |
| **Manage Jenkins** | All administration — nodes, clouds, plugins, security, credentials |
| **My Views / New View** | Organize and filter jobs into dashboards |
| **Build Executor Status** (bottom-left) | Live view of which node is running what |

---

## 7. Manage Jenkins: What Admins Actually Use

### Nodes / Clouds

**Manage Jenkins → Nodes** for permanent agents; **Manage Jenkins → Clouds** for dynamic agent providers (Docker, Kubernetes, EC2). Anything agent-related lives in one of these two.

### Security
- **Security** — authentication realm and authorization strategy
- **Users** — accounts, API tokens
- **Credentials** — where you store SSH keys, API tokens, and passwords

> **Note on credentials:** You don't *have* to use Jenkins as your secret store — Vault, AWS Secrets Manager, or Azure Key Vault all integrate via plugins. But Jenkins has a built-in credential store, and using it is far better than pasting secrets into job configuration. **Scope
> credentials to a folder rather than globally** where you can, so one job can't read another
> team's secrets.

### Prepare for Shutdown

Never just reboot a busy controller — you'd kill running jobs mid-flight. **Manage Jenkins →
Prepare for Shutdown** stops new builds from starting and lets in-flight builds finish. Then
restart.

### Script Console

**Manage Jenkins → Script Console** runs arbitrary Groovy with full controller privileges. It is
the most powerful and most dangerous page in Jenkins — invaluable for bulk fixes, and a complete compromise if the wrong person has access. Treat access to it as equivalent to root on the controller.

---

## 8. Your First Freestyle Job

Go to **Dashboard → New Item**.

> **Notice — don't use spaces in job names**
> Jenkins creates a **filesystem directory named after the job**. Spaces work in most places but
> break shell one-liners, scripts, and `dir()` paths later. Use underscores or dashes.

Name it `my_first_job`, select **Freestyle project**, click **OK**.

### The categories of settings
1. **General** — description, discard old builds, parameters
2. **Source Code Management** — supply a Git URL and Jenkins clones it into the workspace
3. **Build Triggers** — what causes the job to run (Section 14)
4. **Build Environment** — notably "Delete workspace before build starts"
5. **Build Steps** — the actual work
6. **Post-build Actions** — publish test results, archive artifacts, notify

### Adding a build step

Under **Build Steps**, choose **Execute shell**. Whatever you write runs on the agent, inside the
job's workspace.

> **Term Explained — "shell" / "command line"**
> A text-based way of telling a computer to do things instead of clicking icons.
> `echo "hello"` prints `hello` — the "hello world" of shell.

Start simple:

```bash
echo "hello world"
```

Click **Save**, then **Build Now**. Under **Build History**, click the new build number (`#1`), then
**Console Output**.

> **Notice — the empty Git repository error**
> If you opened the Source Code Management section and left the repository URL blank, the first build fails immediately with a Git error. **Fix:** Configure → set SCM to **None** → Save.

Congratulations — you just automated something. Everything from here is the same loop:
**configure → trigger → observe**, with more steps.

---

## 9. Environment Variables and the Workspace

### Built-in environment variables

Jenkins injects a set of variables into every build. The full list for your instance is at
`http://<your-jenkins>/env-vars.html`, and there's a link to it next to the Execute shell box.

The ones you'll use constantly:

| Variable | Meaning |
|---|---|
| `BUILD_NUMBER` | The build number (1, 2, 3…). **Use this for versioning artifacts and image tags.** |
| `BUILD_ID` | Identical to `BUILD_NUMBER` since Jenkins 1.597. (Before that it was a timestamp — you will still find old scripts assuming that.) |
| `BUILD_URL` | Full URL of this build. Perfect for Slack/email notifications. |
| `JOB_NAME` | Name of the job, including folder path |
| `WORKSPACE` | Absolute path to the workspace on the agent. **Use this instead of hardcoding paths.** |
| `NODE_NAME` | Name of the agent running the build (`built-in` for the controller) |
| `EXECUTOR_NUMBER` | Which executor slot on that node — handy for allocating per-build ports |
| `GIT_COMMIT` | Commit SHA that was checked out (with the Git plugin) |
| `BRANCH_NAME` | Branch being built — **Multibranch pipelines only.** Not set in a plain Pipeline job. |

Using them in a shell step:

```bash
echo "hello world"
echo "build number: ${BUILD_NUMBER}"
echo "build url:    ${BUILD_URL}"
echo "running on:   ${NODE_NAME}"
echo "workspace:    ${WORKSPACE}"
```

> **Notice — "why is everything printed twice?"**
> In Console Output each command appears to repeat. The line beginning with `+` is **the command being executed** (the shell's trace output), and the line after it is **that command's standard output**. Since `echo` writes to stdout, you see both. Not a bug.

### Understanding the workspace

Add some file operations:

```bash
ls -ltr
echo "1234" > test.txt
ls -ltr
```

First run: the first `ls` shows nothing, then `test.txt` is created. View it via the **Workspace**
link on the job page.

> **Notice — workspaces persist between builds**
> Run it a second time and the first `ls` now shows `test.txt` already there. The workspace is
> **not** cleared between builds. Stale files from previous runs are a classic source of
> "it passed but shouldn't have" — a build can pick up an artifact it didn't actually produce.
>
> **Fix:** in **Build Environment**, check **Delete workspace before build starts**. In pipelines,
> use the `cleanWs()` step. Both come from the **Workspace Cleanup** plugin, which is included in the suggested-plugins set.

---

## 10. Exploring the Jenkins Filesystem

Get a shell inside the container:

```bash
docker exec -it jenkins bash
```

You're now on the controller. `JENKINS_HOME` is the volume you mounted:

```bash
cd /var/jenkins_home
ls -ltra
```

> **Path warning — `/var/jenkins_home` vs `/var/lib/jenkins`**
> The **Docker image** uses `/var/jenkins_home`. The **Linux .deb/.rpm packages** use
> `/var/lib/jenkins`. **Agents** use whatever remote root you configured (often
> `/home/jenkins/agent`), and the workspace inside that root is different again. Never hardcode a
> path that came from the controller into a step that runs on an agent — use `${WORKSPACE}`.

The directory you care about first:

```bash
cd workspace && ls -ltra
```

You'll see a folder named `my_first_job` — **matching the job name exactly.** This is why spaces in job names cause trouble.

```bash
cd my_first_job
ls
cat test.txt        # → 1234
```

### Why this matters

Getting into the filesystem — `docker exec`, `kubectl exec`, or SSH — is **something you will do constantly when troubleshooting**. It's frequently faster than reasoning about the UI.

### Other useful paths under `JENKINS_HOME`

| Path | Contains |
|---|---|
| `workspace/` | Per-job working directories (on the controller/built-in node only) |
| `jobs/<name>/config.xml` | That job's configuration |
| `jobs/<name>/builds/` | Build history, console logs, and per-build metadata |
| `plugins/` | Installed plugins (`.jpi` files) |
| `users/` | User accounts |
| `secrets/` | Master encryption key and the initial admin password. **Never share this directory.** |
| `logs/` | Controller logs |
| `config.xml` | Global configuration |
| `updates/` | Update-center metadata cache |

> **Backup tip:** backing up `JENKINS_HOME` — especially `jobs/`, `secrets/`, `users/`, and
> `*.xml` — is your disaster recovery plan. `workspace/` is disposable and can be excluded.
> `secrets/` and the rest must be restored **together**: without the master key, every stored
> credential in `credentials.xml` is unreadable.

---

## 11. Freestyle Job: Cloning Git and Running Python

Create **New Item** → `my_python_job` → **Freestyle project**.

### Configure Source Code Management

Select **Git** and enter your repository URL.

> **Credentials:** a public repo needs none. A **private** repo needs a credential — add one under
> Manage Jenkins → Credentials (SSH key, or a username + personal access token). Never put a token in the URL; it ends up in the console log.

### Probe before you assume

**Check what's actually on the machine before writing the real build step:**

```bash
python --version
```

Result: `command not found`.

```bash
python3 --version
```

That works. So the real build step uses `python3`:

```bash
python3 --version
python3 hello_world.py
```

> **Notice — `python` vs `python3`**
> Most modern Linux images ship `python3` with **no `python` alias** (Debian splits it into a
> separate `python-is-python3` package; many minimal images omit it entirely). Assuming `python` exists is a classic build failure. **Run a one-line probe job first** — this habit generalizes to every tool your build depends on.

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
> message is `ensurepip is not available`. On Alpine the packaging differs again. Before relying on
> it, run one probe step on the agent:
>
> ```bash
> python3 -m venv /tmp/probe && /tmp/probe/bin/pip --version && echo "venv OK"
> ```
>
> If that fails, install the right package in your agent image (Section 13) — don't work around it
> in the pipeline.

> **Shell gotcha — `set -euo pipefail`**
> Jenkins decides pass/fail from the script's **final exit code**. Without `set -e`, a command can
> fail in the middle and the build still goes green because the last line succeeded. Put
> `set -euo pipefail` at the top of every non-trivial shell step. This is one of the highest-value habits in this entire document.
>
> - `-e` exit on any error
> - `-u` error on undefined variables
> - `-o pipefail` a pipeline fails if *any* stage of it fails, not just the last

> **Note:** inside the venv, `python` and `pip` now exist and point at the venv — that's why the
> script above can use `python` after activating.

Save and build. The console shows Jenkins cloning the repository, then running your script.

### Why this is more powerful than it looks

Instead of logging onto a server, having SSH access, and running a script by hand, you press a
button — and get triggers, permanent history, full logs, and access control for free.

---

## 12. Setting Up Agents

Go to **Manage Jenkins → Nodes** (permanent agents) or **Manage Jenkins → Clouds** (dynamic).

### New Node = permanent agent

Configures a machine that's always available. Key fields:

| Field | Notes |
|---|---|
| **Number of executors** | Concurrent builds on this machine. Start at 1–2; don't exceed CPU count. |
| **Remote root directory** | A directory dedicated to Jenkins on that machine, e.g. `/home/jenkins/agent` |
| **Labels** | Space-separated tags jobs use to select this agent |
| **Usage** | "Use this node as much as possible" vs "Only build jobs with label expressions matching this node" — use the latter for special-purpose machines |
| **Launch method** | SSH (controller → agent) or inbound (agent → controller). See the launch-method table in Section 2. |

### Configure Clouds = dynamic agents

Install a cloud plugin first: **Manage Jenkins → Plugins → Available**, search `Docker` (or
`Kubernetes`, or `Amazon EC2`), install, and restart Jenkins. Verify afterwards under
**Plugins → Installed**.

> **Two plugins with confusingly similar names.** **Docker** (`docker-plugin`) provides the *cloud*
> — it provisions containers as agents, and it is the one you need here. **Docker Pipeline**
> (`docker-workflow`) provides the *steps* `docker.build`, `docker.image(...).inside`, and
> `agent { docker { ... } }`, used in Section 19. They are independent; most people end up wanting
> both.

### Configuring the Docker cloud

**Manage Jenkins → Clouds → New cloud → Docker**, then open **Docker Cloud Details**.

The critical field is the **Docker Host URI**:

| Situation | URI |
|---|---|
| Docker socket mounted into the Jenkins container | `unix:///var/run/docker.sock` |
| Docker daemon on another host, TLS enabled | `tcp://dockerhost.example.com:2376` + a Docker Server Certificate credential |
| Via a socat proxy on the same Docker network | `tcp://jenkins-docker-socat:2375` |

### Notice — the Docker Desktop connection problem

**The problem:** Jenkins runs inside a container; the Docker daemon runs on your host. The Jenkins container has no route to the host's Docker socket unless you give it one.

**Option A — mount the socket directly (simplest):** add
`-v /var/run/docker.sock:/var/run/docker.sock` to the Jenkins `docker run` and use
`unix:///var/run/docker.sock`. The Jenkins user must be able to read the socket — on most hosts the
socket is owned by `root:docker`, and the `jenkins` user inside the container is not in that group,
so this often needs the container's supplementary group set to the host's `docker` GID
(`--group-add "$(getent group docker | cut -d: -f3)"`). Confirm with
`docker exec jenkins docker version` if you also install a CLI, or simply by whether **Test
Connection** succeeds.

**Option B — socat proxy:**

```bash
docker run -d --name jenkins-docker-socat --restart unless-stopped \
  --network jenkins \
  -v /var/run/docker.sock:/var/run/docker.sock \
  alpine/socat \
  tcp-listen:2375,fork,reuseaddr unix-connect:/var/run/docker.sock
```

Because both containers are on the `jenkins` network, you can address it **by container name** — no IP hunting required:

```
tcp://jenkins-docker-socat:2375
```

Check **Enabled**, then **Test Connection**. A working connection prints the daemon's version
string; anything else, click **Show Details** and read the actual error.

> **🔒 Security warning — the Docker socket is root**
> Anything that can reach `/var/run/docker.sock` can start a privileged container and take over the host. Exposing it over **plain TCP on 2375 is unauthenticated and unencrypted**. This is acceptable on a laptop for learning; on a shared or production machine, use TLS on 2376 with client certificates, or use Kubernetes agents instead.
>
> The socat container above publishes no host port, so 2375 is reachable only from the `jenkins`
> Docker network — do **not** add `-p 2375:2375` to it.

### Creating a Docker agent template

With the cloud connected: **Docker Agent Templates → Add Docker Template**.

| Field | Value | Why |
|---|---|---|
| **Labels** | `docker-agent-python` | How jobs request this agent. **Describe the capability, not the machine.** |
| **Enabled** | ✅ | Otherwise it's never used |
| **Name** | `docker-agent-python` | Same as the label, for clarity |
| **Docker Image** | `jenkins/inbound-agent:latest-alpine-jdk21` (or your own — see Section 13) | The image the agent runs |
| **Instance Capacity** | `2` | Max concurrent agents from this template. **Keep it low at first** — a misconfiguration can otherwise spawn agents until the host falls over. |
| **Remote File System Root** | `/home/jenkins/agent` | Where the workspace is created inside the container |
| **Connect method** | *Attach Docker container* or *Connect with JNLP* | Attach uses `jenkins/agent:*`; JNLP/inbound uses `jenkins/inbound-agent:*` — the image must match the method |
| **Pull strategy** | *Pull once and update latest* (default) | **Change this to "Never pull" for images you built locally** and never pushed to a registry, or provisioning fails trying to pull them. |

> **Private registries:** supply a credential so the daemon can authenticate and pull.

> **Tag check.** `jenkins/inbound-agent:latest-alpine-jdk21` is a real, currently published tag, as
> are `latest-jdk21` (Debian), `latest-alpine-jdk25`, and `latest-jdk25`. Pick Debian if you want
> `apt-get`, Alpine if you want a small image.

---

## 13. Using Labels to Route Jobs to Agents

Open your Freestyle job → **Configure** → check **Restrict where this project can be run** → enter
`docker-agent-python`.

In a pipeline, the same thing is one line:

```groovy
agent { label 'docker-agent-python' }
```

Labels also support expressions: `linux && python3`, `hw-tester || simulator`.

> **Notice — the phantom "label doesn't exist" warning**
> Jenkins sometimes warns that a label doesn't exist when it does. Two causes:
> 1. **A trailing space** inserted by autocomplete. Delete it.
> 2. **Cloud templates aren't always reflected in the label index until an agent has actually been provisioned once.** Navigating out and back in usually clears it; so does running the job.

Save and **Build Now**. The console shows Jenkins waiting for an agent with that label, then
building on a dynamically created container.

> **Notice — be patient**
> Dynamic agents take time to provision: pull the image, start the container, connect back to the controller. A build sitting in the queue for 30 seconds is normal. Don't start troubleshooting before the agent has had a chance to come up.

### Notice — the agent image doesn't have your tools

Point the Python job at a stock agent image and it **fails**: the code checks out fine, but
`python3: not found`.

**Why:** stock Jenkins agent images contain a Java runtime and the agent remoting jar. **Nothing
else.**

**The fix: build your own agent image.**

Debian-based (recommended — `apt` packaging of `venv` is well documented and predictable):

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

Alpine-based (smaller; verify `venv` works before you rely on it):

```dockerfile
# Dockerfile.agent.alpine
FROM jenkins/inbound-agent:latest-alpine-jdk21

USER root
RUN apk add --no-cache python3 py3-pip git curl
USER jenkins
```

Build it, and **verify the toolchain before wiring it into Jenkins**:

```bash
docker build -t jenkins-agent-python:1.0 -f Dockerfile.agent .

# probe the image directly — much faster than debugging through a build
docker run --rm --entrypoint sh jenkins-agent-python:1.0 -c \
  'python3 --version && python3 -m venv /tmp/probe && /tmp/probe/bin/pip --version && echo VENV_OK'
```

Then create a template pointing at that image with the label `docker-agent-python`, and the Python
job runs successfully.

> **Local image, no registry.** If the image only exists on the Docker host (no `docker push`), set
> the template's **Pull strategy** to **Never pull**. Otherwise the plugin tries to pull
> `jenkins-agent-python:1.0` from Docker Hub and fails. If you *do* have a registry:
>
> ```bash
> docker build -t my-registry.example.com/jenkins-agent-python:1.0 -f Dockerfile.agent .
> docker push my-registry.example.com/jenkins-agent-python:1.0
> ```

> **Pin your agent image tags.** `:latest` means your builds change underneath you without warning and are not reproducible. Use `:1.0`, `:2026-07`, or a digest.

**The lesson:** agents are just containers, and containers only contain what you put in them. In a
real environment you maintain your own agent images with your team's toolchain baked in.

> **Practical relevance:** this is exactly how you handle specialized toolchains or hardware — one label for agents with a particular SDK or compiler, another for the two machines with physical test hardware attached. Jobs request what they need by label, and the controller does the routing. For hardware-attached agents, use permanent agents with **1 executor** so two builds can't grab the same device simultaneously.

---

## 14. Automatic Build Triggers

### GitHub webhook (push-based, near-instant)

The repository tells Jenkins about a push, rather than Jenkins asking repeatedly.

1. Install the **GitHub** plugin (included in suggested plugins).
2. In the job: **Build Triggers → GitHub hook trigger for GITScm polling**.
3. In GitHub: **Settings → Webhooks → Add webhook**, Payload URL
   `http://<your-jenkins>/github-webhook/` (**the trailing slash matters**), content type
   `application/json`, event: *Just the push event*.

> **The job's Git URL must match the webhook's repository.** The trigger works by matching the
> pushed repository against the SCM URL configured in the job. A job cloning `git@github.com:me/x`
> and a webhook from `https://github.com/me/y` will silently never fire.

> **Notice — firewalls break webhooks**
> A webhook is GitHub making an **inbound** connection to your Jenkins. If Jenkins is on a private network, GitHub cannot reach it. Options: expose it through a reverse proxy, put it behind an ingress with an allow-list of GitHub's published IP ranges, or fall back to polling.

### Poll SCM (pull-based, works behind a firewall)

Jenkins **reaches out to the repository periodically** to check for changes, so no inbound
connection is needed. Cron-like syntax:

```
H/5 * * * *
```

**Field order:** `MINUTE HOUR DAY-OF-MONTH MONTH DAY-OF-WEEK`

> **Important — what `H` actually means**
> `H` is **hash**, not "random". Jenkins hashes the *job name* to pick a fixed offset inside the interval, so every job gets a different but **stable** slot. Job A polls at :02, :07, :12…, job B at :04, :09, :14… — spreading load instead of having every job hammer the SCM at :00.

Common schedules, since this is where most mistakes happen:

| Expression | Meaning |
|---|---|
| `H/5 * * * *` | Every 5 minutes, at a job-specific offset |
| `H * * * *` | **Once per hour** — `H` alone picks one minute in the hour |
| `H H(0-6) * * *` | Once daily, at a job-specific time between 00:00 and 06:59 |
| `H H * * 1-5` | Once daily, weekdays only |
| `*/5 * * * *` | Every 5 minutes for *every* job at once — works, but this is what `H/5` exists to avoid |

### Build periodically

A plain cron schedule: runs whether or not anything changed. Right for nightly full regressions, scheduled cleanups, or long soak tests. Wrong for reacting to commits.

### Other triggers worth knowing

| Trigger | Use |
|---|---|
| **Build after other projects are built** | Chain jobs |
| **Trigger builds remotely** | `curl` a URL with a token — how external systems kick off builds |
| **Generic Webhook Trigger** (plugin) | Accept a webhook from anything, not just GitHub |
| `cron` / `pollSCM` / `upstream` in `triggers {}` | The pipeline-as-code equivalents |

### Confirming it worked

After a commit, the build starts on its own and the build page shows **"Started by an SCM change"** at the top — proof it was the commit and not a button press.

> **Notice — poll SCM latency is inherent**
> Polling is not instant; with `H/5` the worst case is about five minutes plus the poll itself.
> If you need fast feedback, use webhooks. Shortening the poll interval to compensate just hammers your SCM server — and on hosted Git it can get you rate-limited.

---

## 15. Anatomy of a Jenkinsfile

A `Jenkinsfile` is a **text file containing your pipeline definition** — a template, a set of
instructions. It says what the pipeline does, on what agent, and which plugins and credentials it needs.

### The structural pattern

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

**1. `pipeline`** — the outermost block of every declarative pipeline. Everything lives inside it.

**2. `agent`** — *where* it runs.

```groovy
agent any                              // any available agent
agent none                             // no global agent; each stage declares its own
agent { label 'docker-agent-python' }  // by label
agent { docker { image 'python:3.12-slim' } }  // run inside a container
```

With `agent any` and no agents configured, the build runs on the controller — see the Section 2 warning.

**3. `stages` / `stage`** — the phases of work. Names appear in the UI, so make them meaningful:
`Checkout`, `Lint`, `Test`, `Package`, `Deploy`.

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

> **`post` is where reliability comes from.** Steps in `post { always { ... } }` run even when the stage failed. Test-report publishing belongs there — otherwise a failing test suite aborts the stage and you never get the report that tells you *which* test failed.
>
> `post` conditions run in a defined order, with **`cleanup` always last** — which is why
> `cleanWs()` in `cleanup` doesn't destroy the workspace before `junit` and `archiveArtifacts`
> in `always` have read it.

> **`credentials()` in `environment`.** Masking is best-effort: Jenkins replaces the literal value
> with `****` in the console. It cannot mask a transformed value (base64-encoded, split, echoed one
> character at a time). Also note that a **username/password** credential binds to *three*
> variables — `API_TOKEN`, `API_TOKEN_USR`, and `API_TOKEN_PSW` — which surprises people the first
> time. For anything sensitive, prefer `withCredentials` scoped to the step that needs it.

### The interpolation trap you will hit

```groovy
sh "echo ${MY_VAR}"    // Groovy substitutes BEFORE the shell sees it
sh 'echo $MY_VAR'      // the SHELL expands it at runtime
```

Both often produce the same output, which is why it's confusing. The difference matters:

1. With **double quotes**, the value is baked into the command line. If it contains a secret, that
   secret appears in the console log and in `ps` output. **Never use double-quote interpolation for credentials.**
2. With **double quotes**, a value containing spaces or quotes can break or inject into the
   command.

```groovy
// wrong — token ends up in the log
sh "curl -H 'Authorization: Bearer ${TOKEN}' https://api.example.com"

// right — the shell reads it from the environment, and Jenkins masks it
withCredentials([string(credentialsId: 'my-api-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
}
```

> **Editor tip:** name the file exactly `Jenkinsfile`. VS Code and IntelliJ recognize it and apply
> Groovy highlighting. Jenkins also exposes a linter you can call before pushing:
>
> ```bash
> # via the CLI (needs an API token)
> java -jar jenkins-cli.jar -s http://localhost:8080/ -auth user:token \
>   declarative-linter < Jenkinsfile
>
> # or over HTTP
> curl -X POST -u user:token -F "jenkinsfile=<Jenkinsfile" \
>   http://localhost:8080/pipeline-model-converter/validate
> ```
>
> If your instance enforces CSRF (the default), the `curl` form also needs a crumb header; the CLI
> form handles that for you and is the easier of the two.

### Declarative vs. scripted

> When looking up syntax online, **make sure you're reading about declarative pipelines**. Scripted pipelines (`node { ... }`) are the older, lower-level form. Mixing scripted examples into a declarative pipeline is the most common beginner failure. If you need imperative logic inside declarative, wrap it in a `script { ... }` block.

---

## 16. Your First Declarative Pipeline

**New Item** → `my_first_build_pipeline` → **Pipeline** → **OK**.

### Two ways to supply the script

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

The `agent` line uses a **label** — the mechanism from Section 13, now expressed as code.

Each stage has two steps: an `echo` executed by Jenkins itself, and an `sh` block executed by the agent's shell.

> **Why triple single quotes?** `'''…'''` is a multi-line Groovy string with **no** interpolation —
> `$VAR` reaches the shell untouched. `"""…"""` interpolates in Groovy first. Default to `'''` for
> shell blocks.

> **The shebang inside `sh` is real.** Jenkins writes the script to a temporary file and executes
> it, so `#!/usr/bin/env bash` genuinely selects bash. It must be the **very first characters** of
> the string — `'''` immediately followed by `#!`, with no leading newline or spaces, exactly as
> written above. Without it you get `/bin/sh`, where `set -o pipefail` may not exist.

### Reading the Stage View

Save and build. The view is immediately different from a Freestyle build — it's **split into your
three stages**, each green or red, with per-stage timing. Click any stage to see its logs.

### Notice — upstream stage failure cascades

If Test contains a syntax error, Deliver fails too — even though Deliver's own syntax is fine.
**Because Test is upstream.** A failed stage stops the pipeline; downstream stages never run.

That's the correct default. Where you genuinely want to continue, be explicit:

```groovy
stage('Optional lint') {
    steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
            sh 'ruff check .'
        }
    }
}
```

---

## 17. Moving the Pipeline into Source Control

Writing the pipeline in the UI works, but the point of pipeline-as-code is version control: review, history, blame, rollback.

### Create the Jenkinsfile

Save the pipeline script as a file named `Jenkinsfile` in your repository root and commit it.

### Add triggers as code

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    triggers {
        pollSCM('H/15 * * * *')
    }

    stages {
        // ...
    }
}
```

> **Chicken-and-egg:** a `triggers` block only takes effect **after the pipeline has run at least
> once**, because Jenkins has to read the Jenkinsfile to learn about the trigger. Run it manually
> the first time.

### Point the job at the Jenkinsfile

In the job's **Pipeline** section:

- **Definition:** *Pipeline script from SCM*
- **SCM:** Git
- **Repository URL:** your repo (add credentials if private)
- **Branch Specifier:** `*/main` (Jenkins still defaults to `*/master` in some places — check it)
- **Script Path:** `Jenkinsfile` — any name and path works as long as this field matches

Save and build. The console confirms it **obtained the Jenkinsfile from Git**.

### Notice — a new "Checkout" stage appears and history resets

1. **An extra stage appears** — `Declarative: Checkout SCM`, which you didn't write. With *Pipeline script from SCM*, Jenkins must clone the repo to read the Jenkinsfile, and that clone shows up as a stage. Expected.
2. **The Stage View history clears.** Adding a stage changes the pipeline's shape, so the
   visualization re-baselines. Nothing is lost — the builds still exist and their logs are intact.

> **Notice — the file in Git is the source of truth**
> A classic waste of an afternoon: editing your local template copy, or the old inline script in
> the UI, and wondering why nothing changes. Once the job is set to *from SCM*, **only the
> committed and pushed file matters.**

### Where to go next: Multibranch

Once the Jenkinsfile is in the repo, switch from a Pipeline job to a **Multibranch Pipeline**.
Jenkins then discovers every branch and pull request containing a Jenkinsfile and builds each one automatically — no per-branch job configuration. For most teams this is the destination.

It also unlocks `BRANCH_NAME` and `when { branch '...' }`, both of which do nothing in a plain
Pipeline job — see the warning in Section 18.

---

## 18. Multi-Stage Pipelines

A multi-stage pipeline is **one Jenkinsfile, multiple stages.**

You *could* keep one Jenkinsfile per environment. Or combine them and progress through
environments in sequence:

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
            when { branch 'main' }        // only from main — Multibranch only, see below
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh 'echo "deploy to prod"'
            }
        }
    }
}
```

> **⚠️ `when { branch 'main' }` only works in a Multibranch Pipeline.** It compares against
> `BRANCH_NAME`, which a plain Pipeline job never sets. In a plain Pipeline job the condition is
> simply never true and the stage is **silently skipped forever** — a genuinely nasty failure mode,
> because nothing errors. If you are on a plain Pipeline job, gate on something that exists instead:
>
> ```groovy
> when { expression { env.GIT_BRANCH ==~ /(origin\/)?main/ } }
> ```
>
> Or, better, move to a Multibranch Pipeline (Section 17).

### The safety property

The valuable behavior: if Dev fails, you never reach UAT. Fix it, try again. Only after Dev passes
does it reach UAT; only after UAT passes does it reach Prod. This is the upstream/downstream
behavior from Section 16, used deliberately as a deployment gate.

Two additions above make it production-shaped:

- **`when { branch 'main' }`** — skip the stage entirely on feature branches
- **`input`** — pause and wait for a human. Note that an `input` step **holds an executor** while
  it waits; for long waits, put it outside the agent block or use the `agent none` pattern.

### Running independent work in parallel

Stages that don't depend on each other shouldn't be serialized:

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
> writing the same file will collide. Give each branch a distinct output path (`reports/junit-lint.xml`
> vs `reports/junit-tests.xml`), as the pipelines in Section 19 do.

### Building it up in the UI

1. Create a Pipeline job with the built-in **Hello World** sample → build → one stage
2. Configure, rename the stage to `Dev` → build → the view now shows "Dev"
3. Configure, copy the whole `stage` block, paste, rename the copy to `UAT` → build → two stages

The Stage View then shows both, and the earlier single-stage runs remain visible in history.

---

## 19. A Real Python CI Pipeline

Everything below is Python and shell. The sample project it builds — every file, ready to commit —
is in [Appendix C](#appendix-c--the-sample-project).

### Prerequisite: there is no `tools` block for Python

Jenkins core ships tool installers for JDK, Maven, Gradle, and Git. **There is no Python installer**,
so `tools { python 'Python3' }` fails with `Tool type "python" does not have an install of the name...`.
Choose one of:

1. **Bake Python into the agent image** (Section 13) — recommended, most reproducible
2. **Run the stage in a container**: `agent { docker { image 'python:3.12-slim' } }` (needs the
   Docker Pipeline plugin and a reachable daemon)
3. **Use the system Python + a venv** — what the examples below do, and the most portable

### A note on lint, tests, and build results before you copy anything

Three facts interact here, and getting them wrong is the single most common reason these pipelines
behave unexpectedly:

- `pytest` exits **non-zero** when tests fail, and exits **5** when it collects no tests at all.
- `ruff check` exits **1** when it finds violations.
- With `set -e`, either of those **fails the stage immediately**, which makes the whole build
  **FAILURE**.

The `junit` step's ability to mark a build **UNSTABLE** (yellow) only comes into play if the stage
itself did *not* fail. So:

- **Want a failing test to fail the build (red)?** Do nothing — that's the default above, and it's a
  perfectly reasonable policy.
- **Want failing tests to show as UNSTABLE (yellow) instead**, so "tests failed" reads differently
  from "the pipeline broke"? Wrap the step:

  ```groovy
  catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
      sh '... pytest ...'
  }
  ```

Either policy is defensible. What is *not* defensible is assuming you have one when you've
configured the other. The pipelines below use the default (a failing check fails the build) and
call out where to change it.

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
- **`PYTHONPATH = "${WORKSPACE}"`** so `import app` resolves. Without it, `pytest` run from a venv
  that never installed your project raises `ModuleNotFoundError: No module named 'app'` — one of the
  most common first failures. The alternative is `pip install -e .`, which needs a `pyproject.toml`
  and is slower.
- **`${WORKSPACE}`**, not a hardcoded path. The workspace differs between controller, Docker agent, and Kubernetes pod.
- **Each `sh` step is a separate shell process.** A venv `activate`d in one step is gone in the
  next. That's why the examples call `"$VENV/bin/..."` explicitly — it's more robust than
  re-activating everywhere. (Alternative: set `PATH = "${VENV}/bin:${PATH}"` in `environment`.)
- **`--output-file` rather than `>` redirection.** `ruff ... > file` truncates the file the instant
  the shell opens it, so if ruff dies abnormally (exit 2 — bad config, internal error) you are left
  with a zero-byte XML that `junit` then reports as a malformed test report. `--output-file` writes
  only real output. The same reasoning applies to any tool with a native output-file flag.
- **`junit` in `post { always }`** — a failing test suite makes `pytest` exit non-zero and fails the
  stage. Publishing in `post/always` is what gets you the report anyway.
- **`allowEmptyResults: true`** stops the *first* run from failing on a missing report while you're
  still wiring things up. Once the pipeline is stable, set it to `false` — otherwise a test stage
  that silently produced nothing looks identical to one that passed.

### Stage 2 — Package the application

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
without one. Appendix C includes a minimal working example.

> **The packaging failure you'll hit first.** With both `app/` and `tests/` present and no explicit
> package configuration, setuptools finds two top-level packages and refuses to guess, failing with
> a "Multiple top-level packages discovered" error. Fix it by declaring what to package:
>
> ```toml
> [tool.setuptools.packages.find]
> include = ["app*"]
> ```

`archiveArtifacts` copies files back to the **controller** so they survive the agent being
destroyed. `fingerprint: true` records a checksum, letting Jenkins tell you later which build
produced a given file.

> Don't archive gigabytes. Archived artifacts live in `JENKINS_HOME` forever unless you set `buildDiscarder`. For anything large, push to a real artifact repository (Artifactory, Nexus, S3) and archive only the metadata.

### Stage 3 — Build a Docker image

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

**Prerequisites:** the **Docker Pipeline** plugin (not the Docker *cloud* plugin — see Section 12),
plus a working `docker` CLI **on the agent** with permission to reach a daemon. An agent that is
itself a container needs the socket mounted or a remote daemon configured — and the security
warning from Section 12 applies. Note that the stock `jenkins/inbound-agent` images do **not**
contain a `docker` CLI; you have to add one.

Tag with `BUILD_NUMBER` (or the Git SHA), never only `latest`. `latest` tells you nothing about
which build produced a running container.

### Stage 4 — Smoke-test the built application

```groovy
        stage('Smoke test') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail

                    "$VENV/bin/python" -m app.main --port 8000 &
                    APP_PID=$!
                    trap 'kill "$APP_PID" 2>/dev/null || true' EXIT

                    for _ in $(seq 1 20); do
                        if curl -fsS http://127.0.0.1:8000/health; then
                            echo "app is up"
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

- **Use the `if ... then ... fi` form, not `curl ... && exit 0`.** Both happen to work under
  `set -e`, but only because of a subtle exemption for the left-hand side of `&&`. The `if` form
  needs no such reasoning.
- **A hardcoded port collides.** Two concurrent builds on the same agent both binding 8000 will
  fight. Derive it: `PORT=$((8000 + ${EXECUTOR_NUMBER:-0}))`, or set the agent to 1 executor.

> **Why this exists as a section:** the naive version of this stage — start the app in the
> background and move on — produces a build that mysteriously "passes" while the app never started,
> and leaves a stray process that Jenkins' ProcessTreeKiller then reaps at the end of the build,
> confusing everyone. Start, verify, and stop in one step.

### The complete pipeline

```groovy
pipeline {
    agent { label 'docker-agent-python' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        PYTHONDONTWRITEBYTECODE       = '1'
        VENV                          = "${WORKSPACE}/.venv"
        PYTHONPATH                    = "${WORKSPACE}"
        IMAGE_NAME                    = 'my-python-app'
    }

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
                                --cov=app --cov-report=xml:reports/coverage.xml
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
                '''
            }
        }

        stage('Build image') {
            when { branch 'main' }          // Multibranch only — see Section 18
            steps {
                script {
                    docker.build("${env.IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
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
                            echo "app is up"
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

> **`skipDefaultCheckout(true)` + explicit `cleanWs()` + `checkout scm`** gives you full control over when the workspace is cleaned and when the code arrives. Without it, Jenkins checks out before your first stage and `cleanWs()` would delete the code you just fetched.

> **Two prerequisites this pipeline does not check for you.** The `Build image` stage needs the
> Docker Pipeline plugin plus a `docker` CLI on the agent; if you don't have both, delete that
> stage rather than debugging it — nothing else depends on it. And `when { branch 'main' }` means
> the stage is skipped entirely in a plain Pipeline job.

---

## 20. Pipeline Visualization

### Stage View (built in)

The default view on a Pipeline job: stages as columns, runs as rows, per-stage duration, and a
hover-to-see-logs popup. Comes with the Pipeline: Stage View plugin, part of the suggested set. For
most people this is enough.

### Pipeline Graph View (recommended addition)

A modern graph rendering that lives **inside the normal Jenkins UI** — no separate interface. It
handles parallel stages and nested stages far better than the classic Stage View, and it is the
plugin the Jenkins project now points people toward.

Install: **Manage Jenkins → Plugins → Available → "Pipeline Graph View"**.

> **Blue Ocean is not the answer any more.** Older tutorials — including most of the popular video
> ones — send you to Blue Ocean for pipeline visualization. Blue Ocean has been in maintenance-only
> status for years and is no longer recommended for new installations. Pipeline Graph View is its
> replacement.

---

## 21. Reading Build Output and Reports

### Console Output

The primary artifact of every build. Two links next to each entry in Build History:

- The **build number** → the main build page
- The **status icon** (✅/❌) → straight to console output

Remember from Section 9: lines starting with `+` are the commands; the following lines are their output. That's why things appear "twice".

For long logs, `http://<jenkins>/job/<name>/<n>/consoleText` gives you the raw text — greppable, and much faster than the rendered page.

### Build page information

Shows which agent it ran on, what triggered it, duration, changes included since the last build, and links to the workspace and artifacts.

### Workspace browser

The **Workspace** link lets you browse the files the build produced, right in the UI. Useful for
confirming a build actually generated what you expected. Note that on ephemeral Docker agents the workspace is gone once the container is destroyed — that's exactly why you `archiveArtifacts`.

### Test result reporting

The **JUnit XML** format is the lingua franca. Most Python tools emit it:

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

> **But only if the stage didn't already fail.** As covered in Section 19, `pytest` exiting
> non-zero under `set -e` fails the stage, and a failed stage means the build is FAILURE regardless
> of what `junit` would have said. If you want the yellow, wrap the test step in
> `catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE')`. This surprises nearly everyone
> the first time.

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
    qualityGates: [[threshold: 60.0, metric: 'LINE', baseline: 'PROJECT', unstable: true]]
)
```

### Archiving artifacts

```groovy
archiveArtifacts artifacts: 'dist/*.whl', fingerprint: true, allowEmptyArchive: false
```

### Notifications

For Freestyle jobs, **Post-build Actions** handle email and Slack. For pipelines, use the `post`
block. Either way, include `${BUILD_URL}` so the recipient can click straight through:

```groovy
post {
    failure {
        mail to: 'team@example.com',
             subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
             body: "See ${env.BUILD_URL}"
    }
}
```

The `mail` step needs SMTP configured under **Manage Jenkins → System → E-mail Notification**;
without it the step fails and — because it's inside `post` — can turn an otherwise-informative
failure into a confusing one.

> **Notify on state change, not on every build.** A channel that gets a message for all 200 green
> builds a day is a channel nobody reads. Consider `post { unsuccessful { ... } }` plus a
> `post { fixed { ... } }` "recovered" notification.

### Job dashboard columns

The dashboard shows last success, last failure, and duration per job — a quick health view across everything.

---

## 22. Troubleshooting Guide

| Symptom | Cause | Fix |
|---|---|---|
| First build fails immediately with a Git error | SCM section opened but repository URL left blank | Set **Source Code Management** to **None** |
| `python: command not found` | Image ships `python3`, with no `python` alias | Use `python3`; probe with a one-line job first |
| `ensurepip is not available` from `python3 -m venv` | The `venv` module is packaged separately (Debian: `python3-venv`) | Install it in the agent image; probe the image directly before wiring it in |
| `pip: command not found` inside a venv step | Each `sh` step is a new shell — the previous `activate` is gone | Call `"$VENV/bin/pip"` explicitly, or set `PATH` in `environment` |
| `ModuleNotFoundError: No module named 'app'` | The venv never installed your project, so it isn't on `sys.path` | Set `PYTHONPATH = "${WORKSPACE}"` in `environment`, or `pip install -e .` |
| `Multiple top-level packages discovered` from `python -m build` | setuptools found both `app` and `tests` and won't guess | Add `[tool.setuptools.packages.find]` with `include = ["app*"]` |
| `pip install -r requirements.txt` fails, file not found | The file doesn't exist — an empty one is still required | Commit an empty `requirements.txt` with a comment in it |
| Build is green but tested nothing | `pytest` exited 5 (no tests collected) and nothing checked | Add `set -euo pipefail`; verify the test path; set `allowEmptyResults: false` |
| Tests failed but the build is red, not yellow | The stage failed before `junit` could downgrade it to UNSTABLE | Wrap the test step in `catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE')` |
| `junit` reports a malformed/empty report | `tool > file` truncated the file before the tool crashed | Use the tool's native `--output-file` flag |
| Agent runs but a tool is missing | Stock agent image contains only a Java runtime and the remoting jar | Build a custom agent image with your toolchain baked in |
| `Tool type "python" does not have an install...` | There is no Python tool installer for `tools {}` | Use an agent image, a Docker agent, or a venv |
| Docker cloud can't pull your locally built image | Default pull strategy tries the registry | Set the template's **Pull strategy** to **Never pull**, or push to a registry |
| Label "doesn't exist" warning despite existing | Trailing space from autocomplete, or the label index not yet populated for a cloud template | Remove the space; run the job once |
| Build queued forever waiting for an agent | Dynamic agent still provisioning, or instance capacity reached | **Wait.** Then check the cloud's instance cap and the Docker daemon |
| Docker cloud connection fails | Jenkins container can't reach the Docker daemon | Mount the socket, or run the socat proxy on a shared network and address it by name |
| `permission denied` on `/var/run/docker.sock` | Agent user isn't in the docker group | Fix group membership (`--group-add`) on the agent, or use a remote daemon over TLS |
| A stage is always skipped and never errors | `when { branch 'main' }` in a plain Pipeline job — `BRANCH_NAME` is unset | Use a Multibranch Pipeline, or gate on `env.GIT_BRANCH` |
| Files from a previous build present in workspace | Workspaces persist by default | **Delete workspace before build starts** / `cleanWs()` |
| Two concurrent builds fight over a port or device | Shared agent, hardcoded resource | Derive from `EXECUTOR_NUMBER`, use `lock()`, or set the node to 1 executor |
| Downstream stages fail despite correct syntax | An upstream stage failed | Fix the upstream stage first |
| Console output appears duplicated | The `+` line is the command; the next is its stdout | Not a bug |
| Pipeline changes have no effect | Editing the wrong file when using Pipeline-from-SCM | The Jenkinsfile **in Git** is the source of truth — commit and push |
| Unexpected "Declarative: Checkout SCM" stage | Pipeline-from-SCM auto-adds the clone | Expected behavior |
| Stage View history disappeared | A stage was added, so the visualization re-baselines | Cosmetic; builds still exist |
| `triggers {}` in the Jenkinsfile never fires | Jenkins hasn't read the file yet | Run the job manually once |
| Webhook fires but no build starts | The job's Git URL doesn't match the pushed repository | Make the SCM URL and the webhook repository the same |
| Poll SCM doesn't trigger promptly | Polling has inherent latency; `H` adds a fixed offset | Use webhooks for fast feedback |
| Scheduled job runs far less often than expected | `H * * * *` is **hourly**, not per-minute | Use `H/5 * * * *` for every 5 minutes |
| A secret appeared in the console log | Groovy double-quote interpolation baked it into the command | Use `withCredentials` + single-quoted `sh` |
| Background process dies when the build ends | ProcessTreeKiller cleans up spawned processes | Start/verify/stop in one step; use a real deployment mechanism for services |
| Build hangs forever | No timeout configured | `options { timeout(time: 30, unit: 'MINUTES') }` |
| Controller disk filling up | Unbounded build history and archived artifacts | `buildDiscarder(logRotator(...))` on every job |
| Alarming log line on a successful build | Not everything scary means failure | Verify the actual outcome — then find the real explanation |

### General troubleshooting technique

1. **Get into the filesystem.** `docker exec -it <container> bash` (or `kubectl exec`) and look.
   Often faster than reasoning through the UI.
2. **Read the error, then search it.** Click **Show Details** on connection failures, take the exact
   text, search it. This is genuinely how the work gets done.
3. **Reproduce the step by hand on the agent.** If `sh 'pytest ...'` fails, run that exact command in the agent's shell, in the workspace directory. Nine times out of ten the difference is an environment variable or a `PATH`.
4. **Probe the image, not the pipeline.** `docker run --rm --entrypoint sh <image> -c '<command>'`
   answers "is the tool there?" in two seconds instead of a four-minute build.
5. **Check `Manage Jenkins → System Log`** for controller-side errors that never reach the build log.
6. **Bisect the pipeline.** Comment out stages until it passes, then add them back one at a time.

---

## 23. Best Practices

### Pipeline design

- **Keep the Jenkinsfile in the repository** it builds. Same review process, same history.
- **Keep it thin.** Jenkinsfiles should call scripts (`./scripts/test.sh`), not contain the logic.
  Then developers can run the exact same script locally, and you're not debugging CI logic through a 4-minute feedback loop.
- **Make builds reproducible.** Pin agent image tags, pin dependency versions
  (`requirements.txt` with `==`, or a lockfile).
- **Fail fast.** Put lint and unit tests before anything slow.
- **Set a timeout on every pipeline.** A hung build holds an executor forever.
- **Set `buildDiscarder` on every job.** Otherwise `JENKINS_HOME` grows without limit.
- **Publish test results in `post { always }`.** Otherwise you lose the report exactly when you
  need it.
- **Decide your red/yellow policy explicitly**, and write it down in the Jenkinsfile as a comment.

### Agents

- **Never build on the controller.** Set the built-in node to 0 executors.
- **Maintain your own agent images** with your toolchain baked in, and version them.
- **One executor for agents with attached hardware**, so two builds can't fight over a device.
- **Prefer ephemeral agents.** A clean container per build eliminates an entire class of
  "works on agent 3 but not agent 5" problems.

### Security

- **Store secrets in the credential store**, never in job config, environment blocks, or repos.
- **Use `withCredentials` and single-quoted shell** so secrets don't get interpolated into command lines and logged.
- **Scope credentials to folders** rather than globally.
- **Restrict Script Console access** — it's equivalent to root on the controller.
- **Keep plugins updated.** Most Jenkins CVEs are in plugins, not core.
- **Never expose an unauthenticated Docker socket over TCP** on a shared machine.

### Operations

- **Back up `JENKINS_HOME`** — especially `jobs/`, `users/`, `secrets/`, and `*.xml`. Test the
  restore, and restore `secrets/` together with the rest or every stored credential is lost.
- **Use Prepare for Shutdown** before maintenance.
- **Consider Configuration as Code (JCasC)** to define the whole controller in a YAML file, so it's reviewable and rebuildable rather than a pile of hand-clicked settings.
- **Notify on state change**, not on every build.

### Naming

- **No spaces in job names** — they become directory names.
- **Descriptive labels** that name a capability (`python3`, `hw-nvme-tester`, `windows-msvc`), not
  a machine (`server-04`). Machines get replaced; capabilities persist.

---

## Appendix A — End-to-end walkthrough

The shortest path from nothing to a green pipeline. Every step links back to the section that
explains it.

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

### 2. Confirm what the controller has (Section 11)

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
| **A. Docker agent image** (Sections 12–13) | Highest | You want the realistic setup |
| **B. `agent { docker { image 'python:3.12-slim' } }`** | Medium | You have the Docker Pipeline plugin and a reachable daemon |
| **C. Install into the controller** | Lowest | You just want to see the pipeline run today |

Option C, for learning only — **this is lost whenever the container is recreated**, because it
writes to the container layer and not to the `jenkins_home` volume:

```bash
docker exec -u root jenkins bash -c \
  'apt-get update && apt-get install -y --no-install-recommends python3 python3-venv python3-pip && rm -rf /var/lib/apt/lists/*'
```

Re-run the probe job to confirm. If you take this path, use `agent any` in place of
`agent { label 'docker-agent-python' }` everywhere below.

### 4. Create the sample repository (Appendix C)

Create the files from Appendix C, then:

```bash
git init
git add .
git commit -m "Initial commit: sample Python project with Jenkinsfile"
git branch -M main
git remote add origin https://github.com/<you>/jenkins-sample.git
git push -u origin main
```

### 5. Create the pipeline job (Section 17)

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
| Coverage parsed | **Coverage** on the build page | A percentage and per-file table |
| Artifact archived | **Build Artifacts** | `dist/*.whl` downloadable |
| Stage view renders | Job page | Checkout → Set up → Checks (parallel) → Package |

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
message, the lint violation as a `ruff` test case. **A CI system you haven't seen fail is a CI
system you don't yet trust.**

---

## Appendix B — Command reference

### Docker lifecycle

```bash
docker ps                                    # running containers
docker logs -f jenkins                       # follow controller startup
docker exec -it jenkins bash                 # shell as the jenkins user
docker exec -it -u root jenkins bash         # shell as root (installing packages)
docker stop jenkins && docker start jenkins  # restart
docker restart jenkins                       # same, in one step
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

> Native `.deb`/`.rpm` installs use `/var/lib/jenkins/`. Agents use their own remote root
> (often `/home/jenkins/agent`). Never hardcode a controller path into an agent step — use
> `${WORKSPACE}`.

### Backup and restore

```bash
# Back up the whole volume
docker run --rm \
  -v jenkins_home:/source:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/jenkins-backup-$(date +%F).tar.gz -C /source .

# Restore into a fresh volume
docker run --rm \
  -v jenkins_home:/target \
  -v "$PWD":/backup \
  alpine sh -c 'cd /target && tar xzf /backup/jenkins-backup-2026-07-24.tar.gz'
```

> Restore `secrets/` **together with** `credentials.xml`. Without the master key, every stored
> credential is unrecoverable.

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

### Validate a Jenkinsfile before pushing

```bash
# Requires an API token: your username → Configure → API Token
curl -X POST -u "user:token" \
  -F "jenkinsfile=<Jenkinsfile" \
  http://localhost:8080/pipeline-model-converter/validate
```

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
it. It's deliberately small but exercises every mechanism in Section 19: dependencies, linting,
tests, coverage, packaging, and a health endpoint to smoke-test.

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

```python
```

(Empty file. It makes `tests` a package, which avoids import-path surprises.)

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
pytest==8.3.4
pytest-cov==6.0.0
ruff==0.9.2
build==1.2.2
```

> **Pinned on purpose.** Unpinned dev dependencies mean a tool update can turn your build red
> overnight with no change from you. Pin, then upgrade deliberately.

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "jenkins-sample"
version = "0.1.0"
description = "Sample project for a Jenkins CI pipeline"
requires-python = ">=3.9"
dependencies = []

[tool.setuptools.packages.find]
include = ["app*"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[tool.ruff]
line-length = 100
target-version = "py39"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]

[tool.coverage.run]
source = ["app"]
omit = ["*/tests/*"]
```

> `[tool.setuptools.packages.find]` with `include = ["app*"]` stops setuptools from also packaging
> `tests` into the wheel. Without it, `python -m build` may fail with a "multiple top-level
> packages" error — a common and confusing first failure.

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
> `junit` and `archiveArtifacts` — they should never be committed.

### `Jenkinsfile`

```groovy
pipeline {
    // Using a Docker agent (Sections 12-13). If Python is on the controller
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
        // Needed so `import app` resolves when pytest runs from the workspace root
        PYTHONPATH                    = "${WORKSPACE}"
    }

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
            // Requires the Coverage plugin; delete this line if not installed
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
| Want to add a Docker image build | See Section 19, Stage 3 — needs the Docker Pipeline plugin and a `docker` CLI on the agent |
| Use `main` vs `master` | Set **Branch Specifier** to match |

### Running it locally first

Always confirm the project works outside Jenkins before blaming Jenkins:

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

If any of these fails locally, it will fail in Jenkins for the same reason — and it's far faster to
debug in your own terminal than through a four-minute build cycle.

---

*Verified against Jenkins LTS 2.555.x, July 2026. Version-specific claims — Java baselines, image
tags, plugin recommendations — will age; check the [Jenkins changelog](https://www.jenkins.io/changelog-stable/)
when they matter.*
