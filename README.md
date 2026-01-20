# 🚀 End-to-End Cloud Deployment Project

**Python App → Docker → AWS EC2 → ECR → IAM → VPC → GitHub Actions CI/CD**

---

## 1. Project Purpose

This project was built to **understand cloud deployment properly**.

The goals were:

* A small Python web application (simple HTTP service)
* Containerized with **Docker**
* Source code in **Github*
* CI/CD using **GitHub Actions**
* Image stored in **Amazon ECR**
* App runs on **Amazon EC2**
* Network isolated using **Amazon VPC**
* Secrets handled via **AWS Secrets Manager** and **AWS Systems Manager Parameter Store**
* DNS via **Amazon Route 53**
* Build a **solid foundation** before scaling or using managed services

---

## 2. Final Architecture (What We Built)

### High-Level Flow

```
User (Browser / curl)
        |
        v
Public IP (EC2)
        |
Security Group (port rules)
        |
EC2 Host OS
        |
Docker Port Mapping
        |
Docker Container
        |
Python Flask Application
```

### CI/CD Flow

```
Git Push (main branch)
        |
GitHub Actions
        |
Docker Build
        |
Push Image to Amazon ECR
        |
SSH into EC2
        |
Pull Latest Image
        |
Restart Container
```

---

## 3. Application Layer

### What the App Does

* Python Flask web application
* Stateless (no database, no files)
* Two endpoints:

  * `/` → returns app name and environment
  * `/health` → returns `{ "status": "ok" }`

### Why the App Is Simple

Because this project is about **infrastructure**, not features.

A complex app would:

* Hide networking problems
* Hide deployment failures
* Distract from learning goals

---

## 4. Environment Variables (Important Design Choice)

The app **requires** these environment variables:

| Variable      | Purpose         |
| ------------- | --------------- |
| `APP_NAME`    | App identity    |
| `ENVIRONMENT` | Runtime context |
| `SECRET_KEY`  | Example secret  |

### Intentional Behavior

If **any variable is missing**, the app:

* Prints an error
* Exits immediately

This is deliberate.

> Silent misconfiguration is worse than crashing.

---

## 5. Local Execution (Before Docker)

### Why This Step Exists

If an app doesn’t work **locally**, Docker and AWS will not fix it.

### First Expected Error

Running without env variables resulted in:

```
Missing required environment variables
```

This was **correct behavior**, not a bug.

Only after exporting variables did the app start successfully.

---

## 6. Dockerization

### Why Docker Was Used

Docker provides:

* Reproducible runtime
* Dependency isolation
* Consistent behavior across machines

### Key Docker Concepts Learned

* Image ≠ Container
* `EXPOSE` is documentation only
* Port mapping must be explicit (`-p`)
* Container exits when main process exits

### Docker Validation

The container was run locally and tested with:

```
curl http://localhost:8000
```

Only after this worked did we move to AWS.

---

## 7. GitHub Repository

### Purpose

* Source of truth
* CI/CD trigger
* Versioned history

### What Was Committed

* Application code
* Dockerfile
* Requirements file
* `.gitignore`

### What Was Never Committed

* Secrets
* AWS credentials
* Virtual environments

---

## 8. Amazon ECR (Container Registry)

### What ECR Is

* Private Docker image registry
* IAM-controlled
* Stores images only

### What ECR Is NOT

* Not a runtime
* Not a deployment system
* Not a server

---

## 9. IAM (Identity and Permissions)

### Core Rule Learned

> IAM controls **who can do what**, not how things work.

### Roles Used

#### EC2 Instance Role

Allows EC2 to:

* Pull images from ECR
* Read secrets from Secrets Manager
* Read parameters from Parameter Store

No admin permissions.

#### GitHub Actions Role

Allows CI/CD to:

* Authenticate via OIDC
* Push images to ECR

No EC2 access.

---

## 10. Networking (Most Important Section)

### VPC

* CIDR: `10.0.0.0/16`
* Isolated network
* No traffic allowed by default

### Subnet

* Public subnet: `10.0.1.0/24`
* Auto-assign public IPv4 enabled

### Internet Gateway

* Attached to VPC
* Allows internet access **only if routed**

### Route Table

Critical route added:

```
0.0.0.0/0 → Internet Gateway
```

Without this, public IPs are useless.

---

## 11. Security Groups

### What Security Groups Do

* Stateful firewall
* Default deny
* Port-based rules

### Required Inbound Rules

| Port | Source    |
| ---- | --------- |
| 22   | Your IP   |
| 8000 | 0.0.0.0/0 |

---

## 12. EC2 Instance

### Configuration

* Amazon Linux 2023
* Public subnet
* Public IP enabled
* IAM role attached
* Security group attached

Docker was installed manually to understand runtime behavior.

---

## 13. Secrets and Configuration

### Why Hardcoding Was Removed

Hardcoding:

* Breaks security
* Breaks CI/CD
* Breaks portability

### Correct Separation

| Type              | Service         |
| ----------------- | --------------- |
| Non-secret config | Parameter Store |
| Secrets           | Secrets Manager |

Values were fetched **at runtime**, not baked into images.

---

## 14. Major Errors Faced (In Real Order)

### ❌ Error 1: App Worked Internally but Not Publicly

* `curl 172.x.x.x:8000` → worked
* `curl public-ip:8000` → hung

**Cause**: Missing or wrong security group rule
**Fix**: Allow inbound TCP 8000

---

### ❌ Error 2: Public IP Metadata Returned Empty

Command returned nothing:

```
curl http://169.254.169.254/latest/meta-data/public-ipv4
```

**Cause**: Amazon Linux 2023 enforces IMDSv2
**Fix**: Use token-based metadata access

---

### ❌ Error 3: SSH Worked but App Didn’t

**Mistake**: Assuming SSH success means app is reachable
**Reality**: Each port is independent

**Fix**: Explicitly allow port 8000

---

### ❌ Error 4: Curling Public IP from Inside EC2

Curl hung when run **from the same EC2**.

**Cause**: AWS does not guarantee hairpin NAT
**Lesson**: Public IP must be tested from outside AWS

---

### ❌ Error 5: GitHub Actions ECR Permission Denied

Error:

```
not authorized to perform ecr:InitiateLayerUpload
```

**Cause**: Incomplete IAM policy
**Fix**: Add full ECR push permissions

---

### ❌ Error 6: ECR Repository “Does Not Exist”

Even though repo existed.

**Cause**: Incorrect registry URI construction
**Fix**: Use ECR login output instead of manual strings

---

### ❌ Error 7: Invalid GitHub Actions YAML

Workflow failed before execution.

**Cause**: Incorrect YAML indentation
**Fix**: Proper indentation and structure

---

### ❌ Error 8: CI/CD Succeeded but App Not Reachable

**Cause**: Security group edited earlier but later changed
**Fix**: Re-verify **actual attached security group**

---

## 15. Debugging Method Used (Critical Skill)

Every issue was debugged using this order:

1. Is the app running?
2. Is the container running?
3. Is the port listening on host?
4. Does internal curl work?
5. Does EC2 have public IP?
6. Does SG allow the port?
7. Does route table point to IGW?
8. Test from external machine

No guessing. No random fixes.

---

## 16. CI/CD (GitHub Actions)

### What CI/CD Does Here

* Builds Docker image
* Pushes to ECR using OIDC
* SSHs into EC2
* Pulls latest image
* Restarts container

### What CI/CD Does NOT Do

* Manage networking
* Fix security groups
* Create infrastructure

Automation replaces **manual steps**, not understanding.

---

## 17. DNS and Domain (Decision Taken)

* Route 53 concepts were learned
* Domain purchase was skipped intentionally
* Public IP is sufficient for a learning project

This avoids unnecessary cost while retaining full understanding.

---

## 18. Final State

At the end of this project:

* ✅ App is publicly reachable
* ✅ CI/CD is fully automated
* ✅ No secrets in code or images
* ✅ IAM is least-privileged
* ✅ Networking is understood and debuggable
* ✅ Errors are reproducible and explainable

---

## 19. Key Takeaways (Most Important)

* Cloud failures are deterministic, not random
* Most issues are networking or IAM
* Docker does not manage AWS networking
* SSH success proves nothing about app ports
* Public IP ≠ reachable service
* Automation without understanding is dangerous

---

## Status

**Project Phase:** Manual Deployment + CI/CD
**Result:** ✅ Completed Successfully

---
