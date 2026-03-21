# Spring Boot + GitHub + Jenkins CI/CD POC Guide

A step-by-step guide to build a Spring Boot app, push it to GitHub, and deploy it via Jenkins.

---

## Prerequisites

- Java 17+ and Maven installed
- Git installed and configured
- A GitHub account
- Jenkins installed (locally or on a server)
- Docker (optional but recommended for Jenkins)

---

## Part 1 — Create the Spring Boot Application

### Step 1: Generate the Project

Go to [https://start.spring.io](https://start.spring.io) and configure:

| Field        | Value              |
|--------------|--------------------|
| Project      | Maven              |
| Language     | Java               |
| Spring Boot  | 3.x (latest)       |
| Packaging    | Jar                |
| Java Version | 17                 |
| Dependencies | Spring Web         |

Click **Generate** and unzip the downloaded project.

### Step 2: Add a Simple REST Controller

Create `src/main/java/com/example/demo/HelloController.java`:

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello from Spring Boot POC!";
    }
}
```

### Step 3: Build and Test Locally

```bash
cd demo
./mvnw clean package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

Visit `http://localhost:8080/hello` — you should see the response.

---

## Part 2 — Push to GitHub

### Step 4: Initialize Git Repository

```bash
git init
git add .
git commit -m "Initial Spring Boot POC commit"
```

### Step 5: Create a GitHub Repository

1. Go to [https://github.com/new](https://github.com/new)
2. Name the repo (e.g., `springboot-jenkins-poc`)
3. Leave it **public** (easier for Jenkins access)
4. Do **not** initialize with README (you already have code)

### Step 6: Push Code to GitHub

```bash
git remote add origin https://github.com/<your-username>/springboot-jenkins-poc.git
git branch -M main
git push -u origin main
```

---

## Part 3 — Set Up Jenkins

### Step 7: Install Jenkins

**Option A — Docker (Recommended):**
```bash
docker run -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

**Option B — Local Install:**
Download the installer from [https://www.jenkins.io/download/](https://www.jenkins.io/download/) and follow OS-specific steps.

> Initial admin password is shown in the console log or at `/var/jenkins_home/secrets/initialAdminPassword`.

### Step 8: Configure Jenkins

1. Open Jenkins at `http://localhost:8080`
2. Enter the admin password and complete setup
3. Install **suggested plugins** (includes Git, Maven, Pipeline)
4. Create an admin user

### Step 9: Install Required Plugins

Go to **Manage Jenkins → Plugins → Available** and install:

- **Git Plugin** (usually pre-installed)
- **Maven Integration Plugin**
- **Pipeline Plugin**

---

## Part 4 — Create the Jenkins Pipeline

### Step 10: Add a Jenkinsfile to Your Project

Create a `Jenkinsfile` (no extension) in the **project root**, commit, and push it.

> ⚠️ **Windows Jenkins users:** Use `bat` instead of `sh`. `sh` is Linux-only and will cause `CreateProcess error=2`.

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            when { branch 'main' }
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }
        stage('Test') {
            when { branch 'main' }
            steps {
                bat 'mvn test'
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                bat 'start java -jar target\\demo-0.0.1-SNAPSHOT.jar --server.port=9090'
            }
        }
    }
}
```

```bash
git add Jenkinsfile && git commit -m "Add Jenkinsfile" && git push
```

> 💡 `checkout scm` is preferred over hardcoding the repo URL — it uses the URL from the Jenkins job config automatically.

### Step 11: Create a Pipeline Job in Jenkins

1. **New Item** → name it → select **Pipeline** → OK
2. Under **Pipeline**: set Definition to **Pipeline script from SCM**, SCM to **Git**, paste your repo URL, branch `*/main`, script path `Jenkinsfile`
3. Click **Save**

---

## Part 5 — Run and Verify

### Step 12: Trigger the Pipeline

1. Click **Build Now** on the job page
2. Watch the stages execute in **Stage View**
3. Click any stage to see its console log

### Step 13: Access the App

> ⚠️ Spring Boot and Jenkins both default to port **8080** — they conflict. Always run your app on a different port.

Set the port in `src/main/resources/application.properties`:

```properties
server.port=9090
```

Or pass it at runtime in the Deploy stage (`--server.port=9090`). Then access:

```
http://localhost:9090/hello
```

---

## Part 6 — Auto-Deploy on Git Push (with ngrok)

GitHub webhooks need a publicly accessible Jenkins URL. Since Jenkins is on localhost, use **ngrok** as a tunnel.

### Step 14: Install and Run ngrok

1. Download from [https://ngrok.com/download](https://ngrok.com/download) and sign up (free)
2. Run:

```bash
ngrok http 8080
```

You'll see:
```
Forwarding  https://abc123.ngrok-free.app -> http://localhost:8080
```

Copy the `https://abc123.ngrok-free.app` URL.

> ⚠️ Keep this terminal open while testing. The URL changes every time you restart ngrok on the free plan — update the webhook URL in GitHub when it changes.

### Step 15: Add Webhook in GitHub

1. Go to your repo → **Settings → Webhooks → Add webhook**
2. Fill in:
   - **Payload URL**: `https://abc123.ngrok-free.app/github-webhook/`
   - **Content type**: `application/json`
   - **Trigger**: Just the push event
3. Click **Add webhook**

### Step 16: Enable Trigger in Jenkins

1. Go to your Jenkins job → **Configure**
2. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**
3. Click **Save**

Now every `git push` to `main` will automatically trigger the pipeline.

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot run program "sh"` | Using `sh` on Windows Jenkins | Replace `sh` with `bat` in Jenkinsfile |
| `'mvnw.cmd' is not recognized` | Maven wrapper missing from repo | Use `bat 'mvn ...'` or run `mvn wrapper:wrapper` and push |
| `HTTP 400` on git fetch | Placeholder URL not replaced | Update repo URL in Jenkins job config |
| Jenkins login page on port 8080 | Port conflict with Spring Boot | Run app on a different port e.g. `--server.port=9090` |
| GitHub webhook can't reach Jenkins | Jenkins is on localhost | Use ngrok to expose Jenkins publicly |

---
       
## Summary

```
git push → GitHub → Webhook → ngrok → Jenkins Pipeline
                                            │
                         ┌──────────────────┼──────────────────┐
                      Checkout            Build              Deploy
                     (checkout scm)   (mvn package)    (java -jar :9090)
```

| Part | Tool        | Action                              |
|------|-------------|-------------------------------------|
| 1–3  | Spring Boot | Create and test the app locally     |
| 4–6  | GitHub      | Push source code                    |
| 7–9  | Jenkins     | Install, configure, add plugins     |
| 10–11| Jenkins     | Add Jenkinsfile, create pipeline job|
| 12–13| Jenkins     | Build, run, access on port 9090     |
| 14–16| ngrok       | Auto-trigger pipeline on git push   |




## Part 7 — Webhook 403 Troubleshooting
 
If GitHub webhook shows **Response 403**, work through these fixes in order.
 
### Fix 1: CSRF — "No valid crumb" error
 
Jenkins blocks webhook requests due to CSRF protection.
 
**Option A — Enable proxy compatibility (safer):**
1. Go to **Manage Jenkins → Security → CSRF Protection**
2. Under **Default Crumb Issuer**, check **Enable proxy compatibility**
3. Save → Redeliver the webhook from GitHub
 
**Option B — Disable CSRF fully via Script Console:**
1. Go to **Manage Jenkins → Script Console**
2. Run:
 
```groovy
Jenkins.instance.setCrumbIssuer(null)
Jenkins.instance.save()
```
 
You should see `Result: null` confirming CSRF is disabled.
 
### Fix 2: Authentication — "Authentication required" error
 
Jenkins requires login to accept webhook calls.
 
**Option A — Via UI:**
1. **Manage Jenkins → Security → Authorization**
2. Select **Anyone can do anything** → Save
 
**Option B — Via Script Console:**
 
```groovy
import jenkins.model.*
import hudson.security.*
 
def instance = Jenkins.getInstance()
def strategy = new AuthorizationStrategy.Unsecured()
instance.setAuthorizationStrategy(strategy)
instance.save()
```
 
> ⚠️ Only use "Anyone can do anything" for a local POC. Never in production.
 
### Fix 3: Verify webhook is working
 
Go to GitHub → **Settings → Webhooks** → click your webhook → **Recent Deliveries** → click **Redeliver**.
 
You should see **Response 200** ✅ — the pipeline will now auto-trigger on every push.
 
### Test auto-trigger
 
```bash
echo "test auto deploy" >> README.md
git add .
git commit -m "test auto trigger"
git push origin main
```
 
Watch Jenkins — the build should start within seconds automatically.
 
---

## Part 8 — Checkout Failed "unable to unlink" Error
   
If the Jenkins pipeline fails at the **Checkout SCM** stage with the error:
`error: unable to unlink old 'target/demo-0.0.1-SNAPSHOT.jar': Invalid argument`
 
This happens because the `target/` folder was accidentally pushed to GitHub without a `.gitignore`, and the old JAR file is currently **locked by Windows** because it's still running in the background.
 
### How to fix:
 
1. **Kill the locked process:** Open Command Prompt as **Administrator** (crucial step!) and run:
   ```cmd
   FOR /F "tokens=5" %T IN ('netstat -ano ^| findstr :9090') DO taskkill /F /PID %T
   ```
2. **Ignore and remove the locked folder:** Run these commands in your project directory:
   ```bash
   echo target/ > .gitignore
   git rm -r --cached target
   git add .
   git commit -m "Remove target folder and fix gitignore"
   git push origin main
   ```
3. Click **Build Now** in Jenkins (Future auto-deploys will work flawlessly!).