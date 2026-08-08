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

# DevSecOps Lab 03 — Software Composition Analysis with Trivy

## Objective

This lab introduces **Software Composition Analysis (SCA)** using Trivy.

The goal is to identify known vulnerabilities in third-party dependencies before an application progresses further through the software delivery lifecycle.

By the end of this lab, you will be able to:

* Explain the purpose of Software Composition Analysis.
* Understand the difference between SAST and SCA.
* Scan Python dependencies using Trivy.
* Interpret CVE findings.
* Understand installed and fixed package versions.
* Filter vulnerabilities by severity.
* Use Trivy exit codes as CI/CD security gates.
* Upgrade a vulnerable dependency and verify the remediation.
* Understand direct and transitive dependencies.

---

## Tools Used

* Ubuntu VM
* Docker
* Trivy
* Python `requirements.txt`

---

## SAST vs SCA

SAST analyzes the application's source code:

```text
Source Code
    |
    v
SAST Scanner
    |
    v
Unsafe Code Patterns
```

SCA analyzes third-party dependencies:

```text
Dependency Manifest
       |
       v
   SCA Scanner
       |
       v
Known Vulnerability Database
       |
       v
      CVEs
```

Both controls are required because secure application code can still depend on vulnerable third-party components.

---

# Lab Architecture

```text
requirements.txt
      |
      v
    Trivy
      |
      v
Vulnerability Database
      |
      v
Known CVEs
      |
      v
Security Policy
   /       \
 Pass      Fail
```

---

# Lab Directory

```text
devsecops-labs/
└── 03-sca-trivy/
    └── requirements.txt
```

---

# 1. Create the Lab Environment

```bash
mkdir -p ~/devsecops-labs/03-sca-trivy
cd ~/devsecops-labs/03-sca-trivy
```

Verify the current location:

```bash
pwd
```

---

# 2. Install Trivy with Docker

Pull the Trivy container image:

```bash
docker pull aquasec/trivy:latest
```

Verify Trivy:

```bash
docker run --rm \
  aquasec/trivy:latest \
  version
```

Create a persistent Trivy cache directory:

```bash
mkdir -p "$HOME/.cache/trivy"
```

The persistent cache prevents Trivy from unnecessarily downloading its vulnerability database from scratch for every container execution.

---

# 3. Create an Intentionally Outdated Dependency

Create `requirements.txt`:

```bash
cat > requirements.txt <<'EOF'
requests==2.25.0
EOF
```

Inspect the file:

```bash
cat requirements.txt
```

Expected content:

```text
requests==2.25.0
```

This old package version is intentionally used to demonstrate dependency vulnerability detection.

---

# 4. Run the Initial Vulnerability Scan

Run Trivy against the project filesystem:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  /src
```

During the first execution, Trivy may download or update its vulnerability database.

Trivy analyzes supported dependency manifests such as `requirements.txt` and compares identified package versions against known vulnerability information.

---

# 5. Interpret Trivy Findings

A vulnerability report can contain fields such as:

```text
Library
Vulnerability
Severity
Installed Version
Fixed Version
Title
```

For example:

```text
Package
   |
   +-- Installed Version
   |
   +-- CVE
   |
   +-- Severity
   |
   +-- Fixed Version
```

Important fields include:

### Library

The affected third-party component.

### Vulnerability

The vulnerability identifier, commonly a CVE.

### Severity

Typical severity levels are:

```text
UNKNOWN
LOW
MEDIUM
HIGH
CRITICAL
```

### Installed Version

The package version currently declared by the application.

### Fixed Version

A version in which the vulnerability has been addressed.

The exact CVEs and severity classifications may change over time as vulnerability databases are updated.

---

# 6. Filter Findings by Severity

Display only `MEDIUM`, `HIGH`, and `CRITICAL` findings:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  /src
```

Severity filtering allows engineering teams to implement organization-specific vulnerability policies.

---

# 7. Inspect the Default Exit Code

Run the vulnerability scan:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  /src
```

Immediately inspect the exit code:

```bash
echo $?
```

By default, Trivy can complete successfully with exit code `0` while still reporting security findings.

This behavior allows users to separate vulnerability reporting from pipeline enforcement.

---

# 8. Configure a CI/CD Security Gate

Configure Trivy to return exit code `1` when matching vulnerabilities are detected:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  --exit-code 1 \
  /src
```

Inspect the exit code:

```bash
echo $?
```

If a matching vulnerability is detected, the expected result is:

```text
1
```

This creates a basic security gate:

```text
Trivy
  |
  v
Vulnerabilities?
 /           \
No           Yes
|             |
0             1
|             |
v             v
PASS         FAIL
```

---

# 9. Example Production Severity Policy

A real organization may choose a policy such as:

```text
LOW       -> Report
MEDIUM    -> Report / Create Ticket
HIGH      -> Block
CRITICAL  -> Block
```

A corresponding scan could be:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  /src
```

The security scanner identifies findings, while the organization's security policy determines which findings block delivery.

---

# 10. Upgrade the Dependency

Replace the outdated package version:

```bash
cat > requirements.txt <<'EOF'
requests==2.34.2
EOF
```

Inspect the updated manifest:

```bash
cat requirements.txt
```

Expected content:

```text
requests==2.34.2
```

---

# 11. Re-scan After Remediation

Run the security gate again:

```bash
docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  --exit-code 1 \
  /src
```

Check the exit code:

```bash
echo $?
```

If no vulnerabilities matching the configured policy remain, the expected result is:

```text
0
```

The remediation workflow is therefore:

```text
Detect
  |
  v
Identify Vulnerable Dependency
  |
  v
Review CVE
  |
  v
Identify Fixed Version
  |
  v
Upgrade
  |
  v
Re-scan
  |
  v
Pass
```

---

# Direct vs Transitive Dependencies

A dependency declared directly by the application is a **direct dependency**.

For example:

```text
Application
    |
    v
requests
```

However, that dependency may depend on additional packages:

```text
Application
    |
    v
requests
    |
    +--> urllib3
    |
    +--> certifi
    |
    +--> idna
    |
    +--> charset-normalizer
```

These secondary packages are **transitive dependencies**.

A vulnerability may therefore exist several levels below the application's direct dependencies:

```text
Application
      |
      v
Direct Dependency
      |
      v
Transitive Dependency
      |
      v
Known Vulnerability
```

A simple `requirements.txt` frequently contains only direct dependencies.

For more complete Python dependency scanning, teams commonly generate fully resolved dependency information using mechanisms such as lock files, `pip freeze`, or dependency-management tooling.

---

# Continuous SCA

A clean scan does not mean that a dependency will remain safe forever.

```text
Package Safe Today
       |
       v
New Vulnerability Disclosed
       |
       v
New CVE
       |
       v
Package Becomes a Finding
```

For this reason, SCA should be executed continuously as part of CI/CD rather than treated as a one-time audit.

---

# DevSecOps Pipeline Progress

After completing this lab, the security workflow contains three layers:

```text
Source Code
    |
    v
Gitleaks
Secrets Detection
    |
    v
Semgrep
SAST
    |
    v
Trivy
SCA
    |
    v
Docker Build
```

Each layer answers a different security question:

```text
Gitleaks
   |
   +--> Did we expose credentials?

Semgrep
   |
   +--> Did we write insecure code?

Trivy SCA
   |
   +--> Are our dependencies known to be vulnerable?
```

---

# Complete Lab Command Reference

```bash
# ==========================================
# DevSecOps Lab 03 - SCA with Trivy
# ==========================================


# ------------------------------------------
# 1. Create Lab Environment
# ------------------------------------------

mkdir -p ~/devsecops-labs/03-sca-trivy

cd ~/devsecops-labs/03-sca-trivy

pwd


# ------------------------------------------
# 2. Download Trivy
# ------------------------------------------

docker pull aquasec/trivy:latest


# ------------------------------------------
# 3. Verify Trivy
# ------------------------------------------

docker run --rm \
  aquasec/trivy:latest \
  version


# ------------------------------------------
# 4. Create Persistent Trivy Cache
# ------------------------------------------

mkdir -p "$HOME/.cache/trivy"


# ------------------------------------------
# 5. Create Vulnerable Dependency Manifest
# ------------------------------------------

cat > requirements.txt <<'EOF'
requests==2.25.0
EOF


# ------------------------------------------
# 6. Inspect Dependency Manifest
# ------------------------------------------

cat requirements.txt


# ------------------------------------------
# 7. Run Initial SCA Scan
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  /src


# ------------------------------------------
# 8. Filter by Severity
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  /src


# ------------------------------------------
# 9. Check Default Exit Code
# ------------------------------------------

echo $?


# ------------------------------------------
# 10. Run Trivy as a Security Gate
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  --exit-code 1 \
  /src


# ------------------------------------------
# 11. Check Security Gate Exit Code
# ------------------------------------------

echo $?


# ------------------------------------------
# 12. Example HIGH/CRITICAL Production Gate
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  /src


# ------------------------------------------
# 13. Upgrade the Vulnerable Dependency
# ------------------------------------------

cat > requirements.txt <<'EOF'
requests==2.34.2
EOF


# ------------------------------------------
# 14. Inspect the Updated Dependency
# ------------------------------------------

cat requirements.txt


# ------------------------------------------
# 15. Re-scan After Remediation
# ------------------------------------------

docker run --rm \
  -v "$PWD:/src:ro" \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  fs \
  --scanners vuln \
  --severity MEDIUM,HIGH,CRITICAL \
  --exit-code 1 \
  /src


# ------------------------------------------
# 16. Verify Final Exit Code
# ------------------------------------------

echo $?
```

---

# Lab Summary

In this lab, you:

1. Installed Trivy using Docker.
2. Created an intentionally outdated Python dependency.
3. Scanned `requirements.txt` for known vulnerabilities.
4. Learned how to interpret CVEs.
5. Examined installed and fixed versions.
6. Filtered vulnerabilities by severity.
7. Examined Trivy's default exit-code behavior.
8. Created a CI/CD security gate using `--exit-code`.
9. Upgraded the vulnerable dependency.
10. Re-scanned the project.
11. Learned the difference between direct and transitive dependencies.

The core SCA workflow is:

```text
Dependency Manifest
       |
       v
      SCA
       |
       v
Known Vulnerability
       |
       v
     CVE
       |
       v
Fixed Version
       |
       v
   Upgrade
       |
       v
    Re-scan
```

This adds the third major security control to the DevSecOps pipeline:

```text
Secrets Detection → SAST → SCA
```

# DevSecOps Lab 04 — Container Security with Docker and Trivy

## Objective

This lab introduces container image security using **Docker** and **Trivy**.

The goal is to build an intentionally vulnerable container image, inspect its runtime configuration, scan its operating-system and application dependencies for known vulnerabilities, apply basic container hardening, and verify the remediation.

By the end of this lab, you will be able to:

* Build and tag Docker images.
* Understand the security relevance of container image layers.
* Identify the user running inside a container.
* Scan a local Docker image with Trivy.
* Distinguish OS vulnerabilities from application-library vulnerabilities.
* Filter vulnerabilities by severity.
* Create a container-image security gate.
* Upgrade vulnerable application dependencies.
* Run application containers as a non-root user.
* Use `.dockerignore` to reduce unnecessary build context.
* Rebuild and re-scan a hardened image.

---

## Tools Used

* Ubuntu VM
* Docker
* Python
* Trivy

---

## Security Pipeline

```text
Source Code
    |
    v
Secrets Scan
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
Container Image
    |
    v
Trivy Image Scan
    |
 +--+--+
 |     |
Pass  Fail
```

---

## Container Image Attack Surface

A container image can include security issues in multiple layers:

```text
Container Image
│
├── Base Operating System
│   └── OS packages
│
├── Runtime
│   └── Python
│
├── Application Dependencies
│   ├── requests
│   └── transitive dependencies
│
├── Application Code
│
└── Runtime Configuration
    └── User privileges
```

Container scanning therefore evaluates a different artifact than source-code or dependency-manifest scanning.

---

# 1. Create the Lab Environment

```bash
mkdir -p ~/devsecops-labs/04-container-security
cd ~/devsecops-labs/04-container-security
```

Verify the current directory:

```bash
pwd
```

---

# 2. Create the Application

Create `app.py`:

```bash
cat > app.py <<'EOF'
import requests

print("Container Security Lab")
print("Requests version:", requests.__version__)
EOF
```

Inspect the application:

```bash
cat app.py
```

---

# 3. Create an Intentionally Outdated Dependency

Create `requirements.txt`:

```bash
cat > requirements.txt <<'EOF'
requests==2.25.0
EOF
```

Inspect it:

```bash
cat requirements.txt
```

---

# 4. Create the Initial Dockerfile

Create `Dockerfile`:

```bash
cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
EOF
```

Inspect the file:

```bash
cat Dockerfile
```

This initial Dockerfile intentionally does not configure a dedicated non-root user.

---

# 5. Build the Vulnerable Image

Build and tag the image:

```bash
docker build -t devsecops-lab:vulnerable .
```

Inspect the resulting image:

```bash
docker images devsecops-lab
```

---

# 6. Run the Application

```bash
docker run --rm \
  devsecops-lab:vulnerable
```

The application should display the installed Requests version.

---

# 7. Inspect the Container User

Run:

```bash
docker run --rm \
  devsecops-lab:vulnerable \
  id
```

The initial image will typically run as:

```text
uid=0(root)
```

Running an application as root unnecessarily increases the impact of a successful compromise inside the container.

---

# 8. Inspect Image Layers

Display the image history:

```bash
docker history \
  devsecops-lab:vulnerable
```

Display the full commands associated with each layer:

```bash
docker history \
  --no-trunc \
  devsecops-lab:vulnerable
```

Conceptually:

```text
Application Layer
       |
Dependency Layer
       |
Configuration Layer
       |
Python Runtime
       |
Base OS
```

Each layer contributes content to the final container artifact.

---

# 9. Verify Trivy

```bash
docker run --rm \
  aquasec/trivy:latest \
  version
```

Create a persistent cache directory:

```bash
mkdir -p "$HOME/.cache/trivy"
```

When Trivy itself runs in Docker, the host Docker socket is mounted so that Trivy can inspect images stored in the local Docker Engine.

---

# 10. Scan the Container Image

Run the initial vulnerability scan:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  devsecops-lab:vulnerable
```

Trivy can report vulnerabilities from both operating-system packages and application libraries contained inside the image.

---

# 11. Scan OS Packages Only

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types os \
  devsecops-lab:vulnerable
```

This scan isolates vulnerabilities inherited from the operating system and base image.

---

# 12. Scan Application Libraries Only

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  devsecops-lab:vulnerable
```

This scan isolates vulnerabilities in application-level package dependencies.

---

# 13. Filter HIGH and CRITICAL Vulnerabilities

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  devsecops-lab:vulnerable
```

Important report fields include:

```text
Package
Vulnerability ID
Severity
Installed Version
Fixed Version
```

---

# 14. Create a Container Security Gate

Configure Trivy to fail when `HIGH` or `CRITICAL` findings are present:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:vulnerable
```

Check the exit code:

```bash
echo $?
```

Conceptually:

```text
Docker Build
     |
     v
Container Image
     |
     v
Trivy Scan
     |
 HIGH/CRITICAL?
   /       \
 No        Yes
 |          |
 0          1
 |          |
 v          v
PASS       FAIL
```

This allows container scanning to run before an image is pushed to a registry or deployed.

---

# 15. Isolate the Application-Library Gate

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:vulnerable
```

Check the result:

```bash
echo $?
```

This isolates application dependency findings from operating-system findings.

---

# 16. Upgrade the Application Dependency

Replace the outdated dependency:

```bash
cat > requirements.txt <<'EOF'
requests==2.34.2
EOF
```

Inspect the new manifest:

```bash
cat requirements.txt
```

---

# 17. Harden the Dockerfile

Replace the initial Dockerfile:

```bash
cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd \
    --create-home \
    --uid 10001 \
    --shell /usr/sbin/nologin \
    appuser \
    && chown -R appuser:appuser /app

USER appuser

CMD ["python", "app.py"]
EOF
```

The important security improvement is:

```dockerfile
USER appuser
```

The application no longer needs to run as root.

---

# 18. Add `.dockerignore`

Create:

```bash
cat > .dockerignore <<'EOF'
.git
.gitignore
.env
*.env
__pycache__
*.pyc
README.md
EOF
```

Inspect it:

```bash
cat .dockerignore
```

A `.dockerignore` file reduces unnecessary files included in the Docker build context and helps prevent local files such as `.env` or `.git` content from being unintentionally sent to the builder.

---

# 19. Build the Hardened Image

Build a separate image so that the vulnerable and hardened versions can be compared:

```bash
docker build \
  --no-cache \
  -t devsecops-lab:hardened \
  .
```

List both images:

```bash
docker images devsecops-lab
```

Expected image tags:

```text
devsecops-lab:vulnerable
devsecops-lab:hardened
```

---

# 20. Verify the Hardened Container User

```bash
docker run --rm \
  devsecops-lab:hardened \
  id
```

The result should no longer show UID `0`.

A result similar to the following is expected:

```text
uid=10001(appuser)
```

---

# 21. Verify the Application

```bash
docker run --rm \
  devsecops-lab:hardened
```

The application should still run normally while using the updated dependency and non-root user.

---

# 22. Re-scan Application Libraries

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:hardened
```

Check the exit code:

```bash
echo $?
```

Compare the output with the vulnerable image.

---

# 23. Re-scan the Complete Image

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  devsecops-lab:hardened
```

The application libraries may be clean while vulnerabilities still exist in operating-system packages.

For example:

```text
Container Image
     |
 +---+---+
 |       |
OS     Libraries
 |       |
CVE     Clean
```

These findings require different remediation strategies.

---

# Remediation Strategies

## Application Dependency Vulnerability

```text
CVE
 |
 v
Upgrade Package
 |
 v
Rebuild Image
 |
 v
Re-scan
```

## Base Image / OS Vulnerability

```text
CVE
 |
 v
Update Base Image
 |
 v
Rebuild Image
 |
 v
Re-scan
```

An already-built image does not automatically receive updates when its base image changes.

---

# Container Security Principles

## 1. Scan the Artifact You Deploy

Scanning source files is not enough.

```text
Code Scan
   !=
Final Image Scan
```

The final container image contains additional operating-system and runtime components.

---

## 2. Use Least Privilege

Prefer:

```text
Application
    |
    v
Non-root User
```

instead of:

```text
Application
    |
    v
root
```

when elevated privileges are unnecessary.

---

## 3. Minimize the Image

Include only what the application requires.

```text
Smaller Image
     |
     v
Fewer Packages
     |
     v
Smaller Attack Surface
```

---

## 4. Rebuild Regularly

Container images are immutable artifacts.

```text
New Base Image Security Update
          |
          v
Old Application Image
          |
          X
Not Automatically Updated
```

The application image must be rebuilt and re-scanned.

---

## 5. Enforce Security Before Push

A preferred workflow is:

```text
Code
 |
 v
Docker Build
 |
 v
Trivy Scan
 |
 +---- Pass ----> Push to Registry
 |
 +---- Fail ----> Block
```

rather than discovering vulnerabilities after deployment.

---

# Complete Lab Command Reference

```bash
# ============================================================
# DevSecOps Lab 04 - Container Security with Docker and Trivy
# ============================================================


# ------------------------------------------------------------
# 1. Create Lab Environment
# ------------------------------------------------------------

mkdir -p ~/devsecops-labs/04-container-security

cd ~/devsecops-labs/04-container-security

pwd


# ------------------------------------------------------------
# 2. Create Application
# ------------------------------------------------------------

cat > app.py <<'EOF'
import requests

print("Container Security Lab")
print("Requests version:", requests.__version__)
EOF

cat app.py


# ------------------------------------------------------------
# 3. Create Intentionally Outdated Dependency
# ------------------------------------------------------------

cat > requirements.txt <<'EOF'
requests==2.25.0
EOF

cat requirements.txt


# ------------------------------------------------------------
# 4. Create Initial Dockerfile
# ------------------------------------------------------------

cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
EOF

cat Dockerfile


# ------------------------------------------------------------
# 5. Build Vulnerable Image
# ------------------------------------------------------------

docker build \
  -t devsecops-lab:vulnerable \
  .


# ------------------------------------------------------------
# 6. List Image
# ------------------------------------------------------------

docker images devsecops-lab


# ------------------------------------------------------------
# 7. Run Application
# ------------------------------------------------------------

docker run --rm \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 8. Inspect Container User
# ------------------------------------------------------------

docker run --rm \
  devsecops-lab:vulnerable \
  id


# ------------------------------------------------------------
# 9. Inspect Image Layers
# ------------------------------------------------------------

docker history \
  devsecops-lab:vulnerable

docker history \
  --no-trunc \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 10. Verify Trivy
# ------------------------------------------------------------

docker run --rm \
  aquasec/trivy:latest \
  version


# ------------------------------------------------------------
# 11. Create Persistent Trivy Cache
# ------------------------------------------------------------

mkdir -p "$HOME/.cache/trivy"


# ------------------------------------------------------------
# 12. Scan Complete Vulnerable Image
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 13. Scan OS Packages Only
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types os \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 14. Scan Application Libraries Only
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 15. Display HIGH and CRITICAL Findings
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  devsecops-lab:vulnerable


# ------------------------------------------------------------
# 16. Create Complete Container Security Gate
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:vulnerable

echo $?


# ------------------------------------------------------------
# 17. Create Application Library Security Gate
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:vulnerable

echo $?


# ------------------------------------------------------------
# 18. Upgrade Application Dependency
# ------------------------------------------------------------

cat > requirements.txt <<'EOF'
requests==2.34.2
EOF

cat requirements.txt


# ------------------------------------------------------------
# 19. Create Hardened Dockerfile
# ------------------------------------------------------------

cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd \
    --create-home \
    --uid 10001 \
    --shell /usr/sbin/nologin \
    appuser \
    && chown -R appuser:appuser /app

USER appuser

CMD ["python", "app.py"]
EOF

cat Dockerfile


# ------------------------------------------------------------
# 20. Create .dockerignore
# ------------------------------------------------------------

cat > .dockerignore <<'EOF'
.git
.gitignore
.env
*.env
__pycache__
*.pyc
README.md
EOF

cat .dockerignore


# ------------------------------------------------------------
# 21. Build Hardened Image
# ------------------------------------------------------------

docker build \
  --no-cache \
  -t devsecops-lab:hardened \
  .


# ------------------------------------------------------------
# 22. Compare Images
# ------------------------------------------------------------

docker images devsecops-lab


# ------------------------------------------------------------
# 23. Verify Non-root User
# ------------------------------------------------------------

docker run --rm \
  devsecops-lab:hardened \
  id


# ------------------------------------------------------------
# 24. Verify Hardened Application
# ------------------------------------------------------------

docker run --rm \
  devsecops-lab:hardened


# ------------------------------------------------------------
# 25. Re-scan Hardened Application Libraries
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --pkg-types library \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  devsecops-lab:hardened

echo $?


# ------------------------------------------------------------
# 26. Re-scan Complete Hardened Image
# ------------------------------------------------------------

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/" \
  aquasec/trivy:latest \
  image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  devsecops-lab:hardened
```

---

# Lab Summary

In this lab, you:

1. Created a Python application.
2. Added an intentionally outdated dependency.
3. Built a Docker image.
4. Inspected the image's runtime user.
5. Inspected Docker image layers.
6. Scanned the image using Trivy.
7. Separated OS vulnerabilities from library vulnerabilities.
8. Filtered `HIGH` and `CRITICAL` findings.
9. Converted Trivy into a container security gate.
10. Upgraded the vulnerable application dependency.
11. Added a dedicated non-root application user.
12. Added a `.dockerignore` file.
13. Built a hardened image.
14. Re-scanned the hardened artifact.

The core workflow is:

```text
Dockerfile
    |
    v
Docker Build
    |
    v
Container Image
    |
    v
Trivy Scan
    |
    v
Security Findings
    |
    v
Remediate
    |
    v
Rebuild
    |
    v
Re-scan
```

The DevSecOps pipeline now contains four security layers:

```text
Gitleaks
   |
   v
Secrets Detection
   |
   v
Semgrep
   |
   v
SAST
   |
   v
Trivy Filesystem
   |
   v
SCA
   |
   v
Docker Build
   |
   v
Trivy Image
   |
   v
Container Security
```




# DevSecOps Lab 05 — DAST with OWASP ZAP

## Objective

This lab introduces **Dynamic Application Security Testing (DAST)** using OWASP ZAP.

Unlike SAST, which analyzes source code, DAST evaluates a running application through its externally exposed HTTP interface.

The lab uses **OWASP Juice Shop**, an intentionally vulnerable web application designed for security education and security-tool testing.

By the end of this lab, you will be able to:

* Explain the difference between SAST and DAST.
* Run OWASP Juice Shop using Docker.
* Create a dedicated Docker network for security testing.
* Run OWASP ZAP using Docker.
* Perform a ZAP Baseline Scan.
* Understand passive scanning.
* Generate HTML and JSON security reports.
* Interpret ZAP alerts and exit codes.
* Understand security policy versus scanner findings.
* Perform a controlled ZAP Active Scan against an authorized local target.
* Understand common DAST coverage limitations.

---

## Tools Used

* Ubuntu VM
* Docker
* OWASP Juice Shop
* OWASP ZAP

---

# SAST vs DAST

SAST analyzes application source code:

```text
Source Code
    |
    v
SAST Scanner
    |
    v
Potential Vulnerability
```

The application does not need to be running.

DAST analyzes a running application:

```text
DAST Scanner
     |
     v
HTTP Requests
     |
     v
Running Application
     |
     v
HTTP Responses
     |
     v
Security Analysis
```

A useful mental model is:

```text
SAST -> Inside-out
DAST -> Outside-in
```

---

# Lab Architecture

Both containers are connected to the same dedicated Docker network:

```text
                 Docker Network: zapnet

        +-------------------------+
        |                         |
        |   OWASP Juice Shop      |
        |                         |
        |      Port 3000          |
        |                         |
        +------------+------------+
                     ^
                     |
                     | HTTP
                     |
        +------------+------------+
        |                         |
        |       OWASP ZAP         |
        |                         |
        |      DAST Scanner       |
        |                         |
        +-------------------------+
```

---

# 1. Create the Lab Environment

```bash
mkdir -p ~/devsecops-labs/05-dast-zap

cd ~/devsecops-labs/05-dast-zap

pwd
```

---

# 2. Create the Docker Security-Test Network

Create the network if it does not already exist:

```bash
docker network inspect zapnet >/dev/null 2>&1 || \
docker network create zapnet
```

Verify it:

```bash
docker network ls
```

A shared Docker network allows the ZAP container to reach the target container using Docker DNS and the target container name.

---

# 3. Download OWASP Juice Shop

```bash
docker pull bkimminich/juice-shop
```

---

# 4. Start OWASP Juice Shop

```bash
docker run -d \
  --rm \
  --name juice-shop \
  --network zapnet \
  -p 3000:3000 \
  bkimminich/juice-shop
```

Verify that the container is running:

```bash
docker ps
```

Inspect recent application logs:

```bash
docker logs --tail 20 juice-shop
```

---

# 5. Verify the Target Application

From the Ubuntu host:

```bash
curl -I http://localhost:3000
```

The application can also be accessed in a browser at:

```text
http://localhost:3000
```

The DAST scanner requires a reachable running application.

---

# 6. Download OWASP ZAP

```bash
docker pull ghcr.io/zaproxy/zaproxy:stable
```

Verify ZAP:

```bash
docker run --rm \
  ghcr.io/zaproxy/zaproxy:stable \
  zap.sh -version
```

---

# ZAP Scan Types

ZAP provides several packaged Docker scans.

```text
Baseline Scan
    |
    +-- Spider
    |
    +-- Passive Scan


Full Scan
    |
    +-- Spider
    |
    +-- Passive Scan
    |
    +-- Active Scan


API Scan
    |
    +-- OpenAPI
    +-- SOAP
    +-- GraphQL
```

The Baseline Scan is the appropriate starting point because it performs discovery and passive analysis without running a full active attack against the application.

---

# 7. Run the ZAP Baseline Scan

```bash
docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://juice-shop:3000 \
  -j \
  -m 1 \
  -r zap-baseline-report.html \
  -J zap-baseline-report.json
```

The `-j` option enables the modern spider in addition to the traditional spider, helping discovery in JavaScript-heavy applications.

The scan generates both HTML and JSON reports.

---

# Baseline Scan Workflow

```text
Target
  |
  v
Spider
  |
  v
Discover URLs
  |
  v
HTTP Traffic
  |
  v
Passive Scanner
  |
  v
Security Alerts
```

The passive scanner analyzes the HTTP traffic generated during discovery.

It does not need access to application source code.

---

# 8. Inspect Generated Reports

List the generated files:

```bash
ls -lh
```

Expected report files include:

```text
zap-baseline-report.html
zap-baseline-report.json
```

Pretty-print the JSON report:

```bash
python3 -m json.tool zap-baseline-report.json | less
```

Quickly inspect important alert fields:

```bash
grep -Ei \
  '"alert"|"riskdesc"|"confidence"|"url"' \
  zap-baseline-report.json | head -n 80
```

On a desktop Ubuntu environment, open the HTML report with:

```bash
xdg-open zap-baseline-report.html
```

---

# Understanding a ZAP Finding

Important fields can include:

```text
Alert
Risk
Confidence
URL
Parameter
Evidence
Description
Solution
```

Conceptually:

```text
Alert
  |
  +--> What security issue was detected?


Risk
  |
  +--> How serious is it?


Confidence
  |
  +--> How confident is ZAP?


URL
  |
  +--> Where was it detected?


Parameter
  |
  +--> Which input was involved?


Evidence
  |
  +--> What triggered the alert?


Solution
  |
  +--> How should it be remediated?
```

---

# 9. Inspect the Baseline Exit Code

Immediately after running the scan:

```bash
echo $?
```

The ZAP Baseline script defines the following exit values:

```text
0 -> Success

1 -> At least one rule marked FAIL was triggered

2 -> At least one WARN and no FAIL rules

3 -> Scanner or execution failure
```

By default, Baseline alerts are treated as warnings unless policy configuration changes them to `FAIL` or `IGNORE`.

A result such as:

```text
2
```

does not mean ZAP crashed.

It means the scan ran and reported warning-level policy results.

---

# Scanner Status vs Security Status

A DevSecOps pipeline must distinguish between scanner failures and security findings.

```text
Exit 3
   |
   v
Execution Problem
```

versus:

```text
Exit 2
   |
   v
Scan Completed
   |
   v
Security Warnings Found
```

Understanding tool-specific exit-code semantics is necessary before integrating scanners into CI/CD.

---

# 10. Demonstrate an Allow-Warnings Policy

Run:

```bash
docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://juice-shop:3000 \
  -j \
  -m 1 \
  -I \
  -r zap-policy-report.html \
  -J zap-policy-report.json
```

Inspect the exit code:

```bash
echo $?
```

The `-I` option prevents warning-only results from causing a failure exit status.

This demonstrates the separation between:

```text
Security Detection
       |
       v
Security Policy
       |
   +---+---+
   |       |
Report   Block
```

Security scanners identify findings.

Security policy determines which findings block delivery.

---

# 11. Run a Controlled Active Scan

> **Authorization Warning**
>
> Active security scanning must only be performed against systems you own or have explicit authorization to test.
>
> This lab uses OWASP Juice Shop, an intentionally vulnerable local training target.

Run the ZAP Full Scan:

```bash
docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
  -t http://juice-shop:3000 \
  -m 2 \
  -r zap-full-report.html \
  -J zap-full-report.json
```

The ZAP Full Scan performs both passive and active security testing. ZAP explicitly documents that the Full Scan sends real attack traffic to the configured target.

---

# Baseline vs Full Scan

```text
                 Baseline        Full

Spider              YES           YES

Passive Scan        YES           YES

Active Scan          NO           YES
```

Passive scanning analyzes traffic that has already occurred.

```text
HTTP Request
      |
      v
HTTP Response
      |
      v
Passive Analysis
```

Active scanning sends additional test requests to determine how the application behaves.

```text
ZAP
 |
 v
Security Test Request
 |
 v
Application
 |
 v
Response
 |
 v
Analysis
```

---

# 12. Inspect the Full Scan Report

List the reports:

```bash
ls -lh zap-full-report.*
```

Inspect the JSON:

```bash
python3 -m json.tool zap-full-report.json | less
```

Quickly inspect security alerts:

```bash
grep -Ei \
  '"alert"|"riskdesc"|"confidence"|"url"' \
  zap-full-report.json | head -n 100
```

On a desktop environment:

```bash
xdg-open zap-full-report.html
```

---

# DAST Coverage

DAST only analyzes application functionality that the scanner can reach.

Consider:

```text
Application
│
├── /
│
├── /login
│
├── /products
│
├── /api/products
│
└── /admin/internal
```

If ZAP discovers only:

```text
/
/login
/products
/api/products
```

then:

```text
/admin/internal
```

may not be tested.

This creates a **coverage gap**.

---

# Common Causes of DAST Coverage Gaps

Examples include:

```text
Authentication
       |
       v
Protected Routes


JavaScript Navigation
       |
       v
Undiscovered Routes


API Endpoints
       |
       v
No UI Links


Role-Based Access
       |
       v
Admin-only Functionality
```

More advanced DAST programs therefore use capabilities such as:

```text
Authentication

Contexts

API specifications

Modern spiders

Multiple test users

Role-specific scanning
```

---

# DAST in the DevSecOps Pipeline

The pipeline now looks like:

```text
Developer
    |
    v
Source Code
    |
    v
Gitleaks
Secrets Scan
    |
    v
Semgrep
SAST
    |
    v
Trivy Filesystem
SCA
    |
    v
Docker Build
    |
    v
Trivy Image
Container Security
    |
    v
Run Test Environment
    |
    v
OWASP ZAP
DAST
    |
 +--+--+
 |     |
Pass  Fail
 |     |
 v     X
Release
```

---

# Cleanup

Stop Juice Shop:

```bash
docker stop juice-shop
```

Because the container was started using `--rm`, Docker removes it after it stops.

Remove the test network:

```bash
docker network rm zapnet
```

Verify the remaining containers:

```bash
docker ps -a
```

The ZAP reports remain in the lab directory.

---

# Complete Lab Command Reference

```bash
# ============================================================
# DevSecOps Lab 05 - DAST with OWASP ZAP
# ============================================================


# ------------------------------------------------------------
# 1. Create Lab Environment
# ------------------------------------------------------------

mkdir -p ~/devsecops-labs/05-dast-zap

cd ~/devsecops-labs/05-dast-zap

pwd


# ------------------------------------------------------------
# 2. Create Docker Network
# ------------------------------------------------------------

docker network inspect zapnet >/dev/null 2>&1 || \
docker network create zapnet

docker network ls


# ------------------------------------------------------------
# 3. Download OWASP Juice Shop
# ------------------------------------------------------------

docker pull bkimminich/juice-shop


# ------------------------------------------------------------
# 4. Start OWASP Juice Shop
# ------------------------------------------------------------

docker run -d \
  --rm \
  --name juice-shop \
  --network zapnet \
  -p 3000:3000 \
  bkimminich/juice-shop


# ------------------------------------------------------------
# 5. Verify Target Container
# ------------------------------------------------------------

docker ps

docker logs --tail 20 juice-shop


# ------------------------------------------------------------
# 6. Verify Web Application
# ------------------------------------------------------------

curl -I http://localhost:3000


# ------------------------------------------------------------
# 7. Download OWASP ZAP
# ------------------------------------------------------------

docker pull ghcr.io/zaproxy/zaproxy:stable


# ------------------------------------------------------------
# 8. Verify ZAP
# ------------------------------------------------------------

docker run --rm \
  ghcr.io/zaproxy/zaproxy:stable \
  zap.sh -version


# ------------------------------------------------------------
# 9. Run Baseline DAST Scan
# ------------------------------------------------------------

docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://juice-shop:3000 \
  -j \
  -m 1 \
  -r zap-baseline-report.html \
  -J zap-baseline-report.json


# ------------------------------------------------------------
# 10. Inspect Baseline Exit Code
# ------------------------------------------------------------

echo $?


# ------------------------------------------------------------
# 11. List ZAP Reports
# ------------------------------------------------------------

ls -lh


# ------------------------------------------------------------
# 12. Inspect Baseline JSON Report
# ------------------------------------------------------------

python3 -m json.tool \
  zap-baseline-report.json | less


# ------------------------------------------------------------
# 13. Extract Important Alert Fields
# ------------------------------------------------------------

grep -Ei \
  '"alert"|"riskdesc"|"confidence"|"url"' \
  zap-baseline-report.json | head -n 80


# ------------------------------------------------------------
# 14. Optional - Open HTML Report on Desktop Ubuntu
# ------------------------------------------------------------

xdg-open zap-baseline-report.html


# ------------------------------------------------------------
# 15. Run Baseline with Allow-Warnings Policy
# ------------------------------------------------------------

docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://juice-shop:3000 \
  -j \
  -m 1 \
  -I \
  -r zap-policy-report.html \
  -J zap-policy-report.json


# ------------------------------------------------------------
# 16. Inspect Policy Exit Code
# ------------------------------------------------------------

echo $?


# ------------------------------------------------------------
# 17. Run Authorized Active DAST Scan
# ------------------------------------------------------------

docker run --rm \
  --network zapnet \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
  -t http://juice-shop:3000 \
  -m 2 \
  -r zap-full-report.html \
  -J zap-full-report.json


# ------------------------------------------------------------
# 18. Inspect Full Scan Exit Code
# ------------------------------------------------------------

echo $?


# ------------------------------------------------------------
# 19. List Full Scan Reports
# ------------------------------------------------------------

ls -lh zap-full-report.*


# ------------------------------------------------------------
# 20. Inspect Full Scan JSON
# ------------------------------------------------------------

python3 -m json.tool \
  zap-full-report.json | less


# ------------------------------------------------------------
# 21. Extract Full Scan Alerts
# ------------------------------------------------------------

grep -Ei \
  '"alert"|"riskdesc"|"confidence"|"url"' \
  zap-full-report.json | head -n 100


# ------------------------------------------------------------
# 22. Optional - Open Full HTML Report
# ------------------------------------------------------------

xdg-open zap-full-report.html


# ------------------------------------------------------------
# 23. Stop Test Application
# ------------------------------------------------------------

docker stop juice-shop


# ------------------------------------------------------------
# 24. Remove Docker Test Network
# ------------------------------------------------------------

docker network rm zapnet


# ------------------------------------------------------------
# 25. Verify Cleanup
# ------------------------------------------------------------

docker ps -a
```

---

# Key Concepts

## DAST Requires a Running Target

```text
Application Stopped
       |
       X
DAST Cannot Reach Target
```

versus:

```text
Application Running
       |
       v
HTTP Interface
       |
       v
DAST Scanner
```

---

## Passive Scanning

```text
Existing HTTP Traffic
       |
       v
Observe
       |
       v
Analyze
       |
       v
Findings
```

Passive scanning does not need to actively inject test payloads.

---

## Active Scanning

```text
ZAP
 |
 v
Security Test Request
 |
 v
Application
 |
 v
Behavior
 |
 v
Security Finding
```

Active scanning must only be performed against authorized targets.

---

## Scanner vs Policy

A finding alone does not determine whether a deployment should fail.

```text
Scanner
   |
   v
Finding
   |
   v
Security Policy
   |
 +─+──────+
 |        |
Allow    Block
```

This distinction becomes especially important when integrating security tooling into CI/CD.

---

## DAST Coverage Matters

A scanner cannot effectively test application functionality it cannot reach.

```text
Application Attack Surface
           |
           v
     Scanner Coverage
           |
     +-----+-----+
     |           |
Discovered   Undiscovered
     |           |
     v           v
  Tested      Not Tested
```

Authentication and application discovery are therefore major parts of mature DAST programs.

---

# Lab Summary

In this lab, you:

1. Created a dedicated Docker security-testing network.
2. Started OWASP Juice Shop as an intentionally vulnerable local target.
3. Installed OWASP ZAP using Docker.
4. Performed a Baseline DAST scan.
5. Used traditional and modern application discovery.
6. Generated HTML and JSON reports.
7. Interpreted ZAP findings.
8. Learned ZAP Baseline exit-code semantics.
9. Applied an allow-warnings policy.
10. Performed an authorized Active Scan against the local training application.
11. Compared passive and active scanning.
12. Learned why DAST coverage matters.
13. Cleaned up the local testing environment.

The complete security workflow now covers:

```text
Secrets
   |
   v
Gitleaks

Source Code
   |
   v
Semgrep

Dependencies
   |
   v
Trivy SCA

Container Image
   |
   v
Trivy Image

Running Application
   |
   v
OWASP ZAP
```

The final stage is to combine these controls into an automated **CI/CD DevSecOps pipeline**.

# DevSecOps Lab 06 — Local CI/CD Security Pipeline

## Objective

This lab integrates the security controls introduced throughout the DevSecOps learning path into a single automated local pipeline.

The pipeline performs:

* Secrets Detection with Gitleaks
* Static Application Security Testing with Semgrep
* Software Composition Analysis with Trivy
* Docker Image Build
* Container Vulnerability Scanning with Trivy
* Test Environment Deployment
* Dynamic Application Security Testing with OWASP ZAP

The pipeline uses security-tool exit codes to enforce automated security gates.

---

## Pipeline Architecture

```text
Git Repository
      |
      v
Gitleaks
Secrets Detection
      |
      v
Semgrep
SAST
      |
      v
Trivy Filesystem
SCA
      |
      v
Docker Build
      |
      v
Trivy Image
Container Security
      |
      v
Test Deployment
      |
      v
OWASP ZAP
DAST
      |
   +--+--+
   |     |
 PASS   FAIL
   |     |
   v     X
Release Block
```

---

## Tools

* Ubuntu
* Bash
* Git
* Docker
* Gitleaks
* Semgrep
* Trivy
* OWASP ZAP
* Python
* Flask

---

## Project Structure

```text
06-cicd-integration/
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── semgrep-rules.yml
├── zap-rules.conf
├── pipeline.sh
└── reports/
```

---

# Application

The application exposes two endpoints:

```text
/
```

and:

```text
/health
```

The `/health` endpoint is used by the pipeline to verify that the temporary test deployment is ready before DAST begins.

The application also sets basic HTTP security headers including:

```text
X-Content-Type-Options
X-Frame-Options
Content-Security-Policy
Cache-Control
```

---

# Security Stages

## Stage 1 — Secrets Detection

Gitleaks scans the Git repository before any build operation occurs.

```text
Git History
     |
     v
Gitleaks
     |
  +--+--+
  |     |
Clean Secret
  |     |
  v     v
PASS   FAIL
```

The pipeline stops immediately when the secrets gate fails.

---

## Stage 2 — SAST

Semgrep analyzes application source code using a custom security rule.

The lab rule detects:

```python
os.system(...)
```

The scan runs with:

```text
--error
```

so findings are converted into a non-zero exit code and become a CI/CD security gate.

---

## Stage 3 — Software Composition Analysis

Trivy scans the project dependencies before the container image is built.

The policy used in this lab blocks:

```text
HIGH
CRITICAL
```

vulnerabilities that have available fixes.

Conceptually:

```text
requirements.txt
       |
       v
     Trivy
       |
       v
Known Vulnerabilities
       |
   +---+---+
   |       |
Allowed  Blocked
```

---

## Stage 4 — Container Build

If the source-code security stages pass, the application is built into a Docker image:

```text
devsecops-pipeline:local
```

The image runs the application as a dedicated non-root user.

---

## Stage 5 — Container Security

Trivy scans the final Docker image.

This stage analyzes the actual artifact that would eventually be published or deployed.

```text
Docker Image
     |
     +--> OS Packages
     |
     +--> Application Libraries
     |
     v
   Trivy
```

`HIGH` and `CRITICAL` vulnerabilities matching the configured policy block the pipeline.

---

## Stage 6 — Test Deployment

The image is started inside an isolated Docker network.

The pipeline performs a health check against:

```text
http://localhost:8080/health
```

The pipeline waits until the application becomes reachable or fails if the application never becomes healthy.

---

## Stage 7 — DAST

OWASP ZAP performs a Baseline Scan against the running test application.

The pipeline defines explicit DAST policy rules for:

```text
X-Frame-Options
X-Content-Type-Options
```

These alerts are configured as blocking findings.

Other warning-level findings are reported but do not automatically block this lab pipeline.

---

# Fail-Fast Behavior

The pipeline begins with:

```bash
set -Eeuo pipefail
```

As a result, a failing security gate prevents later delivery stages from running.

For example:

```text
Gitleaks
   |
 PASS
   |
Semgrep
   |
 FAIL
   |
   X
```

The pipeline does not proceed to:

```text
SCA
Docker Build
Container Scan
DAST
```

This implements a fail-fast DevSecOps workflow.

---

# Security Policy vs Security Scanner

A scanner identifies security findings.

A security policy determines what happens to those findings.

```text
Scanner
   |
   v
Finding
   |
   v
Policy
   |
 +─+──────+
 |        |
Allow    Block
```

Examples from this lab include:

```text
Trivy HIGH/CRITICAL
        |
        v
      BLOCK
```

and:

```text
ZAP normal warning
        |
        v
      REPORT
```

A mature DevSecOps implementation should define these policies explicitly rather than treating every security finding identically.

---

# Running the Pipeline

Make the script executable:

```bash
chmod +x pipeline.sh
```

Run:

```bash
./pipeline.sh
```

A successful execution ends with:

```text
============================================================
 DEVSECOPS PIPELINE PASSED
============================================================
```

---

# Testing the SAST Security Gate

Back up the safe application:

```bash
cp app.py app.py.safe
```

Introduce an intentionally insecure API:

```bash
sed -i '1i import os' app.py
```

```bash
sed -i '/def index():/a\    os.system("echo unsafe-test")' app.py
```

Verify:

```bash
grep -n "os.system" app.py
```

Run the pipeline:

```bash
./pipeline.sh
```

The expected flow is:

```text
Gitleaks
   |
 PASS
   |
Semgrep
   |
Finding
   |
 FAIL
```

The Docker build should not execute.

Restore the secure application:

```bash
mv app.py.safe app.py
```

Re-run:

```bash
./pipeline.sh
```

---

# Complete Command Reference

```bash
# ============================================================
# DevSecOps Lab 06 - Local CI/CD Security Pipeline
# ============================================================


# ------------------------------------------------------------
# 1. Create Lab Environment
# ------------------------------------------------------------

mkdir -p ~/devsecops-labs/06-cicd-integration

cd ~/devsecops-labs/06-cicd-integration

mkdir -p reports


# ------------------------------------------------------------
# 2. Create Dependency Manifest
# ------------------------------------------------------------

cat > requirements.txt <<'EOF'
Flask==3.1.3
EOF


# ------------------------------------------------------------
# 3. Create Application
# ------------------------------------------------------------

cat > app.py <<'EOF'
from flask import Flask, jsonify

app = Flask(__name__)


@app.after_request
def add_security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Cache-Control"] = "no-store"
    return response


@app.route("/")
def index():
    return jsonify(
        application="DevSecOps CI/CD Lab",
        status="running"
    )


@app.route("/health")
def health():
    return jsonify(status="healthy"), 200


if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=8080,
        debug=False
    )
EOF


# ------------------------------------------------------------
# 4. Create Semgrep Rule
# ------------------------------------------------------------

cat > semgrep-rules.yml <<'EOF'
rules:
  - id: lab.python.dangerous-os-system
    message: Avoid os.system(); passing untrusted data may lead to command injection.
    severity: ERROR
    languages:
      - python
    pattern: os.system(...)
EOF


# ------------------------------------------------------------
# 5. Create Dockerfile
# ------------------------------------------------------------

cat > Dockerfile <<'EOF'
FROM python:3.13-slim-bookworm

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd \
    --create-home \
    --uid 10001 \
    --shell /usr/sbin/nologin \
    appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8080

CMD ["python", "app.py"]
EOF


# ------------------------------------------------------------
# 6. Create .dockerignore
# ------------------------------------------------------------

cat > .dockerignore <<'EOF'
.git
.gitignore
reports
*.env
.env
__pycache__
*.pyc
README.md
pipeline.sh
semgrep-rules.yml
zap-rules.conf
EOF


# ------------------------------------------------------------
# 7. Create .gitignore
# ------------------------------------------------------------

cat > .gitignore <<'EOF'
reports/
.env
*.env
__pycache__/
*.pyc
EOF


# ------------------------------------------------------------
# 8. Create ZAP Policy
# ------------------------------------------------------------

cat > zap-rules.conf <<'EOF'
10020	FAIL	(X-Frame-Options Header Scanner)
10021	FAIL	(X-Content-Type-Options Header Missing)
EOF


# ------------------------------------------------------------
# 9. Initialize Git Repository
# ------------------------------------------------------------

git init

git config user.name "DevSecOps Lab"

git config user.email "devsecops-lab@example.local"

git add .

git commit -m "Initial secure DevSecOps application"

git status


# ------------------------------------------------------------
# 10. Create Pipeline
# ------------------------------------------------------------

cat > pipeline.sh <<'EOF'
#!/usr/bin/env bash

set -Eeuo pipefail

IMAGE_NAME="devsecops-pipeline:local"
CONTAINER_NAME="devsecops-pipeline-app"
NETWORK_NAME="devsecops-pipeline-net"
TRIVY_CACHE="${HOME}/.cache/trivy"

mkdir -p reports
mkdir -p "$TRIVY_CACHE"


stage() {
    echo
    echo "============================================================"
    echo " $1"
    echo "============================================================"
}


cleanup() {
    echo
    echo "[CLEANUP] Removing test resources..."

    docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
    docker network rm "$NETWORK_NAME" >/dev/null 2>&1 || true
}

trap cleanup EXIT


stage "STAGE 1/7 - Secrets Scan: Gitleaks"

docker run --rm \
    -v "$PWD:/repo" \
    -w /repo \
    ghcr.io/gitleaks/gitleaks:latest \
    git \
    --redact


stage "STAGE 2/7 - SAST: Semgrep"

docker run --rm \
    -v "$PWD:/src" \
    semgrep/semgrep:latest \
    semgrep scan \
    --error \
    --config /src/semgrep-rules.yml \
    /src/app.py


stage "STAGE 3/7 - SCA: Trivy Filesystem"

docker run --rm \
    -v "$PWD:/src:ro" \
    -v "$TRIVY_CACHE:/root/.cache/" \
    aquasec/trivy:latest \
    fs \
    --scanners vuln \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --exit-code 1 \
    /src


stage "STAGE 4/7 - Build Container Image"

docker build \
    -t "$IMAGE_NAME" \
    .


stage "STAGE 5/7 - Container Security: Trivy Image"

docker run --rm \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v "$TRIVY_CACHE:/root/.cache/" \
    aquasec/trivy:latest \
    image \
    --scanners vuln \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --exit-code 1 \
    "$IMAGE_NAME"


stage "STAGE 6/7 - Deploy Test Application"

docker network create "$NETWORK_NAME" >/dev/null

docker run -d \
    --name "$CONTAINER_NAME" \
    --network "$NETWORK_NAME" \
    -p 8080:8080 \
    "$IMAGE_NAME" >/dev/null


echo "Waiting for application health check..."

APP_READY=0

for attempt in $(seq 1 20); do

    if curl --silent --fail \
        http://localhost:8080/health >/dev/null; then

        APP_READY=1
        break
    fi

    sleep 1
done


if [ "$APP_READY" -ne 1 ]; then
    echo "Application failed health check."
    docker logs "$CONTAINER_NAME"
    exit 1
fi

echo "Application is healthy."


stage "STAGE 7/7 - DAST: OWASP ZAP"

docker run --rm \
    --network "$NETWORK_NAME" \
    -v "$PWD:/zap/wrk/:rw" \
    ghcr.io/zaproxy/zaproxy:stable \
    zap-baseline.py \
    -t "http://${CONTAINER_NAME}:8080" \
    -m 1 \
    -I \
    -c zap-rules.conf \
    -r reports/zap-report.html \
    -J reports/zap-report.json


echo
echo "============================================================"
echo " DEVSECOPS PIPELINE PASSED"
echo "============================================================"
EOF


# ------------------------------------------------------------
# 11. Make Pipeline Executable
# ------------------------------------------------------------

chmod +x pipeline.sh


# ------------------------------------------------------------
# 12. Download Required Images
# ------------------------------------------------------------

docker pull ghcr.io/gitleaks/gitleaks:latest

docker pull semgrep/semgrep:latest

docker pull aquasec/trivy:latest

docker pull ghcr.io/zaproxy/zaproxy:stable

docker pull python:3.13-slim-bookworm


# ------------------------------------------------------------
# 13. Run Complete DevSecOps Pipeline
# ------------------------------------------------------------

./pipeline.sh


# ------------------------------------------------------------
# 14. Inspect DAST Reports
# ------------------------------------------------------------

ls -lh reports/

python3 -m json.tool \
    reports/zap-report.json | less


# ------------------------------------------------------------
# 15. Optional - Open HTML Report
# ------------------------------------------------------------

xdg-open reports/zap-report.html


# ------------------------------------------------------------
# 16. Intentionally Break SAST Gate
# ------------------------------------------------------------

cp app.py app.py.safe

sed -i '1i import os' app.py

sed -i '/def index():/a\    os.system("echo unsafe-test")' app.py

grep -n "os.system" app.py


# ------------------------------------------------------------
# 17. Verify Pipeline Blocks Vulnerable Code
# ------------------------------------------------------------

./pipeline.sh


# ------------------------------------------------------------
# 18. Restore Secure Application
# ------------------------------------------------------------

mv app.py.safe app.py


# ------------------------------------------------------------
# 19. Re-run Pipeline
# ------------------------------------------------------------

./pipeline.sh
```

---

# Final DevSecOps Workflow

The complete learning path now implements:

```text
                    SOFTWARE DELIVERY

                          |
                          v

                    Source Code
                          |
                          v
                    +-----------+
                    | Gitleaks  |
                    |  Secrets  |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    | Semgrep   |
                    |   SAST    |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    |  Trivy    |
                    |   SCA     |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    |  Docker   |
                    |   Build   |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    |  Trivy    |
                    | Container |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    | Test Env  |
                    +-----+-----+
                          |
                          v
                    +-----------+
                    | OWASP ZAP |
                    |   DAST    |
                    +-----+-----+
                          |
                     +----+----+
                     |         |
                    PASS      FAIL
                     |         |
                     v         X
                  Release    Block
```

---

# Key Takeaway

DevSecOps is not:

```text
DevOps + a collection of security scanners
```

A better model is:

```text
DevSecOps
   =
Security Controls
   +
Security Policies
   +
Automation
   +
Continuous Feedback
```

The fundamental workflow is:

```text
Detect
  |
  v
Evaluate Policy
  |
  v
Block or Allow
  |
  v
Remediate
  |
  v
Re-run
```

That is the foundation of a real DevSecOps delivery pipeline.

