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
