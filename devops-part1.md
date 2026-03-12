

# 🚀 DEVOPS MASTERY ROADMAP — FROM ZERO TO JOB-READY

> **For:** BTech CSE student (India) | **Goal:** Campus Placements + Portfolio + Deep Understanding
> **Scope:** This is Part 1 — Sections 1–3 + Phase 1–2 Details. Ask for "Part 2" to continue.

---

---

## 1️⃣ PHASE OVERVIEW TABLE

| Phase | Focus | Key Skills | Outcome | Time (2 hr/day) |
|-------|-------|-----------|---------|-----------------|
| **0** | Orientation & Mindset | What DevOps is, career paths, mental models | Clear vision, no confusion | 2–3 days |
| **1** | Fundamentals | Linux, Terminal, Networking basics, Git | Navigate Linux, push code to GitHub, understand how internet works | 5–6 weeks |
| **2** | Core Concepts | Bash/Python scripting, Docker, Web servers, Cloud basics (AWS) | Containerize apps, deploy to cloud, write automation scripts | 7–8 weeks |
| **3** | Intermediate | CI/CD (GitHub Actions, Jenkins), Terraform, Ansible, K8s intro | Build full pipelines, provision infra as code | 7–8 weeks |
| **4** | Advanced | Kubernetes deep dive, Monitoring (Prometheus+Grafana), Logging (ELK), Secrets | Run production-grade clusters, set up observability | 6–8 weeks |
| **5** | Expert / Mastery | GitOps (ArgoCD), Service Mesh, DevSecOps, Cloud Design Patterns | Architect complex systems, pass senior-level interviews | 5–6 weeks |
| **6** | Real-World Application | Portfolio projects, open source, interview prep, certifications | Resume-ready portfolio, crack interviews | Ongoing |

> **Total: ~32–40 weeks at 2 hr/day** (8–10 months). Faster on vacation with Beast mode.

---

---

## 2️⃣ SKILL TREE (VISUAL MAP)

```
DEVOPS
│
├── 🟢 FUNDAMENTALS [Phase 1] ─────────────────────────────────────────
│   ├── Linux OS Basics
│   │   ├── File System & Navigation
│   │   ├── Users & Permissions
│   │   └── Package Management
│   ├── Terminal Mastery
│   │   ├── Core Commands (ls, grep, find, awk, sed)
│   │   ├── Process Monitoring (ps, top, htop, kill)
│   │   ├── Networking Tools (curl, wget, netstat, ss, ping, traceroute)
│   │   └── Text Manipulation (grep, sed, awk, cut, sort)
│   ├── Networking Fundamentals
│   │   ├── OSI Model & TCP/IP
│   │   ├── DNS, HTTP/HTTPS
│   │   ├── SSL/TLS, SSH
│   │   └── FTP/SFTP, Ports, Firewalls
│   └── Git & GitHub
│       ├── Commits, Branches, Merges
│       ├── Pull Requests, Conflicts
│       └── Git Workflows (GitFlow, Trunk-based)
│
├── 🟡 CORE CONCEPTS [Phase 2] ────────────────────────────────────────
│   ├── Scripting ─────────────────→ depends on Terminal
│   │   ├── Bash Scripting
│   │   └── Python for DevOps
│   ├── Web Servers ───────────────→ depends on Networking
│   │   ├── Nginx (reverse proxy, load balancer)
│   │   └── Forward Proxy vs Reverse Proxy vs Load Balancer
│   ├── Docker ────────────────────→ depends on Linux, Networking
│   │   ├── Images, Containers, Volumes
│   │   ├── Dockerfile, Docker Compose
│   │   └── Networking in Docker
│   └── Cloud Basics (AWS) ───────→ depends on Networking, Linux
│       ├── EC2, S3, VPC, IAM
│       ├── Security Groups, Key Pairs
│       └── AWS CLI
│
├── 🟠 INTERMEDIATE [Phase 3] ─────────────────────────────────────────
│   ├── CI/CD ─────────────────────→ depends on Git, Docker
│   │   ├── GitHub Actions
│   │   └── Jenkins
│   ├── Infrastructure as Code ────→ depends on Cloud, Networking
│   │   └── Terraform (HCL language)
│   ├── Configuration Management ──→ depends on Linux, Scripting
│   │   └── Ansible (YAML playbooks)
│   └── Kubernetes Intro ──────────→ depends on Docker, Networking
│       ├── Pods, Services, Deployments
│       └── kubectl, YAML manifests
│
│   ║ CI/CD and Terraform can be learned IN PARALLEL ║
│   ║ Ansible and K8s Intro can be learned IN PARALLEL ║
│
├── 🔴 ADVANCED [Phase 4] ─────────────────────────────────────────────
│   ├── Kubernetes Deep Dive ──────→ depends on K8s Intro
│   │   ├── Ingress, ConfigMaps, Secrets
│   │   ├── StatefulSets, DaemonSets
│   │   ├── Helm Charts
│   │   ├── RBAC, Network Policies
│   │   └── Managed K8s (EKS/GKE)
│   ├── Monitoring ────────────────→ depends on K8s, Cloud
│   │   ├── Prometheus + Grafana
│   │   └── Application Monitoring (OpenTelemetry, Jaeger)
│   ├── Logs Management ───────────→ depends on K8s, Docker
│   │   └── ELK Stack (Elasticsearch, Logstash, Kibana) / Loki
│   └── Secret Management ────────→ depends on K8s, Cloud
│       └── HashiCorp Vault
│
│   ║ Monitoring and Logging can be learned IN PARALLEL ║
│
├── ⚫ EXPERT [Phase 5] ───────────────────────────────────────────────
│   ├── GitOps ────────────────────→ depends on K8s, CI/CD, Git
│   │   └── ArgoCD
│   ├── Service Mesh ──────────────→ depends on K8s Deep Dive
│   │   └── Istio (awareness level)
│   ├── DevSecOps ─────────────────→ depends on CI/CD, Docker, K8s
│   │   ├── Container Scanning (Trivy)
│   │   ├── SAST/DAST in pipelines
│   │   └── Policy as Code (OPA)
│   └── Cloud Design Patterns ────→ depends on Cloud, K8s
│       ├── High Availability
│       ├── Disaster Recovery
│       └── Cost Optimization
│
│   ║ GitOps, Service Mesh, DevSecOps can be PARALLEL ║
│
└── 🏆 REAL-WORLD [Phase 6] ──────────────────────────────────────────
    ├── Portfolio Projects (5 resume-worthy)
    ├── Open Source Contributions
    ├── Interview Preparation
    └── Optional: AWS SAA / CKA Certification
```

---

---

## 3️⃣ PHASE 0 — ORIENTATION & MINDSET

### What is DevOps in Simplest Language?

**DevOps** is a set of practices, tools, and a cultural philosophy that brings together **software development** (Dev) and **IT operations** (Ops). Instead of developers writing code and "throwing it over the wall" to a separate operations team to deploy and manage, DevOps makes the same team (or tightly collaborating teams) responsible for writing, testing, deploying, monitoring, and maintaining the software — **automatically and continuously**. Think of it as building a highway between "code written" and "code running for real users" and then automating the entire journey.

---

### Why Does DevOps Exist? What Problem Does It Solve?

- **Before DevOps:** Dev team writes code → hands to QA → hands to Ops → Ops manually deploys → something breaks → blame game → takes weeks/months to release
- **With DevOps:** Code pushed → automatically tested → automatically deployed → automatically monitored → fix issues in minutes → release multiple times per day
- **Core problems solved:**
  - Slow, manual, error-prone deployments
  - "It works on my machine" syndrome
  - No visibility into what's happening in production
  - Months-long release cycles (now: minutes)
  - Blame culture between Dev and Ops

---

### Who Uses DevOps in Industry?

| Company | How They Use DevOps |
|---------|-------------------|
| **Amazon** | Deploys code every 11.7 seconds using automated pipelines |
| **Netflix** | Pioneered chaos engineering; auto-scales across regions |
| **Google** | Invented SRE (Site Reliability Engineering), the Google version of DevOps |
| **Flipkart** | Handles Big Billion Day traffic with auto-scaling K8s clusters |
| **Razorpay** | CI/CD pipelines for payment infra with zero-downtime deploys |
| **Zomato** | Kubernetes-based microservices with Prometheus monitoring |
| **Swiggy** | Terraform-managed AWS infra, GitOps workflows |
| **CRED** | Fully Dockerized services, ArgoCD for GitOps |

---

### Common Myths & Misconceptions

| # | Myth | Reality |
|---|------|---------|
| 1 | "DevOps is a job title" | DevOps is a **culture/practice**. "DevOps Engineer" is a role that implements these practices. |
| 2 | "DevOps = just CI/CD tools" | Tools are 30%. Culture, automation mindset, and collaboration are 70%. |
| 3 | "I need to be a sysadmin first" | No. You need Linux basics + coding. Deep sysadmin skills come gradually. |
| 4 | "DevOps means no manual work" | Goal is to automate repetitive tasks. Thinking, designing, debugging remain human. |
| 5 | "Docker = DevOps" | Docker is ONE tool. DevOps includes networking, CI/CD, monitoring, IaC, and much more. |
| 6 | "DevOps isn't for freshers" | Many companies (including FAANG) hire freshers for DevOps/SRE/Platform Engineering roles from campus. |
| 7 | "Coding isn't needed" | You WILL write Python, Bash, YAML, HCL, Go daily. DevOps engineers are engineers. |
| 8 | "Cloud = DevOps" | Cloud is a platform. DevOps practices can run on bare metal, VMs, or cloud. |

---

### Right Mindset Before Starting

- ✅ **Automate everything that repeats** — if you do it twice, script it
- ✅ **Break things intentionally** — learning happens when things fail; use VMs/containers freely
- ✅ **"Infrastructure as Code" thinking** — treat servers like code: version-controlled, reproducible, disposable
- ✅ **T-shaped skills** — go deep in 2–3 tools, but be aware of the entire ecosystem
- ✅ **Hands-on > theory** — watching a Docker tutorial ≠ knowing Docker. Run it. Break it. Fix it.
- ✅ **Read error messages** — 80% of DevOps debugging is reading logs and error messages carefully
- ✅ **Comfort with the terminal** — GUI is a crutch. The terminal is your home now.
- ✅ **Patience with networking** — networking concepts feel abstract initially but are the backbone of EVERYTHING

---

### How DevOps Fits into CSE Career Paths

| Role | What You Do | Avg CTC (India, fresher) | DevOps Overlap |
|------|------------|-------------------------|----------------|
| **DevOps Engineer** | Build CI/CD, manage infra, automate deployments | ₹8–25 LPA | 100% |
| **SRE (Site Reliability Engineer)** | Ensure uptime, incident response, monitoring | ₹12–35 LPA | 85% (+ coding) |
| **Platform Engineer** | Build internal developer platforms, tooling | ₹12–30 LPA | 80% |
| **Cloud Engineer** | Design and manage cloud infrastructure | ₹8–22 LPA | 70% |
| **Backend Engineer** | Build APIs, services (needs deployment knowledge) | ₹8–30 LPA | 30–40% |
| **Full-Stack Developer** | Frontend + Backend + basic deployment | ₹6–20 LPA | 20–30% |
| **Security Engineer (DevSecOps)** | Secure pipelines, infra, containers | ₹10–28 LPA | 60% |

> **Key insight:** DevOps skills make you MORE VALUABLE in ANY engineering role. Even backend engineers who understand Docker, CI/CD, and cloud get promoted faster.

---

---

## 4️⃣ PHASE 1 — FUNDAMENTALS

---

### 🎯 Phase Goal

> **"Navigate a Linux system confidently via terminal, understand how the internet works (DNS, HTTP, SSH, ports), and use Git/GitHub like a professional developer — all without a GUI."**

---

### 📚 Skills Table

| # | Skill | Key Concepts | Difficulty | Interview Freq |
|---|-------|-------------|------------|---------------|
| 1 | Linux OS Basics | Distros, file system hierarchy (/etc, /var, /home), boot process | 🟢 Easy | 🔥 High |
| 2 | File System Navigation | ls, cd, pwd, mkdir, rm, cp, mv, find, locate, tree | 🟢 Easy | 🔥 High |
| 3 | Users & Permissions | chmod, chown, chgrp, rwx model, sudo, root vs user, /etc/passwd | 🟢 Easy | 🔥 High |
| 4 | Package Management | apt (Debian/Ubuntu), yum/dnf (RHEL), installing/removing/updating packages | 🟢 Easy | 🌤️ Medium |
| 5 | Text Editors (Terminal) | Vim basics (i, Esc, :wq, :q!, dd, yy, p), nano as backup | 🟡 Medium | 🌤️ Medium |
| 6 | Process Management | ps, top, htop, kill, systemctl, services, daemons, PID, foreground/background | 🟡 Medium | 🔥 High |
| 7 | Text Manipulation | grep, sed, awk, cut, sort, uniq, wc, head, tail, pipes (\|), redirection (>, >>, <) | 🟡 Medium | 🔥 High |
| 8 | Networking Tools | ping, traceroute, curl, wget, netstat, ss, nslookup, dig, ifconfig/ip | 🟡 Medium | 🔥 High |
| 9 | Networking Fundamentals | OSI model, TCP vs UDP, IP addressing, subnets, ports, DNS resolution | 🟡 Medium | 🔥 High |
| 10 | HTTP/HTTPS | Request/response cycle, methods (GET/POST), status codes, headers | 🟢 Easy | 🔥 High |
| 11 | SSH & SSL/TLS | SSH key-pair auth, ssh-keygen, scp, SSL certificates, TLS handshake | 🟡 Medium | 🔥 High |
| 12 | DNS Deep Dive | A record, CNAME, MX, NS, TTL, how DNS resolves step-by-step | 🟡 Medium | 🔥 High |
| 13 | Firewall Basics | iptables/ufw, ports, allowing/blocking traffic, security groups concept | 🟡 Medium | 🌤️ Medium |
| 14 | Git Fundamentals | init, add, commit, status, log, diff, .gitignore | 🟢 Easy | 🔥 High |
| 15 | Git Branching & Merging | branch, checkout, merge, rebase, conflicts, stash | 🟡 Medium | 🔥 High |
| 16 | GitHub Workflow | Push, pull, fork, clone, Pull Requests, code review, Issues | 🟢 Easy | 🔥 High |
| 17 | Git Workflows | GitFlow, Trunk-based development, feature branches | 🟡 Medium | 🌤️ Medium |

---

### 💡 Mental Models & Analogies

| Concept | Real-World Analogy |
|---------|--------------------|
| **Linux File System** | Think of it as a building: `/` is the ground floor (root), `/home` is your apartment, `/etc` is the manager's office (config files), `/var` is the warehouse (logs, data), `/tmp` is a whiteboard (temporary, erased on reboot) |
| **Permissions (rwx)** | Like a library book: **r** = you can read it, **w** = you can write/edit in it, **x** = you can run/perform it (like running a machine). Owner/Group/Others = the person who bought it, their family, and strangers |
| **Pipes (\|)** | An assembly line in a factory. Output of one machine (command) becomes input of the next. `cat file.txt \| grep "error" \| wc -l` = "read file → filter errors → count them" |
| **Processes & PIDs** | Every running program is a worker in an office. PID is their employee ID badge. `kill` = telling a specific worker to stop. `systemctl` = HR managing departments (services) |
| **OSI Model** | Sending a letter: Layer 7 (Application) = you write the letter; Layer 4 (Transport) = postal service decides truck vs plane (TCP vs UDP); Layer 3 (Network) = GPS routing; Layer 1 (Physical) = the actual road |
| **DNS** | Phone book of the internet. You say "google.com" → DNS looks up the actual address (IP: 142.250.x.x) → your browser goes there |
| **HTTP request** | Ordering food at a restaurant. GET = "show me the menu." POST = "I want to order this." Status 200 = "here's your food." Status 404 = "we don't have that dish." Status 500 = "the kitchen caught fire" |
| **SSH** | A secret tunnel into a remote computer. Key pair = you have a key (private), the server has the lock (public key). Only YOUR key opens THAT lock. No password needed. |
| **SSL/TLS** | A tamper-proof envelope for your letter. Even if someone intercepts it, they can't read or modify it. The padlock icon in your browser = "this envelope is sealed" |
| **Git commits** | Save points in a video game. Each commit = a snapshot. You can always go back to any save point if you mess up |
| **Git branches** | Parallel universes. `main` is the real world. A branch = "what if I try this experiment?" If it works → merge back to reality. If not → delete the universe, no harm done |
| **Firewall** | Security guard at a building entrance. Rules say "let delivery guys in (port 80/443), block strangers (everything else)" |

---

### 🛠️ Practice

| Skill | Exercise / Problem | Platform | Difficulty |
|-------|--------------------|----------|------------|
| Linux Basics | Complete "Linux Survival" interactive tutorial | linuxsurvival.com | 🟢 |
| Linux Basics | Solve Bandit wargame Levels 0–15 | overthewire.org | 🟢→🟡 |
| File Navigation | "Find all .log files > 10MB, sort by size" — do on your VM | Local VM | 🟢 |
| Permissions | Create 3 users, set up shared folder with specific rwx for each | Local VM | 🟡 |
| Text Manipulation | Extract all IP addresses from a log file using grep + awk | Local VM | 🟡 |
| Text Manipulation | "Linux Text Processing" challenges | HackerRank (Linux Shell) | 🟡 |
| Process Management | Start a background process, find its PID, kill it gracefully | Local VM | 🟢 |
| Vim | Complete `vimtutor` (built into Linux, just type `vimtutor`) | Terminal | 🟡 |
| Networking | Trace the DNS resolution of 5 domains using `dig` + `nslookup` | Local VM | 🟡 |
| Networking | "Networking" section challenges | HackerRank | 🟡 |
| HTTP | Use `curl` to make GET and POST requests to `httpbin.org` | Terminal | 🟢 |
| SSH | Generate SSH key pair, SSH into a remote VM (use free AWS EC2 or DigitalOcean) | Cloud VM | 🟡 |
| SSH | Set up passwordless SSH between two machines/VMs | Local VMs | 🟡 |
| Git | Complete "Learn Git Branching" interactive tutorial (ALL levels) | learngitbranching.js.org | 🟢→🟡 |
| Git | Simulate merge conflict → resolve it manually | Local | 🟡 |
| GitHub | Create a repo, make 3 branches, open PRs, merge with review comments | GitHub | 🟢 |
| Git | "Introduction to Git" challenges | HackerRank | 🟢 |
| Full Phase 1 | Set up Ubuntu VM, SSH into it, install Nginx, check access via curl, view logs with grep | Local VM | 🟡 |

---

### 🧪 Milestone Checkpoint

> ✅ **You're ready for Phase 2 ONLY when you can:**
> 1. Open a terminal on a Linux VM and navigate to any directory, create/delete files, change permissions — **without Googling commands**
> 2. Find a specific error in a 10,000-line log file using `grep`, `awk`, and pipes — **in under 2 minutes**
> 3. Explain what happens when you type `google.com` in a browser — from DNS resolution to HTTP response — **step by step, in your own words**
> 4. SSH into a remote machine using key-based auth — **without looking at notes**
> 5. Create a Git repo, make branches, create a merge conflict, and resolve it — **confidently**
> 6. Explain the difference between TCP and UDP with an example — **using your own analogy**

---

### ⚠️ Common Mistakes

| # | Mistake | Why It Happens | Fix |
|---|---------|---------------|-----|
| 1 | Running everything as `root` | "sudo makes errors go away" | Create a regular user. Use sudo ONLY when needed. Understand WHY it's needed |
| 2 | Memorizing commands without understanding | Tutorial-following mode | After learning a command, EXPLAIN it to yourself: "what does each flag do?" |
| 3 | Skipping networking | "Boring theory, I want Docker" | Docker, Kubernetes, Cloud — ALL break without networking knowledge. Invest now |
| 4 | Using GUI file manager on Linux | Comfort zone | Force yourself to use terminal for 2 weeks. It becomes natural |
| 5 | Making Git commits without messages | Laziness | Every commit message should explain WHY, not WHAT. "Fix login bug" not "update" |
| 6 | Not practicing SSH | "I'll learn when I need it" | You'll need it EVERY DAY in DevOps. Set it up now on a free VM |
| 7 | Ignoring file permissions | "chmod 777 fixes everything" | 777 = anyone can do anything = security disaster. Learn proper permission model |
| 8 | Not reading `man` pages | "Too long, I'll Google" | `man grep` is faster and more accurate than random blog posts. Learn to read man pages |

---

### 🚫 What NOT To Learn Yet

| Topic | Why Skip Now | Learn In Phase |
|-------|-------------|---------------|
| Docker | Need Linux + Networking first | Phase 2 |
| Cloud (AWS/Azure) | Need networking + SSH + Linux admin first | Phase 2 |
| Bash scripting (advanced) | Need terminal comfort first | Phase 2 |
| Kubernetes | Need Docker + networking first | Phase 3–4 |
| CI/CD pipelines | Need Git + Docker first | Phase 3 |
| Terraform | Need Cloud basics first | Phase 3 |
| Networking deep dive (subnets, CIDR) | Basic DNS/HTTP/SSH is enough for now | Phase 2–3 |
| Windows Server | Linux dominates DevOps; Windows is niche | Skip / on-the-job |

---

### 📖 Resources (FREE First)

| Rank | Resource | Type | Free/Paid |
|------|----------|------|-----------|
| 🥇 | **Linux full course — TrainWithShubham (YouTube)** | Video (Hindi) | Free |
| 🥇 | **Computer Networking Full Course — Kunal Kushwaha (YouTube)** | Video (Hindi/Eng) | Free |
| 🥇 | **Learn Git Branching — learngitbranching.js.org** | Interactive | Free |
| 🥇 | **Bandit Wargame — overthewire.org** | Hands-on | Free |
| 🥈 | **The Missing Semester of CS (MIT)** — Linux & Git lectures | Video + Notes | Free |
| 🥈 | **Linux Basics for Hackers (book by OccupyTheWeb)** | Book | Paid (optional) |
| 🥈 | **Git & GitHub — CodeHelp (Love Babbar, YouTube)** | Video (Hindi) | Free |
| 🥈 | **Networking Fundamentals — NetworkChuck (YouTube)** | Video (Eng) | Free |
| 🥉 | **HackerRank Linux Shell challenges** | Practice | Free |
| 🥉 | **freeCodeCamp — Git for Professionals (YouTube)** | Video (Eng) | Free |
| 🥉 | **NPTEL — Computer Networks (IIT)** | Video (Eng) | Free |

---

### 💼 Interview Relevance

**Companies asking Phase 1 topics:** Literally ALL tech companies — Amazon, Google, Microsoft, Flipkart, Razorpay, Zomato, Swiggy, Atlassian, Uber, Goldman Sachs (infra teams)

**Frequency:** 🔥🔥🔥 (Every DevOps/SRE interview starts here)

| Question | Approach Hint |
|----------|--------------|
| "What happens when you type `google.com` in the browser?" | DNS lookup → TCP handshake → TLS handshake → HTTP GET → server processes → HTTP response → browser renders. Mention each layer briefly |
| "Explain Linux file permissions. What does `chmod 755` mean?" | Owner=rwx (7), Group=r-x (5), Others=r-x (5). Convert binary: 7=111=rwx. Explain who is owner/group/others |
| "What's the difference between `git merge` and `git rebase`?" | Merge creates a merge commit (preserves history). Rebase replays your commits on top of target branch (linear history). Mention when to use each |
| "How does SSH work?" | Asymmetric key pair: client has private key, server has public key. Client proves identity without sending password. Mention key exchange |
| "What is the OSI model? Explain each layer briefly." | Physical → Data Link → Network → Transport → Session → Presentation → Application. Give one example per layer. Mention that real world uses TCP/IP model (4 layers) |
| "How does DNS resolution work?" | Browser cache → OS cache → Recursive resolver → Root nameserver → TLD nameserver → Authoritative nameserver → IP returned → cached |

---

### ⏱️ Time Estimate

| Sub-topic | Hours |
|-----------|-------|
| Linux basics + terminal | 20–25 hrs |
| Text manipulation & process mgmt | 10–15 hrs |
| Networking fundamentals | 15–20 hrs |
| SSH, SSL/TLS, Firewall basics | 8–10 hrs |
| Git & GitHub | 12–15 hrs |
| Practice & milestone exercises | 10–15 hrs |
| **Total** | **~75–100 hrs (5–6 weeks @ 2hr/day)** |

---

---

## 4️⃣ PHASE 2 — CORE CONCEPTS

---

### 🎯 Phase Goal

> **"Write Bash & Python automation scripts, containerize any application with Docker, set up Nginx as a web server/reverse proxy, and deploy a basic app on AWS EC2 — all from scratch."**

---

### 📚 Skills Table

| # | Skill | Key Concepts | Difficulty | Interview Freq |
|---|-------|-------------|------------|---------------|
| 1 | Bash Scripting | Variables, conditionals, loops, functions, exit codes, cron jobs | 🟡 Medium | 🔥 High |
| 2 | Python for DevOps | File I/O, os/sys modules, subprocess, requests, json, boto3 (AWS SDK), argparse | 🟡 Medium | 🔥 High |
| 3 | YAML & JSON | Syntax, nesting, lists, dictionaries — used in EVERY DevOps tool | 🟢 Easy | 🔥 High |
| 4 | Web Server Concepts | Forward proxy, reverse proxy, load balancer, caching — what each does | 🟡 Medium | 🔥 High |
| 5 | Nginx | Install, configure, serve static site, reverse proxy to app, SSL termination | 🟡 Medium | 🔥 High |
| 6 | Docker Fundamentals | Images, containers, registries, Docker daemon, container vs VM | 🟡 Medium | 🔥 High |
| 7 | Dockerfile | FROM, RUN, COPY, EXPOSE, CMD, ENTRYPOINT, multi-stage builds, .dockerignore | 🟡 Medium | 🔥 High |
| 8 | Docker Compose | Multi-container apps, services, networks, volumes, docker-compose.yml | 🟡 Medium | 🔥 High |
| 9 | Docker Networking | Bridge, host, none, custom networks, container-to-container communication | 🟡 Medium | 🔥 High |
| 10 | Docker Volumes | Named volumes, bind mounts, data persistence, tmpfs | 🟡 Medium | 🌤️ Medium |
| 11 | Docker Best Practices | Layer caching, image size optimization, non-root user, security scanning | 🟡 Medium | 🔥 High |
| 12 | Cloud Basics (AWS) | Regions, AZs, global infra, shared responsibility model | 🟢 Easy | 🌤️ Medium |
| 13 | AWS IAM | Users, groups, roles, policies, least privilege principle | 🟡 Medium | 🔥 High |
| 14 | AWS EC2 | Instances, AMIs, instance types, key pairs, security groups, elastic IPs | 🟡 Medium | 🔥 High |
| 15 | AWS VPC | Subnets (public/private), route tables, internet gateway, NAT gateway | 🔴 Hard | 🔥 High |
| 16 | AWS S3 | Buckets, objects, permissions, static website hosting, versioning | 🟢 Easy | 🌤️ Medium |
| 17 | AWS CLI | Configure credentials, manage EC2/S3/IAM from terminal | 🟡 Medium | 🌤️ Medium |

---

### 💡 Mental Models & Analogies

| Concept | Real-World Analogy |
|---------|--------------------|
| **Bash Script** | A recipe card. Instead of cooking (typing commands) step-by-step each time, you write the recipe once and say "follow this recipe" (run the script). Cron job = "cook this every morning at 8 AM automatically" |
| **YAML** | A neatly organized grocery list with indentation showing categories. Spaces/indentation matter — like Python. `services: → web: → image: nginx` is a nested list |
| **Forward Proxy** | A middleman who buys things on your behalf so the shop doesn't know who you are (VPN/proxy for browsing) |
| **Reverse Proxy** | A receptionist at a company. You (client) ask for "sales department" — the receptionist (Nginx) routes you to the right person (backend server). You never go directly to the person |
| **Load Balancer** | A traffic policeman at a busy intersection. 1000 cars (requests) arrive → policeman sends 333 to Road A, 333 to Road B, 334 to Road C (3 backend servers). No single road gets jammed |
| **Container (Docker)** | A shipping container. Your app + all its dependencies packed in a standard box. Runs the same on your laptop, your friend's laptop, or a server in Mumbai. Unlike a VM (which ships the entire building), a container ships just the apartment |
| **Docker Image vs Container** | Image = blueprint/recipe, Container = the actual dish cooked from that recipe. One image → many containers. Image is read-only; container is a running instance |
| **Dockerfile** | Assembly instructions for IKEA furniture. Step-by-step: "start with Ubuntu (FROM), install Node (RUN), copy your code (COPY), expose port 3000 (EXPOSE), run the app (CMD)" |
| **Docker Compose** | Managing a restaurant kitchen. Instead of starting each station (database, backend, frontend) manually, you write ONE menu (docker-compose.yml) and say "start the kitchen" → all stations fire up together, connected |
| **Docker Volume** | External hard drive for your container. Container dies → data survives. Without it, deleting a container = deleting everything inside (like a RAM-only laptop) |
| **AWS Region & AZ** | Region = a city (Mumbai, Virginia). AZ = different buildings in that city. If one building catches fire (server failure), your app still runs in another building |
| **AWS IAM** | Employee access cards in an office. IAM User = employee. Role = "intern" or "manager" (determines what doors/rooms you can enter). Policy = the actual list of doors allowed. Principle of least privilege = give the intern ONLY the access they need |
| **AWS VPC** | Your own private neighborhood inside the AWS city. You decide the streets (subnets), who can enter (security groups), and which houses face the road (public subnet) vs which are behind walls (private subnet) |
| **Security Group** | Bouncers at a nightclub door. Rules: "Allow entry on port 80 (web), port 443 (HTTPS), port 22 (SSH from my IP only). Block everything else." |
| **AWS S3** | Google Drive but for machines. Unlimited storage, accessible via URLs/API. Buckets = folders. Objects = files. Can make files public (static website) or private |

---

### 🛠️ Practice

| Skill | Exercise / Problem | Platform | Difficulty |
|-------|--------------------|----------|------------|
| Bash | Write script to automate user creation (accepts username, creates home dir, sets password) | Local VM | 🟡 |
| Bash | Write script to monitor disk usage, send alert if > 80% | Local VM | 🟡 |
| Bash | Write log rotation script (archive logs older than 7 days) | Local VM | 🟡 |
| Bash | HackerRank "Bash" section — first 15 challenges | HackerRank | 🟢→🟡 |
| Python | Write script to check if a list of websites are up (using `requests`) | Local | 🟢 |
| Python | Write script to parse a JSON config file and extract values | Local | 🟢 |
| Python | Automate AWS EC2 start/stop using `boto3` | AWS Free Tier | 🟡 |
| YAML | Hand-write docker-compose and K8s-style YAML files (without copy-pasting) | Local | 🟢 |
| Nginx | Set up Nginx to serve a static HTML site on port 80 | VM | 🟢 |
| Nginx | Configure Nginx as reverse proxy to a Node.js/Flask app | VM | 🟡 |
| Nginx | Set up SSL with self-signed certificate on Nginx | VM | 🟡 |
| Docker | Containerize a simple Python Flask/Node.js "Hello World" app | Local | 🟡 |
| Docker | Write multi-stage Dockerfile (build stage + production stage) | Local | 🟡 |
| Docker | Create Docker Compose for 3-tier app: React frontend + Node API + PostgreSQL | Local | 🟡 |
| Docker | Push custom image to Docker Hub | Docker Hub | 🟢 |
| Docker | Debug a failing container using `docker logs`, `docker exec`, `docker inspect` | Local | 🟡 |
| Docker | Reduce a Docker image from 1GB to under 100MB (Alpine base, multi-stage) | Local | 🔴 |
| AWS | Launch EC2, SSH into it, install Nginx, access via public IP | AWS Free Tier | 🟡 |
| AWS | Create VPC with public + private subnet, launch EC2 in each, NAT gateway for private | AWS Free Tier | 🔴 |
| AWS | Create S3 bucket, host static website, configure bucket policy | AWS Free Tier | 🟡 |
| AWS | Create IAM users with different policies, test access limits | AWS Free Tier | 🟡 |
| Full Phase 2 | **Mini-project:** Containerize a 2-tier app → push image to DockerHub → deploy on AWS EC2 → set up Nginx reverse proxy → access via domain/IP | AWS + Docker | 🔴 |

---

### 🧪 Milestone Checkpoint

> ✅ **You're ready for Phase 3 ONLY when you can:**
> 1. Write a Bash script with loops, conditionals, and functions — **from scratch, no Google**
> 2. Explain the difference between a forward proxy, reverse proxy, and load balancer — **with your own analogy**
> 3. Write a Dockerfile for any Python/Node app, build the image, and run it — **in under 10 minutes**
> 4. Set up Docker Compose for a multi-container app (app + database) — **without copy-pasting**
> 5. Explain Docker networking (bridge, host, custom) — **draw it on paper**
> 6. Launch an EC2 instance, SSH into it, and deploy a containerized app — **end to end, from memory**
> 7. Explain what VPC, subnet, security group, and IAM are — **in simple English, to a non-tech friend**

---

### ⚠️ Common Mistakes

| # | Mistake | Why It Happens | Fix |
|---|---------|---------------|-----|
| 1 | Using `latest` tag in Dockerfiles | Seems convenient | Always pin specific versions: `python:3.11-slim`, not `python:latest`. `latest` is unpredictable |
| 2 | Running containers as root | Default behavior | Add `USER appuser` in Dockerfile. Production containers should NEVER run as root |
| 3 | Huge Docker images (1GB+) | Using `ubuntu` base + not cleaning up | Use Alpine/slim bases, multi-stage builds, combine RUN commands, remove cache |
| 4 | Storing secrets in Dockerfiles | "It's just my project" | NEVER put passwords/API keys in Dockerfile or docker-compose.yml. Use `.env` files (and .gitignore them) |
| 5 | Not using .dockerignore | Didn't know it exists | Like `.gitignore` but for Docker. Exclude `node_modules`, `.git`, `.env`, large files |
| 6 | AWS Free Tier surprise bills | Forgot to stop/terminate resources | Set billing alerts at $5. ALWAYS terminate EC2 instances and delete EBS volumes when done |
| 7 | Skipping VPC/networking on AWS | "I'll just use default VPC" | Understanding VPC is CRITICAL for interviews and real work. Learn it properly |
| 8 | Copy-pasting YAML | "YAML is just config" | Write YAML by hand until indentation is muscle memory. One wrong space = everything breaks |
| 9 | Learning AWS console only | GUI feels easier | Learn AWS CLI alongside console. In real jobs, you'll use CLI/IaC, never click through console |

---

### 🚫 What NOT To Learn Yet

| Topic | Why Skip Now | Learn In Phase |
|-------|-------------|---------------|
| Kubernetes | Need solid Docker foundation first | Phase 3–4 |
| CI/CD (Jenkins, GitHub Actions) | Need Docker + Git mastery first | Phase 3 |
| Terraform | Need Cloud basics (VPC, IAM, EC2) first | Phase 3 |
| Ansible | Need scripting + Linux admin first | Phase 3 |
| AWS advanced services (Lambda, ECS, RDS) | Master EC2/VPC/IAM/S3 first | Phase 3–4 |
| Docker Swarm | Kubernetes won. Skip Swarm entirely | Excluded |
| LXC containers | Docker dominates. LXC is niche | Excluded |
| Multiple cloud providers | Master ONE (AWS) first | Phase 5+ |
| Monitoring (Prometheus, Grafana) | Nothing to monitor yet | Phase 4 |

---

### 📖 Resources (FREE First)

| Rank | Resource | Type | Free/Paid |
|------|----------|------|-----------|
| 🥇 | **Docker Full Course — TrainWithShubham (YouTube, Hindi)** | Video | Free |
| 🥇 | **Docker Tutorial — TechWorld with Nana (YouTube, Eng)** | Video | Free |
| 🥇 | **AWS Zero to Hero — Abhishek Veeramalla (YouTube, Eng/Hindi)** | Video | Free |
| 🥇 | **Bash Scripting Tutorial — TrainWithShubham (YouTube)** | Video | Free |
| 🥈 | **Docker Official Getting Started Docs** | Docs | Free |
| 🥈 | **AWS Free Tier + AWS Skill Builder** | Hands-on + Course | Free |
| 🥈 | **Python for DevOps — freeCodeCamp (YouTube)** | Video | Free |
| 🥈 | **Nginx Crash Course — Traversy Media (YouTube)** | Video | Free |
| 🥉 | **"Docker Deep Dive" by Nigel Poulton** | Book | Paid (optional) |
| 🥉 | **KodeKloud — Docker for Beginners (free labs)** | Interactive Labs | Free tier |
| 🥉 | **AWS Certified Cloud Practitioner — freeCodeCamp (YouTube)** | Video | Free |

---

### 💼 Interview Relevance

**Companies asking Phase 2 topics:** Amazon, Google, Microsoft, Flipkart, Razorpay, Atlassian, Uber, Swiggy, CRED, Zomato, Goldman Sachs (SRE), JPMorgan (Cloud)

**Frequency:** 🔥🔥🔥 (Docker and AWS questions appear in 90%+ of DevOps interviews)

| Question | Approach Hint |
|----------|--------------|
| "What is the difference between a container and a VM?" | VM virtualizes hardware (needs full OS), container virtualizes OS (shares kernel). Containers are lighter, faster, use fewer resources. Draw the architecture diagram |
| "Explain Docker image layers. How does caching work?" | Each Dockerfile instruction = a layer. Docker caches unchanged layers. Put rarely-changing steps (install deps) BEFORE frequently-changing steps (copy code). Changing any layer invalidates cache for ALL layers below |
| "What is a multi-stage Docker build?" | Use one stage to BUILD (compile, install), another stage to RUN (copy only final binary). Result: tiny production images without build tools |
| "Explain the difference between CMD and ENTRYPOINT" | CMD = default command, can be overridden at runtime. ENTRYPOINT = fixed command, CMD becomes arguments. Use ENTRYPOINT when container IS the executable |
| "What is an AWS VPC? How would you design one?" | Virtual Private Cloud = isolated network. Design: 2 AZs, each with 1 public subnet (web servers) + 1 private subnet (databases). Internet Gateway for public, NAT Gateway for private (outbound only). Security Groups for instance-level firewall |
| "How would you secure an EC2 instance?" | Private subnet if possible, security group (minimal ports), key-pair SSH (disable password auth), IAM role (not access keys), regular updates, disable root login |
| "Write a Bash script to..." (live coding) | Practice 10+ Bash scripts. Common asks: disk usage alert, log parser, backup automation, health check script |

---

### ⏱️ Time Estimate

| Sub-topic | Hours |
|-----------|-------|
| Bash scripting | 15–20 hrs |
| Python for DevOps | 15–20 hrs |
| YAML / JSON | 3–5 hrs |
| Web servers (Nginx, proxy concepts) | 10–12 hrs |
| Docker (images → compose → networking) | 30–40 hrs |
| AWS basics (IAM, EC2, VPC, S3, CLI) | 25–35 hrs |
| Practice projects & exercises | 15–20 hrs |
| **Total** | **~115–150 hrs (7–8 weeks @ 2hr/day)** |

---

---

> ## 📣 This is Part 1. The roadmap continues with:
>
> **Part 2 will cover:**
> - Phase 3 (CI/CD, Terraform, Ansible, Kubernetes Intro)
> - Phase 4 (Kubernetes Deep Dive, Monitoring, Logging, Secrets)
> - Phase 5 (GitOps, Service Mesh, DevSecOps, Cloud Patterns)
> - Phase 6 (Portfolio, Interviews, Growth)
>
> **Part 3 will cover:**
> - Section 5: Project Ladder (15 projects)
> - Section 6: Dependency Map
> - Section 7: Glossary
> - Section 8: Excluded Topics Table
> - Section 9: Don't Skip List
> - Section 10: Learning Strategy + Stuck Protocol
> - Section 11: Tools & Setup per Phase
> - Section 12: Daily Routines (Chill / Steady / Beast)
> - Section 13: Community & Help
>
> **Type "Part 2" to continue.**
