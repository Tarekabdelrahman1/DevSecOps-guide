# DevSecOps Lab 01 — Secrets Detection with Gitleaks

## Objective

This lab demonstrates the security risks of storing secrets in source code and Git history, and teaches how to detect them using **Gitleaks**.

By the end of this lab, you will be able to:

* Detect secrets in local files.
* Detect secrets in Git history.
* Understand the difference between `gitleaks dir` and `gitleaks git`.
* Understand the importance of exit codes in CI/CD pipelines.
* Understand why deleting a secret from a file does not necessarily remove it from Git history.
* Understand why real exposed secrets must be revoked or rotated.

---

## Tools Used

* Ubuntu VM
* Git
* Docker
* Gitleaks

---

## Lab Structure

```text
devsecops-labs/
└── 01-secrets-gitleaks/
    ├── .git/
    ├── .gitignore
    └── app.env.example
```

During the lab, we will also create:

```text
app.env
```

This file will intentionally contain a fake secret.

Later, we will delete the file to simulate a developer attempting to remove an exposed credential after it has already been committed to Git.

---

# 1. Verify the Prerequisites

Check that Git is installed:

```bash
git --version
```

Check that Docker is installed:

```bash
docker --version
```

Verify that the Docker Engine is running:

```bash
docker info > /dev/null && echo "Docker is working"
```

Expected output:

```text
Docker is working
```

If Docker returns a permission or daemon-related error, resolve that issue before continuing.

---

# 2. Download Gitleaks

Pull the Gitleaks Docker image:

```bash
docker pull ghcr.io/gitleaks/gitleaks:latest
```

Verify that Gitleaks runs successfully:

```bash
docker run --rm ghcr.io/gitleaks/gitleaks:latest version
```

The command should return the installed Gitleaks version.

---

# 3. Create the Lab Environment

Create the lab directory:

```bash
mkdir -p ~/devsecops-labs/01-secrets-gitleaks
```

Navigate to the directory:

```bash
cd ~/devsecops-labs/01-secrets-gitleaks
```

Initialize a Git repository:

```bash
git init
```

Configure a Git identity for this repository only:

```bash
git config user.name "DevSecOps Lab"
```

```bash
git config user.email "devsecops-lab@example.local"
```

Verify the repository status:

```bash
git status
```

---

# 4. Create a Fake Secret

> **Security Note:** Only use fake credentials in this lab. Never place real passwords, API keys, access tokens, or cloud credentials in a test repository.

Create an environment file containing a fake AWS Access Key:

```bash
cat > app.env <<'EOF'
APP_ENV=development
AWS_ACCESS_KEY_ID=AKIAZ2X3C4V5B6N7M2Q3
EOF
```

Display the file:

```bash
cat app.env
```

Expected content:

```text
APP_ENV=development
AWS_ACCESS_KEY_ID=AKIAZ2X3C4V5B6N7M2Q3
```

The value is intentionally formatted to resemble an AWS Access Key so that Gitleaks can detect it.

---

# 5. Scan the Current File with Gitleaks

Run a direct scan against the file:

```bash
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  dir -v /repo/app.env
```

Gitleaks may produce output similar to:

```text
RuleID: aws-access-token
File: app.env
Line: 2
Secret: ...
```

Important fields include:

* `RuleID`: The Gitleaks detection rule that was triggered.
* `File`: The file containing the detected secret.
* `Line`: The line where the secret was found.
* `Secret`: The value identified as potentially sensitive.

In a real CI/CD environment, avoid unnecessarily exposing detected secret values in pipeline logs.

---

# 6. Check the Exit Code

Run the scan again:

```bash
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  dir -v /repo/app.env
```

Immediately check the exit code:

```bash
echo $?
```

When a leak is detected, you should typically see:

```text
1
```

This behavior is extremely useful in CI/CD pipelines.

For example:

```text
Gitleaks Scan
     |
     v
 Exit Code
   /    \
  0      1
  |      |
Pass    Fail
         |
         v
   Stop Pipeline
```

A security scanner can therefore act as a **Security Gate**.

If secrets are detected, the pipeline can fail before vulnerable code progresses further through the software delivery lifecycle.

---

# 7. Add the Secret to Git History

Add the vulnerable file to Git:

```bash
git add app.env
```

Commit it:

```bash
git commit -m "Add application configuration"
```

View the Git history:

```bash
git log --oneline
```

At this point, the secret has become part of the repository's **Git history**.

The situation now looks like this:

```text
app.env
   |
   v
git add
   |
   v
git commit
   |
   v
Git History
   |
   v
Secret Stored
```

---

# 8. Scan Git History

Run Gitleaks against the Git repository:

```bash
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  git -v /repo
```

You may see information similar to:

```text
RuleID:
File:
Line:
Commit:
Author:
Date:
```

The `Commit` field is particularly important.

It indicates that Gitleaks has associated the detected secret with a specific commit in Git history.

This is different from simply detecting a secret in the current version of a file.

---

# 9. Simulate Removing the Secret

Now simulate a developer discovering the mistake and deleting the vulnerable file:

```bash
rm app.env
```

Add the local environment file to `.gitignore`:

```bash
echo "app.env" > .gitignore
```

Create a safe environment template:

```bash
cat > app.env.example <<'EOF'
APP_ENV=development
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
EOF
```

Stage the changes:

```bash
git add -A
```

Commit the remediation:

```bash
git commit -m "Remove hardcoded credentials"
```

Inspect the current directory:

```bash
ls -la
```

The file:

```text
app.env
```

should no longer exist in the current working directory.

A developer might now assume that the security problem has been resolved.

However, the secret may still exist in previous commits.

---

# 10. Scan the Current Safe File

Scan the new environment template:

```bash
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  dir -v /repo/app.env.example
```

The previously hardcoded secret should no longer be detected in this file.

The current working directory is therefore clean from that specific secret.

---

# 11. Scan Git History Again

Run the Git history scan again:

```bash
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  git -v /repo
```

Even though `app.env` was deleted from the current working directory, Gitleaks may still detect the secret in an older commit.

The important concept is:

```text
Current Working Directory
        |
        v
Secret Removed
        |
       OK

Git History
        |
        v
Old Commit
        |
        v
Secret Still Exists
```

This demonstrates one of the most important concepts in secrets management:

> **Deleting a secret from the current version of the source code does not automatically remove it from Git history.**

---

# Complete Lab Command Reference

The following commands contain the complete lab workflow in execution order.

```bash
# ==========================================
# DevSecOps Lab 01 - Gitleaks
# ==========================================

# 1. Check prerequisites
git --version
docker --version
docker info > /dev/null && echo "Docker is working"

# 2. Download Gitleaks
docker pull ghcr.io/gitleaks/gitleaks:latest

# 3. Verify Gitleaks
docker run --rm ghcr.io/gitleaks/gitleaks:latest version

# 4. Create lab directory
mkdir -p ~/devsecops-labs/01-secrets-gitleaks
cd ~/devsecops-labs/01-secrets-gitleaks

# 5. Initialize Git repository
git init

# 6. Configure Git identity locally
git config user.name "DevSecOps Lab"
git config user.email "devsecops-lab@example.local"

# 7. Check Git status
git status

# 8. Create a fake secret
cat > app.env <<'EOF'
APP_ENV=development
AWS_ACCESS_KEY_ID=AKIAZ2X3C4V5B6N7M2Q3
EOF

# 9. View the file
cat app.env

# 10. Scan the current file
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  dir -v /repo/app.env

# 11. Check the Gitleaks exit code
echo $?

# 12. Add the vulnerable file to Git
git add app.env

# 13. Commit the secret
git commit -m "Add application configuration"

# 14. View Git history
git log --oneline

# 15. Scan Git history
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  git -v /repo

# 16. Remove the vulnerable file
rm app.env

# 17. Prevent the local environment file from being committed again
echo "app.env" > .gitignore

# 18. Create a safe environment template
cat > app.env.example <<'EOF'
APP_ENV=development
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
EOF

# 19. Stage all changes
git add -A

# 20. Commit the remediation
git commit -m "Remove hardcoded credentials"

# 21. Inspect the current files
ls -la

# 22. Scan the safe current file
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  dir -v /repo/app.env.example

# 23. Scan Git history again
docker run --rm \
  -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest \
  git -v /repo

# 24. Optional: inspect Git history again
git log --oneline
```

---

# Key Concepts

## `gitleaks dir`

Use:

```bash
gitleaks dir
```

to scan the current contents of files and directories.

Conceptually:

```text
Current Files
     |
     v
Gitleaks
     |
     v
Secret Detection
```

This answers the question:

> Does the current filesystem contain a detectable secret?

---

## `gitleaks git`

Use:

```bash
gitleaks git
```

to scan a Git repository and its commit history.

Conceptually:

```text
Git Repository
      |
      v
Commit History
      |
      v
Gitleaks
      |
      v
Historical Secret Detection
```

This answers a different question:

> Has a secret been committed to the repository at some point in its history?

---

# What to Do When a Real Secret Is Exposed

If a real credential is accidentally committed, simply deleting the file is not enough.

A typical incident response workflow is:

```text
Secret Detected
      |
      v
Revoke / Rotate
      |
      v
Investigate Exposure
      |
      v
Remove Hardcoded Secret
      |
      v
Clean Git History if Required
      |
      v
Store Secret Securely
      |
      v
Add Automated Detection
      |
      v
Prevent Recurrence
```

## Why Revoke or Rotate First?

Once a real secret has been committed, you should assume that it may have been exposed.

Removing it from Git does not guarantee that:

* Nobody cloned the repository.
* Nobody copied the credential.
* It was not captured by CI/CD logs.
* It was not cached elsewhere.
* It was not already used by an unauthorized party.

For this reason, **revocation or rotation is usually one of the highest-priority remediation actions**.

---

# DevSecOps Perspective

Secret detection should happen as early as possible in the Software Development Life Cycle.

A weak workflow looks like:

```text
Developer
   |
   v
Code
   |
   v
Commit
   |
   v
Push
   |
   v
Security Scan
```

At that point, the secret may already exist remotely.

A better DevSecOps workflow is:

```text
Developer
   |
   v
Code
   |
   v
Gitleaks
   |
   +---- Clean ----> Commit / Pipeline
   |
   +---- Secret ---> BLOCK
```

The same concept can later be extended into CI/CD:

```text
Developer Push
      |
      v
CI Pipeline
      |
      v
Gitleaks Scan
      |
   +--+--+
   |     |
Clean   Leak
   |     |
   v     v
Continue FAIL
```

This is an example of **Shift Left Security**: moving security controls earlier in the development lifecycle rather than waiting until deployment or production.

---

# Summary

In this lab, you learned that:

* Secrets should never be hardcoded into source code.
* Gitleaks can detect credentials and other sensitive values.
* `gitleaks dir` scans current files and directories.
* `gitleaks git` scans Git repositories and their history.
* Git history can retain secrets even after they are deleted from the current source code.
* Exit codes allow security scanners to become automated CI/CD security gates.
* Real exposed credentials should be revoked or rotated.
* Secret detection should happen as early as possible in the development workflow.

The most important lesson from this lab is:

```text
Deleting a Secret from Source Code
             !=
Deleting a Secret from Git History
```

This principle will become increasingly important as we integrate security scanning into the rest of the DevSecOps pipeline.



# DevSecOps Lab 02 — SAST with Semgrep

## Objective

This lab introduces **Static Application Security Testing (SAST)** using Semgrep.

The lab demonstrates how security issues can be detected directly in source code before an application is built, containerized, or deployed.

By the end of this lab, you will be able to:

* Explain the purpose of SAST.
* Run Semgrep locally using Docker.
* Create a simple custom Semgrep rule.
* Scan Python source code for an insecure API.
* Interpret Semgrep findings.
* Understand Semgrep exit-code behavior.
* Use `--error` to simulate a CI/CD security gate.
* Remediate a finding and verify that it is no longer detected.

---

## Tools Used

* Ubuntu VM
* Docker
* Python source code
* Semgrep Community Edition

---

## Lab Architecture

```text
Developer
    |
    v
Source Code
    |
    v
Semgrep SAST
    |
    v
Security Rules
    |
 +--+--+
 |     |
Clean Finding
 |     |
 v     v
Pass  Fix
```

---

## Lab Directory

```text
devsecops-labs/
└── 02-sast-semgrep/
    ├── app.py
    └── semgrep-rules.yml
```

---

# 1. Create the Lab Directory

```bash
mkdir -p ~/devsecops-labs/02-sast-semgrep
cd ~/devsecops-labs/02-sast-semgrep
```

Verify the current directory:

```bash
pwd
```

---

# 2. Install Semgrep Using Docker

Pull the Semgrep Docker image:

```bash
docker pull semgrep/semgrep:latest
```

Verify the installation:

```bash
docker run --rm \
  semgrep/semgrep:latest \
  semgrep --version
```

---

# 3. Create a Vulnerable Python Application

Create `app.py`:

```bash
cat > app.py <<'EOF'
import os

def ping_host(host):
    command = "ping -c 1 " + host
    os.system(command)

host = input("Enter host: ")
ping_host(host)
EOF
```

Display the file:

```bash
cat app.py
```

The vulnerable data flow is:

```text
User Input
    |
    v
host
    |
    v
String Concatenation
    |
    v
os.system()
    |
    v
Operating System Shell
```

The use of `os.system()` is intentionally included so that the lab can demonstrate static detection of a dangerous API.

---

# 4. Create a Custom Semgrep Rule

Create `semgrep-rules.yml`:

```bash
cat > semgrep-rules.yml <<'EOF'
rules:
  - id: lab.python.dangerous-os-system
    message: Avoid os.system(); passing untrusted input may lead to command injection.
    severity: ERROR
    languages:
      - python
    pattern: os.system(...)
EOF
```

Display the rule:

```bash
cat semgrep-rules.yml
```

The important components are:

```yaml
id: lab.python.dangerous-os-system
```

The unique identifier for the rule.

```yaml
severity: ERROR
```

The severity assigned to the finding.

```yaml
languages:
  - python
```

The language analyzed by the rule.

```yaml
pattern: os.system(...)
```

The code pattern that Semgrep searches for.

---

# 5. Run the First SAST Scan

Run Semgrep:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --config /src/semgrep-rules.yml \
  /src/app.py
```

The scan should identify:

```python
os.system(command)
```

using the rule:

```text
lab.python.dangerous-os-system
```

---

# 6. Understand the Finding

A Semgrep finding typically provides information such as:

```text
Rule ID
Severity
Message
File
Line
Code
```

For this lab, the important values are:

```text
Rule ID:
lab.python.dangerous-os-system

Severity:
ERROR

Message:
Avoid os.system(); passing untrusted input may lead to command injection.
```

The highlighted code should correspond to:

```python
os.system(command)
```

---

# 7. Check the Default Exit Code

Run the scan:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --config /src/semgrep-rules.yml \
  /src/app.py
```

Check the exit code immediately:

```bash
echo $?
```

A normal `semgrep scan` can successfully complete while still reporting findings.

---

# 8. Turn the Scan into a Security Gate

Run Semgrep with `--error`:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --error \
  --config /src/semgrep-rules.yml \
  /src/app.py
```

Check the exit code:

```bash
echo $?
```

With the vulnerable code present, the expected result is:

```text
1
```

This behavior can be used as a CI/CD security gate.

```text
Semgrep
   |
   v
Finding?
 /      \
No      Yes
|        |
0        1
|        |
Pass    Fail
```

---

# 9. Remediate the Finding

Replace the vulnerable implementation:

```bash
cat > app.py <<'EOF'
import subprocess

def ping_host(host):
    subprocess.run(
        ["ping", "-c", "1", host],
        check=True
    )

host = input("Enter host: ")
ping_host(host)
EOF
```

Display the remediated source code:

```bash
cat app.py
```

The application no longer uses:

```python
os.system()
```

Instead, command arguments are passed as a list:

```python
subprocess.run(
    ["ping", "-c", "1", host],
    check=True
)
```

---

# 10. Scan the Remediated Code

Run Semgrep again:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --error \
  --config /src/semgrep-rules.yml \
  /src/app.py
```

Check the exit code:

```bash
echo $?
```

For the custom rule used in this lab, the expected result is:

```text
0
```

The rule no longer finds:

```text
os.system(...)
```

---

# 11. Optional Registry Scan

If network access is available, Semgrep can also select rules automatically:

```bash
docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --config auto \
  --metrics=off \
  /src
```

This step is optional and is not required to complete the core lab.

---

# Complete Lab Command Reference

```bash
# ==========================================
# DevSecOps Lab 02 - SAST with Semgrep
# ==========================================


# ------------------------------------------
# 1. Create Lab Environment
# ------------------------------------------

mkdir -p ~/devsecops-labs/02-sast-semgrep

cd ~/devsecops-labs/02-sast-semgrep

pwd


# ------------------------------------------
# 2. Download Semgrep
# ------------------------------------------

docker pull semgrep/semgrep:latest


# ------------------------------------------
# 3. Verify Semgrep
# ------------------------------------------

docker run --rm \
  semgrep/semgrep:latest \
  semgrep --version


# ------------------------------------------
# 4. Create Vulnerable Application
# ------------------------------------------

cat > app.py <<'EOF'
import os

def ping_host(host):
    command = "ping -c 1 " + host
    os.system(command)

host = input("Enter host: ")
ping_host(host)
EOF


# ------------------------------------------
# 5. Inspect Vulnerable Code
# ------------------------------------------

cat app.py


# ------------------------------------------
# 6. Create Custom Semgrep Rule
# ------------------------------------------

cat > semgrep-rules.yml <<'EOF'
rules:
  - id: lab.python.dangerous-os-system
    message: Avoid os.system(); passing untrusted input may lead to command injection.
    severity: ERROR
    languages:
      - python
    pattern: os.system(...)
EOF


# ------------------------------------------
# 7. Inspect Semgrep Rule
# ------------------------------------------

cat semgrep-rules.yml


# ------------------------------------------
# 8. Run Initial SAST Scan
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --config /src/semgrep-rules.yml \
  /src/app.py


# ------------------------------------------
# 9. Check Default Exit Code
# ------------------------------------------

echo $?


# ------------------------------------------
# 10. Run Semgrep as a Security Gate
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --error \
  --config /src/semgrep-rules.yml \
  /src/app.py


# ------------------------------------------
# 11. Check Security Gate Exit Code
# ------------------------------------------

echo $?


# ------------------------------------------
# 12. Remediate Vulnerable Code
# ------------------------------------------

cat > app.py <<'EOF'
import subprocess

def ping_host(host):
    subprocess.run(
        ["ping", "-c", "1", host],
        check=True
    )

host = input("Enter host: ")
ping_host(host)
EOF


# ------------------------------------------
# 13. Inspect Remediated Code
# ------------------------------------------

cat app.py


# ------------------------------------------
# 14. Scan the Remediated Application
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --error \
  --config /src/semgrep-rules.yml \
  /src/app.py


# ------------------------------------------
# 15. Verify the Final Exit Code
# ------------------------------------------

echo $?


# ------------------------------------------
# 16. Optional - Scan with Semgrep Registry
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src" \
  semgrep/semgrep:latest \
  semgrep scan \
  --config auto \
  --metrics=off \
  /src
```

---

# Key Concepts

## SAST

Static Application Security Testing analyzes source code without requiring the application to run.

```text
Source Code
    |
    v
Static Analysis
    |
    v
Security Findings
```

---

## Source and Sink

Security analysis frequently considers how data moves between a source and a sensitive destination.

```text
Source
  |
  v
Data
  |
  v
Sink
```

In the vulnerable example:

```text
input()
   |
   v
host
   |
   v
command
   |
   v
os.system()
```

---

## Finding vs Vulnerability

A static-analysis finding should be reviewed in context.

```text
SAST Finding
     |
     v
   Triage
     |
 +---+---+
 |       |
True    False
Positive Positive
```

A scanner finding is evidence that code requires investigation; it should not automatically be interpreted as proof that an application is exploitable.

---

## Security Gates

Exit codes allow security tooling to control a CI/CD workflow.

```text
Source Code
     |
     v
   Semgrep
     |
  +--+--+
  |     |
  0     1
  |     |
  v     v
Pass   Fail
  |
  v
Build
```

---

# DevSecOps Workflow

Eventually, this SAST stage can become part of a larger DevSecOps pipeline:

```text
Developer
    |
    v
Secret Scan
    |
    v
SAST
    |
    v
SCA
    |
    v
Docker Build
    |
    v
Container Scan
    |
    v
Deploy
    |
    v
DAST
```

The purpose of SAST is to identify security problems as early as possible, ideally before vulnerable source code progresses further through the delivery pipeline.

---

# Lab Summary

In this lab, you:

1. Installed Semgrep using Docker.
2. Created intentionally vulnerable Python source code.
3. Created a custom Semgrep rule.
4. Performed a static source-code scan.
5. Identified the use of a dangerous API.
6. Examined Semgrep's exit-code behavior.
7. Used `--error` to create a security gate.
8. Remediated the insecure code.
9. Re-ran the scan and verified the remediation.

The key workflow is:

```text
Write Code
    |
    v
SAST Scan
    |
    v
Finding
    |
    v
Triage
    |
    v
Remediate
    |
    v
Re-scan
    |
    v
Pass
```

This is a fundamental **Shift Left Security** pattern and will later be integrated into the complete CI/CD pipeline.

