# Jenkins Beginner's Guide: From Basics to Practical Application

We will cover what Jenkins is, how to set it up cleanly using Docker, and how to create a complete, real-world automated pipeline task.

## Catalog
1. [Introduction](#Introduction)
2. [Setting](#Setting)

## Introduction
**What is Jenkins?**
Before we start writing code, let's understand what problem Jenkins solves.

In traditional software development, after writing code, a developer has to manually perform several repetitive steps:
1. Compile the code.
2. Run unit tests to make sure nothing is broken.
3. Package the application into a deployable format (like a ZIP or a Docker image).
4. Log into a remote server and upload the package to deploy it.

Doing this manually every single time is time-consuming and prone to human error. **Jenkins is an open-source automation server that eliminates this manual labor.**

Jenkins achieves this through a concept known as **CI/CD**:
- **Continuous Integration (CI):** Every time you push new code to a repository (like GitHub), Jenkins automatically detects the change, downloads the code, builds it, and runs tests immediately.
- **Continuous Deployment (CD):** If all tests pass successfully, Jenkins can automatically push the new version of your application straight to your testing or production servers.

---

## Setting
### Prerequisites
* You must have **Docker Desktop** installed and running on your computer.

### Step 1: Run the Jenkins Container
Open your terminal (Terminal on Mac/Linux, or Command Prompt / PowerShell on Windows), copy the exact command below, and press Enter:

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  --name my-jenkins \
  jenkins/jenkins:lts-jdk17
```

💡 Explaining the Command Parameters:
-d: Runs the container in the background (detached mode). You can close your terminal and Jenkins will keep running.

-p 8080:8080: Maps port 8080 of the container to port 8080 on your local computer. This allows you to access the Jenkins website dashboard.

-p 50000:50000: Used internally by Jenkins to communicate with advanced build agents. Keep it at default.

-v jenkins_home:/var/jenkins_home: Creates a persistent data volume named jenkins_home. This ensures your accounts, jobs, and settings are saved on your hard drive even if you stop or delete the Docker container.

--name my-jenkins: Gives a friendly, easy-to-remember name to your container.

jenkins/jenkins:lts-jdk17: Downloads the official Long-Term Support (LTS) version of Jenkins running on Java 17.

Step 2: Unlock Jenkins
Open your web browser and go to: http://localhost:8080

You will be greeted by an Unlock Jenkins screen asking for an Administrator password.

To find this password, go back to your terminal and run the following command to print the container logs:

Bash
docker logs my-jenkins
Scroll down through the log output until you find a long string of numbers and letters trapped between two rows of asterisks (*****************).

Copy that alphanumeric password, paste it into the browser window, and click Continue.

Step 3: Complete Initial Configurations
Install Plugins: On the next screen, click the option on the left side: "Install suggested plugins". Jenkins will automatically install the most common tools like Git and Pipeline support. Wait a few minutes for the progress bars to finish.

Create Admin User: Fill out the registration form to create your first primary account. Choose a username, password, and email address that you will remember easily.

Instance Configuration: Click Save and Continue on the default URL screen, then click Start using Jenkins. You will now see the clean main dashboard interface.

3. Understanding Job Types: Freestyle vs. Pipeline
In Jenkins, every automation task you want to run is defined inside an Item (also commonly called a Job). As a beginner, you will hear about two main types of jobs:

Type A: Freestyle Project
How it works: You configure everything visually by checking boxes, clicking dropdown menus, and filling out text forms on the web browser UI.

The Problem: If your build process becomes complex, the webpage becomes massive, unreadable, and impossible to back up or track changes efficiently.

Type B: Pipeline 🌟 (The Modern Industry Standard)
How it works: You define your entire automation process using code inside a text file called a Jenkinsfile written in a language called Groovy.

Why it is better: This follows the concept of Pipeline as Code. Because your pipeline is just text, you can save it inside your Git project repository alongside your application code. This allows you to track exactly who changed the build process and when.

4. Hands-on Lab: Creating Your First Automated Task
Let's build a real Declarative Pipeline task that simulates a professional continuous delivery workflow containing four distinct stages:
Checkout ➡️ Build ➡️ Test ➡️ Deploy.

Step 1: Create a New Job
On the left sidebar of your Jenkins home dashboard, click New Item.

Enter a clear name for your task, such as: my-first-pipeline-task.

Select Pipeline from the list of job categories.

Click the OK button at the bottom of the page.

Step 2: Paste the Pipeline Script
You are now inside the configuration page for your new job. Scroll all the way down to the Pipeline section.

Ensure the Definition dropdown menu is set to Pipeline script.

Clear out any existing placeholder code in the Script box, and paste the following complete Declarative Pipeline code:

Groovy
pipeline {
    // This tells Jenkins to run this automation workflow on any available machine node
    agent any 

    // This section contains the chronological steps of your automated workflow
    stages {
        
        stage('Checkout') {
            steps {
                echo '=== STEP 1: DOWNLOADING CODE ==='
                echo 'Connecting to Git repository and pulling the latest commits...'
                // In a real project, you would write: git '[https://github.com/user/repo.git](https://github.com/user/repo.git)'
            }
        }

        stage('Build') {
            steps {
                echo '=== STEP 2: COMPILING APPLICATION ==='
                echo 'Compiling source code and assembling binary files...'
                // For Node.js projects, you would write: sh 'npm run build'
                // For Java projects, you would write: sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                echo '=== STEP 3: RUNNING AUTOMATED TESTS ==='
                echo 'Executing unit tests and analyzing code quality...'
                // For Python projects, you would write: sh 'pytest'
            }
        }

        stage('Deploy') {
            steps {
                echo '=== STEP 4: DEPLOYING TO ENVIRONMENT ==='
                echo 'All tests passed successfully! Uploading build packages to the production server...'
            }
        }
    }

    // The Post section triggers actions conditionally after the execution finishes
    post {
        always {
            echo '========================================'
            echo 'Cleaning up the build workspace and temporary storage directories...'
        }
        success {
            echo '🎉 SUCCESS: The entire automated pipeline has finished without any errors!'
        }
        failure {
            echo '❌ ERROR: The pipeline failed at some point. Please read the log output below to debug.'
        }
    }
}
Click the Save button in the lower-left corner of the interface.

Step 3: Run the Task & Analyze Results
You will be redirected to your pipeline job's specific home screen. Click Build Now on the left menu.

Under the Build History panel in the lower-left corner, an item labeled #1 will appear with a blinking progress indicator.

In the center of the screen, the Stage View matrix will generate automatically. You will see individual blocks for Checkout, Build, Test, and Deploy turn solid green one by one as they succeed.

To view what happened under the hood, hover over build #1 in the history panel, click the small dropdown arrow that appears next to it, and select Console Output. This is where you can read every single detail and see the text strings printed by the echo commands.

5. Essential Best Practices for Beginners
Once you feel comfortable executing basic manual pipelines, you will need to apply these two critical real-world strategies:

Tip 1: Hooking Up Automated Git Triggers
Instead of logging into Jenkins and clicking "Build Now" manually, you can make Jenkins trigger automatically whenever a developer executes a git push command.

Install the official GitHub Integration plugin via the Jenkins plugin center.

Go to your pipeline job configuration page, find the Build Triggers block, and check the box for GitHub hook trigger for GITScm polling.

Open your GitHub Repository in your browser, go to Settings ➡️ Webhooks, and click Add webhook.

Set the Payload URL field to: http://<YOUR_PUBLIC_JENKINS_IP_OR_URL>:8080/github-webhook/. Leave the content type as JSON, and save it. GitHub will now automatically alert Jenkins whenever code updates.

Tip 2: Managing Sensitive Credentials Safely
Never write passwords, API secret tokens, or private SSH server keys as plain text inside your pipeline scripts. If you check that script into GitHub, anyone can steal your data.

Go to your main Jenkins dashboard page, click Manage Jenkins ➡️ Credentials.

Click Global and add a new credential. Select the kind of secret it is (e.g., Secret text or Username with password), assign your secret values, and give it a distinctive identifier name under ID (such as production-db-secret).

You can now inject that secret safely into your automated pipeline code without exposing it in the logs by utilizing the following block structure inside your Jenkinsfile:

Groovy
environment {
    // This securely fetches the credential value and links it to an environment variable
    DATABASE_PASSWORD = credentials('production-db-secret')
}
