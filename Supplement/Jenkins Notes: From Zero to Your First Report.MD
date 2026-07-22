# Jenkins Notes: From Zero to Your First Report

---

## Table of Contents

1. [What Problem Does Jenkins Actually Solve?](#1.What_Problem_Does_Jenkins_Actually_Solve?)
2. [What Is Jenkins?](#2.what_is_jenkins?)
3. [Vocabulary](#3.Vocabulary)
4. [Setting Up Jenkins on Your Computer](#4-setting-up-jenkins-on-your-computer)
5. [Logging In and First-Time Setup](#5-logging-in-and-first-time-setup)
6. [Your First Job — the Simple Way](#6-your-first-job--the-simple-way)
7. [The "Real" Way: Pipelines and Jenkinsfiles](#7-the-real-way-pipelines-and-jenkinsfiles)
8. [A Full Practice Task: Build → Test → Report](#8-a-full-practice-task-build--test--report)
9. [Reading the Final Report](#9-reading-the-final-report)
10. [Extra Plugins Worth Knowing About](#10-extra-plugins-worth-knowing-about)
11. [When Something Breaks](#11-when-something-breaks)
12. [Good Habits (Best Practices)](#12-good-habits-best-practices)
13. [Quick Recap](#13-quick-recap)

---

## 1. What_Problem_Does_Jenkins_Actually_Solve?

Imagine you're on a team of 5 developers, all writing code for the same app. Every day, everyone makes changes and combines their work.

Without any automation, here's what usually goes wrong:
- Someone's code works on their own laptop, but breaks the moment it's combined with everyone else's.
- Nobody notices a bug until days later, right before the product is supposed to ship.
- Every release involves someone manually running a checklist of commands — easy to forget a step.
- Feedback ("did my change break anything?") takes hours or days instead of minutes.

**Jenkins exists to remove all of that manual, error-prone work.** It watches your code, and every time something changes, it automatically:
1. Downloads the latest code
2. Builds it (compiles it, packages it)
3. Runs the tests
4. Tells you (and your team) whether it passed or failed
5. Optionally, packages and ships it somewhere

>**Term Explained — "Build"**
>"Building" software means turning your source code into something runnable — compiling C/C++ into a binary, packaging Python files into a distributable package, etc.

>**Term Explained — "CI/CD"**
 >**CI = Continuous Integration**: every time someone changes code, it's automatically combined ("integrated") with everyone else's code and tested — so problems surface within minutes.
 >**CD = Continuous Delivery** (packaged and ready to release, with a human clicking "deploy") or **Continuous Deployment** (fully automatic release).
>Jenkins is the tool that *runs* this automation; CI/CD is the *practice/philosophy* it's used for.

---

## 2. What_is_jenkins?

Jenkins is a piece of software — a **server application** — that you install on a computer (your laptop, or a company server). Once running, you interact with it through a **web browser**, except this "website" is running on your own machine.

>**Term Explained — "Server"**
>A "server" here just means a program that stays running in the background, waiting to respond to requests — in Jenkins' case, waiting for you to click buttons in your browser, or waiting for your code repository to say "hey, new code was just pushed, please build it." It does *not* have to be a physical server in a data center — it can be your own laptop.


---

## 3. Vocabulary

| Term                    | Plain-English Meaning                                                                                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Job / Project**       | A task you've told Jenkins to do — e.g., "build my app."                                                                                                                 |
| **Build**               | One single run/execution of a job. Builds are numbered #1, #2, #3... forever increasing.                                                                                 |
| **Freestyle Project**   | A job you configure by clicking through a web form — no coding required. Good for learning.                                                                              |
| **Pipeline**            | A job whose steps are written as *code* (in a file called a Jenkinsfile), rather than clicked together in a form.                                                        |
| **Jenkinsfile**         | A plain text file, usually stored alongside your project's code, that describes what the Pipeline should do, step by step.                                               |
| **Stage**               | One labeled phase inside a Pipeline (e.g., "Build," "Test," "Deploy"). Stages show up as boxes in Jenkins' visual timeline.                                              |
| **Node / Agent**        | Agents are the "hands" doing the work — which can be the same machine, or separate ones.                                                                                 |
| **Executor**            | A single "slot" on a node that can run one build at a time. If a node has 2 executors, it can run 2 builds simultaneously.                                               |
| **Workspace**           | The folder on disk where Jenkins downloads your project's code and does its work for one specific job.                                                                   |
| **Plugin**              | An installable add-on that gives Jenkins new abilities (e.g., understanding Git, sending Slack messages, publishing HTML reports).                                       |
| **Artifact**            | Any file a build produces that's worth keeping — a compiled program, a `.zip` file, a report, a log.                                                                     |
| **Trigger**             | The event that starts a build — clicking a button manually, a schedule (like "every night at 2am"), or automatically whenever new code is pushed (called a **webhook**). |
| **Repository ("repo")** | A project's folder of code, tracked by a version control system like **Git**, usually hosted on a site like GitHub or GitLab.                                            |
| **Console Output**      | The full text log of everything that happened during a build — your first stop when troubleshooting.                                                                     |

> **Term Explained — "Git" / "Version Control"**
> Git is a separate tool (not part of Jenkins) that tracks every change ever made to a project's code, letting multiple people collaborate without overwriting each other's work. Jenkins doesn't replace Git — it *reads from* a Git repository to get the latest code before building it.

---

## 4. Setting Up Jenkins on Your Computer

There are three common ways to install Jenkins. If you're just learning, **Option A (Docker)** is strongly recommended — it's the fastest, and it's easy to completely remove afterward without leaving anything on your system.

> **📌 Term Explained — "Docker" / "Container"**
> Docker lets you run software inside an isolated little bundle called a **container** — like a lightweight, disposable virtual computer that already has everything the software needs pre-installed. Instead of manually installing Java, Jenkins, and its dependencies on your real computer, you just tell Docker "run the Jenkins container," and it handles everything, in a way that's easy to delete cleanly afterward.

### 4.1 Option A: Run Jenkins with Docker (Recommended for Learning)

**Prerequisite:** Docker Desktop (or Docker Engine) installed on your computer.

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

Let's break down every part of this command, since it looks intimidating at first:

- `docker run` → "start a new container"
- `-d` → **detached mode**: run in the background, so your terminal is free to keep using
- `--name jenkins` → give this container the nickname "jenkins" so it's easy to refer to later
- `-p 8080:8080` → **port mapping**. Explained below.
- `-p 50000:50000` → a second port, used only if you later connect additional Agent machines
- `-v jenkins_home:/var/jenkins_home` → **volume mount**. Explained below.
- `jenkins/jenkins:lts` → the name of the container image to use — the official Jenkins "Long Term Support" (stable) version

> **📌 Term Explained — "Port"**
> A port is like a numbered door on your computer that a specific program listens through. Web servers commonly use port `8080` for a browser-based tool like Jenkins. `-p 8080:8080` means "connect port 8080 on my real computer to port 8080 inside the container," so that visiting `http://localhost:8080` in your browser reaches Jenkins.

> **📌 Term Explained — "Volume" (Persistent Storage)**
> Containers are normally *temporary* — if you delete one, everything inside disappears. A **volume** is a way to save specific data (like your Jenkins configuration and job history) outside the container, so it survives even if you delete and recreate the container.

After running this command, open your browser and go to:

```
http://localhost:8080
```

> **📌 Term Explained — "localhost"**
> `localhost` is a special address that always means "this same computer I'm using right now." So `http://localhost:8080` means "the web service running on port 8080 of my own machine" — not somewhere on the internet.

### 4.2 Option B: Native Install on Ubuntu/Debian Linux

```bash
# 1. Install Java — Jenkins is written in Java and needs it to run
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre

# 2. Add Jenkins' official package source
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  "https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# 3. Install Jenkins itself
sudo apt update
sudo apt install -y jenkins

# 4. Start Jenkins and make it start automatically on every boot
sudo systemctl enable jenkins
sudo systemctl start jenkins

# 5. Confirm it's running
sudo systemctl status jenkins
```

Then visit `http://localhost:8080` (or your server's IP address followed by `:8080`).

### 4.3 Option C: Native Install on Windows

1. Download the `.msi` installer from Jenkins' official download page.
2. Double-click it and follow the wizard — it will ask where to install Jenkins and which port to use (default: `8080`).
3. Jenkins installs itself as a **Windows Service**, meaning it starts automatically in the background whenever your PC turns on — you don't need to manually launch it.
4. Open `http://localhost:8080` in your browser.

---

## 5. Logging In and First-Time Setup

The very first time you open Jenkins, it won't let you in without an **initial admin password** — this is a security step to make sure only you (the person who just installed it) can set it up.

### 5.1 Finding the Password

**If you used Docker:**
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**If you used the Ubuntu/Debian native install:**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**If you used Windows:**
The setup screen will show you the exact file path, typically something like:
```
C:\Program Files\Jenkins\secrets\initialAdminPassword
```

Copy the long string of characters it prints out, and paste it into the "Unlock Jenkins" screen in your browser.

### 5.2 Installing Plugins

Next, Jenkins asks whether to install **Suggested Plugins** or pick manually.

> **📌 Beginner Tip:** Choose "Install suggested plugins." This gives you Git support, Pipeline support, and other essentials — you don't need to understand each one yet, just let it install.

### 5.3 Creating Your Admin Account

Fill in a username, password, full name, and email. This is *your* login — separate from the temporary initial admin password.

### 5.4 Confirming the Jenkins URL

Jenkins will show a URL it will use to refer to itself (useful for email links, webhook callbacks, etc.). For learning on your own computer, leave the default (`http://localhost:8080/`) as-is.

### 5.5 You're In!

You'll now see the **Dashboard** — the home page where all your jobs will eventually be listed.

---

## 6. Your First Job — the Simple Way

Let's create the simplest possible job, so you can see the whole cycle (configure → run → check output) before anything gets complicated.

### Step-by-Step

1. On the Dashboard, click **New Item** (top-left).
2. Type a name, e.g., `hello-jenkins`.
3. Select **Freestyle project**, then click **OK**.
4. Scroll down to the **Build Steps** section.
5. Click **Add build step** → choose **Execute shell** (on Linux/macOS/Docker) or **Execute Windows batch command** (on Windows).

   > **📌 Term Explained — "Shell" / "Command Line"**
   > A shell (or command line) is a text-based way of telling your computer to do things, instead of clicking icons. `echo "hello"` simply prints the word "hello" to the screen — it's the programming equivalent of a "hello world" test.

6. Type this into the text box:
   ```bash
   echo "Hello, Jenkins! This is my very first automated build."
   date
   ```
7. Click **Save**.
8. On the job's page, click **Build Now** (left sidebar).
9. Under **Build History**, click the new build number (e.g., `#1`).
10. Click **Console Output** to see exactly what happened.

Congratulations — you just automated something for the first time. Everything from here is the same basic loop: **configure → trigger → observe result**, just with more steps involved.

---

## 7. The "Real" Way: Pipelines and Jenkinsfiles

Freestyle jobs (what you just did) are fine for learning, but almost no real company uses them for actual projects, because:

- The configuration lives *only* inside Jenkins' database — not in your code, so nobody can review changes to it like they review code.
- It doesn't scale well to multi-step processes (build → test → deploy → notify).

**Pipelines** solve this by describing the entire process as *code*, in a file usually named `Jenkinsfile`, which lives right next to your project's source code in its Git repository.

### 7.1 Declarative vs. Scripted Pipelines

There are two writing styles:

- **Declarative Pipeline** — a more structured, easier-to-read format. **Recommended for beginners.**
- **Scripted Pipeline** — full programming flexibility using the Groovy language, but a steeper learning curve.

We'll use Declarative syntax throughout these notes.

### 7.2 A Simple Pipeline Example

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                // checkout scm   // uncomment this when connected to a real Git repo
            }
        }

        stage('Build') {
            steps {
                echo 'Building the project...'
                sh 'echo "Simulating a build step"'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'echo "Simulating tests passing"'
            }
        }

        stage('Report') {
            steps {
                echo 'Generating reports...'
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished. Cleaning up...'
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed. Check the logs.'
        }
    }
}
```

> **📌 Term Explained — "agent any"**
> This tells Jenkins "run this on whatever machine is available" — it doesn't matter which specific node/agent handles it.

> **📌 Term Explained — "post" block**
> This runs *after* all the stages finish, no matter what happened — useful for cleanup, or sending a notification whether the build succeeded or failed.

### 7.3 Creating the Pipeline Job

1. Click **New Item** → name it, e.g., `hello-pipeline`.
2. Select **Pipeline** → click **OK**.
3. Scroll to the **Pipeline** section near the bottom.
4. For **Definition**, choose one of:
   - **Pipeline script** — paste the Groovy code directly into a text box (fastest, good for learning).
   - **Pipeline script from SCM** — point Jenkins at a Git repository that already contains a `Jenkinsfile` (the realistic, professional approach).
5. Click **Save**, then **Build Now**.
6. Watch the **Stage View** — a horizontal row of boxes showing each stage turning green (success) or red (failure) as it runs.

---

## 8. A Full Practice Task: Build → Test → Report

Now let's do something closer to a real project: a small Python app, with actual unit tests, producing a real report.

> **📌 Term Explained — "Unit Test"**
> A unit test is a small, automated check that verifies one specific piece of code behaves correctly — e.g., "does the `add()` function return 5 when given 2 and 3?" Instead of a human manually checking this every time, the computer checks it in milliseconds, every single build.

### 8.1 Sample Project Files

```
my-python-app/
├── app.py
├── test_app.py
├── requirements.txt
└── Jenkinsfile
```

**app.py**
```python
def add(a, b):
    return a + b

if __name__ == "__main__":
    print(add(2, 3))
```

**test_app.py**
```python
import unittest
from app import add

class TestApp(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)

if __name__ == "__main__":
    unittest.main()
```

**requirements.txt**
```
pytest
pytest-html
```

> **📌 Term Explained — "requirements.txt"**
> This is a standard Python file listing which external packages/libraries your project needs. `pip install -r requirements.txt` tells Python's package manager to read this list and install everything on it.

### 8.2 The Jenkinsfile for This Task

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'pytest --junitxml=report.xml --html=report.html --self-contained-html'
            }
        }

        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: 'report.html', fingerprint: true
                junit 'report.xml'
            }
        }
    }

    post {
        always {
            echo 'Task complete — check the Test Result and archived report.'
        }
    }
}
```

> **📌 Term Explained — "JUnit XML"**
> JUnit is originally a Java testing framework, but its **XML report format** became a universal standard that almost every test tool (including Python's pytest) can also produce. Jenkins understands this format natively and turns it into graphs and pass/fail tables automatically — that's what the `junit 'report.xml'` line does.

> **📌 Term Explained — "Fingerprint"**
> `fingerprint: true` tells Jenkins to calculate a unique checksum for the archived file, so Jenkins can track exactly which build produced which file, even if the same file gets copied or reused elsewhere later.

### 8.3 What Each Stage Actually Does

| Stage | In Plain Language |
|---|---|
| Checkout | "Go download the latest version of the code from Git." |
| Install Dependencies | "Install everything listed in requirements.txt so the code can actually run." |
| Run Tests | "Run all the unit tests, and while doing so, save the results as two report files." |
| Archive Results | "Keep a copy of the HTML report forever (or until manually deleted), and show the JUnit results in Jenkins' built-in report screen." |

### 8.4 Running It

1. Push this folder (including the `Jenkinsfile`) to a Git repository (e.g., on GitHub).
2. Create a Pipeline job using **Pipeline script from SCM**, pointing at that repository.
3. Click **Build Now**.
4. Watch the **Stage View** — each box turns green or red as its stage completes.

---

## 9. Reading the Final Report

This is the part that actually matters day-to-day: knowing *where to look* after a build finishes.

### 9.1 The Built-in Test Result Page

Because the Jenkinsfile included `junit 'report.xml'`, Jenkins automatically creates:

- A **Test Result Trend** graph on the job's main page (pass/fail count over time)
- A **Test Result** page per build, listing every individual test, and whether it passed, failed, or was skipped
- The ability to click into any single test and see its pass/fail history across many builds

**Where to find it:** Job page → click a build number → **Test Result**

### 9.2 Custom HTML Reports

To see the fuller `report.html` (from `pytest-html`) directly inside Jenkins' interface (not just as a downloadable file):

1. Install the **HTML Publisher Plugin** (via **Manage Jenkins → Plugins**).
2. Add this stage to your Jenkinsfile:

```groovy
stage('Publish HTML Report') {
    steps {
        publishHTML(target: [
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: '.',
            reportFiles: 'report.html',
            reportName: 'Test HTML Report'
        ])
    }
}
```

3. After the next build, a new link labeled **Test HTML Report** appears in the job's left sidebar — clicking it shows the full report right inside Jenkins.

### 9.3 Archived Artifacts

Any file saved with `archiveArtifacts` becomes downloadable from:

**Job page → build number → Build Artifacts**

This is how you'd keep, for example, a compiled program, a `.zip` package, or a report — permanently tied to the exact build that produced it.

### 9.4 Console Output

Every single build has a **Console Output** — a scrolling text log of literally everything that happened. This is always the first place to look when a build fails and you don't know why.

### 9.5 Notifications (Optional)

You can have Jenkins automatically email or Slack-message a summary after every build:

```groovy
post {
    always {
        emailext (
            subject: "Build ${currentBuild.currentResult}: Job ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Check the full report: ${env.BUILD_URL}",
            to: 'team@example.com'
        )
    }
}
```

### 9.6 What a Complete "Report" Looks Like in Practice

By the time you're done, opening a finished build in Jenkins should show you:

1. ✅ A green/red status per stage (visual Stage View)
2. 📊 A pass/fail test trend graph
3. 📄 A clickable HTML report inside Jenkins
4. 📦 A downloadable artifact
5. 📧 (Optional) an emailed or Slack-posted summary

This combination — not just one single file — is what most teams mean when they say "check the CI report."

---

## 10. Extra Plugins Worth Knowing About

| Plugin Name | What It Adds |
|---|---|
| Git Plugin | Lets Jenkins pull code from Git repositories |
| Pipeline Plugin | Enables Jenkinsfile-based Pipelines (comes with "suggested plugins") |
| JUnit Plugin | Understands JUnit-format XML reports and turns them into graphs/tables |
| HTML Publisher Plugin | Displays custom HTML reports inside the Jenkins interface |
| Blue Ocean | A more modern, visual interface for viewing Pipelines |
| Email Extension Plugin | More customizable email notifications than the built-in default |
| Slack Notification Plugin | Posts build results directly into a Slack channel |
| Credentials Binding Plugin | Securely stores and injects secrets (passwords, API keys) without exposing them in logs |
| Docker Pipeline Plugin | Lets pipeline steps run *inside* Docker containers, for consistent build environments |

**Where to manage these:** Manage Jenkins → Plugins → Available plugins

---

## 11. When Something Breaks

| Symptom | Likely Cause | What to Try |
|---|---|---|
| Can't open `localhost:8080` | Jenkins isn't actually running | Check with `docker ps` (Docker) or `sudo systemctl status jenkins` (native) |
| "Permission denied" during a shell step | Jenkins' user account doesn't have file access | Check folder/file ownership in the workspace |
| Fails at `checkout scm` | No Git credentials configured | Go to **Manage Jenkins → Credentials** and add them |
| No tests show in Test Result | Wrong file path in the `junit` step, or the report file wasn't generated | Confirm the exact filename/path matches what pytest actually produced |
| Build sits in "queue" forever | No available agent/executor | Check **Manage Jenkins → Nodes**; add an agent or increase executor count |
| Console Output looks outdated | Browser cache | Hard-refresh the page (Ctrl+Shift+R / Cmd+Shift+R) |
| "This Jenkins instance appears to be offline" | The underlying service or container crashed | `docker start jenkins` or `sudo systemctl restart jenkins` |

> **📌 Extra Knowledge — Reading error messages**
> Beginners often skim past the actual error text because it looks scary. In almost every case, the *last few lines* of Console Output contain the actual cause — search that exact error text online, and you'll usually find someone else who hit the same issue.

---

## 12. Good Habits (Best Practices)

- **Prefer Pipelines (Jenkinsfile) over Freestyle jobs** for anything beyond quick experiments — Pipelines are reviewable, version-controlled code.
- **Keep the Jenkinsfile inside the project's own repository**, so pipeline changes get reviewed alongside code changes, not hidden inside Jenkins' database.
- **Never hardcode passwords or API keys** directly into a Jenkinsfile — use the Credentials Manager instead.
- **Keep pipelines fast.** Slow builds discourage people from committing often, which defeats the whole purpose of CI.
- **Fail fast.** Put cheap, quick checks (like code style/lint checks) early in the pipeline, before expensive steps like full test suites.
- **Only archive what's actually useful.** Archiving huge files on every single build wastes disk space over time.
- **Name stages clearly.** `stage('Run Unit Tests')` tells you far more at a glance than `stage('Step 2')`.
- **Use webhooks, not polling.** A webhook means Git tells Jenkins "new code just arrived" instantly, instead of Jenkins repeatedly checking "anything new yet?" on a timer.

---

## 13. Quick Recap

By working through these notes, you now understand:

1. **Why** CI/CD tools like Jenkins exist — to remove manual, error-prone build/test/release work
2. **What** Jenkins is — a self-hosted automation server you access through a web browser
3. **How to install it** — via Docker (recommended for learning), or natively on Linux/Windows
4. **The core vocabulary** — jobs, builds, pipelines, stages, agents, artifacts, triggers
5. **How to build a Freestyle job**, and why real teams prefer Pipelines instead
6. **How to write a Jenkinsfile** that checks out code, installs dependencies, runs tests, and archives results
7. **Where to find the final report** — Test Result page, HTML Publisher report, archived artifacts, and optional notifications

### Suggested Next Topics to Explore

- **Blue Ocean** — a more visual way to view Pipelines
- **Multibranch Pipelines** — Jenkins automatically creates a pipeline for every Git branch
- **Distributed builds** — running jobs across multiple remote agent machines
- **Docker-based builds** — running each build step inside its own container for consistency
- **Shared Libraries** — reusing common pipeline code across many projects

---

*Tip for studying: don't just read this once — actually spin up Jenkins with Docker and click through every step yourself. Muscle memory from doing it once is worth more than reading it three times.*
