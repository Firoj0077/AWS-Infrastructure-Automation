# AWS-Infrastructure-Automation
### Terraform · Jenkins · Ansible · GitHub · Amazon S3
A one-click, fully automated pipeline that provisions AWS EC2 servers with Terraform, discovers them automatically with Ansible's AWS dynamic inventory, and configures them for deployment, all orchestrated by a parameterized Jenkins pipeline. Terraform state is stored remotely in Amazon S3, and every build sends an email notification with its status.

**Result:** Infrastructure that used to take manual console work and SSH sessions is now created, configured and deployed in **~18 seconds per pipeline run**, with zero manual login to the servers.


---
## Table of Contents
- [Architecture](#architecture)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Pipeline Stages](#pipeline-stages)
- [Repository Structure](#repository-structure)
- [Setup Guide](#setup-guide)
- [Jenkinsfile](#jenkinsfile)
- [Results](#results)
- [Challenges & Learnings](#challenges--learnings)
- [Future Improvements](#future-improvements)

---
## Architecture

```
   ┌──────────────┐   git push   ┌──────────────┐
   │   Developer  │ ───────────▶ │    GitHub    │
   └──────────────┘              └──────┬───────┘
                                        │ checkout
                                        ▼
 ┌───────────────────────────────────────────────────────────────┐
 │                 Jenkins Server (EC2 + IAM Role)               │
 │                                                               │
 │  ┌────────────┐  ┌──────┐  ┌──────┐  ┌──────────────┐  ┌────────┐
 │  │ infra-code │─▶│ init │─▶│ plan │─▶│ Action       │─▶│ Deploy │
 │  │ (checkout) │  │      │  │      │  │ apply/destroy│  │Ansible │
 │  └────────────┘  └──────┘  └──────┘  └──────┬───────┘  └───┬────┘
 └──────────────────────────────────────────────┼──────────────┼─┘
                                                │              │ SSH (.pem)
                         ┌──────────────┐       │              │
                         │  Amazon S3   │◀──────┤ tfstate      │
                         │ (state file) │       │              │
                         └──────────────┘       ▼              ▼
                                       ┌─────────────────────────────┐
                                       │   Prod EC2 Instances        │
                                       │ (tag: Environment=dev)      │
                                       │ discovered via aws_ec2 plugin│
                                       └─────────────────────────────┘
##Flow
1. Code is pushed to GitHub.
2. Jenkins checks out the repo and runs Terraform (`init → plan → apply/destroy`), chosen through a build parameter.
3. Terraform launches EC2 instances with the required key pair and tags, and stores its state in S3.
4. Ansible's `aws_ec2` dynamic inventory plugin discovers the new servers **by tag**, with no hard-coded IPs.
5. The Ansible playbook connects over SSH and configures the servers (installs packages, pulls the latest source code from GitHub).
6. Jenkins emails the build result.

## Key Features
| Feature | Description |

| **Infrastructure as Code** | EC2 instances defined in Terraform, so they are versioned, repeatable and reviewable |
| **Parameterized pipeline** | `Build with Parameters` lets you pick `apply` or `destroy` from a single job |
| **Dynamic inventory** | Ansible `aws_ec2` plugin finds servers by tag, so new instances are configured automatically |
| **Remote state in S3** | Terraform state file is backed up centrally, which prevents state loss and enables teamwork |
| **Agentless configuration** | Ansible configures servers over SSH using a `.pem` key, with no agents to install |
| **Safe teardown** | On `destroy`, the pipeline skips Ansible and exits cleanly |
| **Notifications** | Email alert after every build with status, job name, build number and URL |

## Tech Stack

| Category | Tool |

| Cloud | AWS (EC2, IAM, S3) |
| Infrastructure as Code | Terraform |
| CI/CD | Jenkins (Declarative Pipeline) |
| Configuration Management | Ansible + `aws_ec2` inventory plugin |
| Source Control | Git & GitHub |
| Scripting | Bash, YAML, HCL, Groovy |
| Libraries | Python 3, pip, boto3, botocore |

## Pipeline Stages

| # | Stage | What it does |

| 1 | **infra-code** | Clones the repository from GitHub |
| 2 | **init** | `terraform init`: downloads providers and configures the S3 backend |
| 3 | **plan** | `terraform plan`: previews infrastructure changes |
| 4 | **Action** | `terraform apply` or `terraform destroy` with `-auto-approve`, based on the build parameter |
| 5 | **Deploy** | Runs the Ansible playbook against the dynamically discovered servers |
| — | **post** | Sends an email notification with the build result |




