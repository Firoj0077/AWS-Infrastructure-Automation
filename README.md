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
- [Setup Guide](#setup-guide)
- [Jenkinsfile](#jenkinsfile)
- [Results](#results)

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
                                       ┌──────────────────────────────┐
                                       │   Prod EC2 Instances         │
                                       │ discovered via aws_ec2 plugin│
                                       └──────────────────────────────┘
```

### Flow
1. Code is pushed to GitHub.
2. Jenkins checks out the repo and runs Terraform (`init → plan → apply/destroy`), chosen through a build parameter.
3. Terraform launches EC2 instances with the required key pair and tags, and stores its state in S3.
4. Ansible's `aws_ec2` dynamic inventory plugin discovers the new servers **by tag**, with no hard-coded IPs.
5. The Ansible playbook connects over SSH and configures the servers (installs packages, pulls the latest source code from GitHub).
6. Jenkins emails the build result.

---

## Key Features

| Feature | Description |
|---|---|
| **Infrastructure as Code** | EC2 instances defined in Terraform, so they are versioned, repeatable and reviewable |
| **Parameterized pipeline** | `Build with Parameters` lets you pick `apply` or `destroy` from a single job |
| **Dynamic inventory** | Ansible `aws_ec2` plugin finds servers by tag, so new instances are configured automatically |
| **Remote state in S3** | Terraform state file is backed up centrally, which prevents state loss and enables teamwork |
| **Agentless configuration** | Ansible configures servers over SSH using a `.pem` key, with no agents to install |
| **Safe teardown** | On `destroy`, the pipeline skips Ansible and exits cleanly |
| **Notifications** | Email alert after every build with status, job name, build number and URL |

---

## Tech Stack

| Category | Tool |
|---|---|
| Cloud | AWS (EC2, IAM, S3) |
| Infrastructure as Code | Terraform |
| CI/CD | Jenkins (Declarative Pipeline) |
| Configuration Management | Ansible + `aws_ec2` inventory plugin |
| Source Control | Git & GitHub |
| Scripting | Bash, YAML, HCL, Groovy |
| Libraries | Python 3, pip, boto3, botocore |

---

## Pipeline Stages

| # | Stage | What it does |
|---|---|---|
| 1 | **infra-code** | Clones the repository from GitHub |
| 2 | **init** | `terraform init`: downloads providers and configures the S3 backend |
| 3 | **plan** | `terraform plan`: previews infrastructure changes |
| 4 | **Action** | `terraform apply` or `terraform destroy` with `-auto-approve`, based on the build parameter |
| 5 | **Deploy** | Runs the Ansible playbook against the dynamically discovered servers |
| — | **post** | Sends an email notification with the build result |

---

## Setup Guide

### Overview
1. Launch an instance with a key pair, port 8080 open and an IAM Admin role.
2. Install Terraform and write Terraform code to launch an instance with the same key pair.
3. Push that code to GitHub.
4. Set up Jenkins on the server.
5. Create a Jenkins job (automate Terraform) to launch instances.
6. Install Ansible and its dependencies (python-pip, python-devel, openssl and boto3).
7. Configure the `aws_ec2` plugin to get the prod server details.
8. Allow all permissions for the Ansible configuration on the prod servers.
9. Make an SSH connection between the servers using the `.pem` file.
10. Write a playbook to automate the configuration of the prod servers.

### Step 2: Install Terraform
```bash
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
yum install terraform -y
```

### Step 5: Jenkins job
1. Choice parameter (`action`: `apply` / `destroy`)
2. Integrate the GitHub repo
3. Shell:

```bash
#!/bin/bash
set -xe

cd /var/lib/jenkins/workspace/MyDeployment

terraform init
terraform plan
terraform $action --auto-approve

sleep 20

if [ "$action" = "destroy" ]; then
    exit 0
else
    ansible-playbook -i /opt/ansible/inventory/ /var/lib/jenkins/workspace/MyDeployment/ansible/deployment.yml
fi
```

### Steps 6–9: Ansible setup
```bash
amazon-linux-extras install ansible2 -y
yum install python python-pip python-devel openssl -y
pip install boto3
```

Edit `/etc/ansible/ansible.cfg:
[defaults]
inventory = /opt/ansible/inventory/aws_ec2.yml
host_key_checking = False

[inventory]
enable_plugins = aws_ec2
```


mkdir -p /opt/ansible/inventory
vim /opt/ansible/inventory/aws_ec2.yml

---
plugin: aws_ec2
regions:
  - us-east-1
filters:
  tag:aws:autoscaling:groupName: web-server-asg
```

---

## Jenkinsfile

```groovy
pipeline {
    agent any

    stages {
        stage("infra-code") {
            steps {
                git branch: 'main', url: 'https://github.com/Firoj0077/AWS-Infrastructure-Automation.git'
            }
        }
        stage("init") {
            steps {
                sh 'echo -e "yes\n" | terraform init'
            }
        }
        stage("plan") {
            steps {
                sh 'terraform plan'
            }
        }
        stage("Action") {
            steps {
                sh 'terraform $action --auto-approve'
            }
        }
        stage("Deploy") {
            steps {
                sh 'ansible-playbook -i /opt/ansible/inventory/ /var/lib/jenkins/workspace/MyDeployment/ansible/deployment.yml'
            }
        }
    }
}
```

---

## Results

### Jenkins Stage View: consecutive successful builds
<img width="940" alt="Jenkins Stage View" src="https://github.com/user-attachments/assets/b6393186-785f-4887-adf1-2fa2a673770d" />

### Ansible deployment logs: 2 servers configured, 0 failures
<img width="940" alt="Ansible Deploy Logs" src="https://github.com/user-attachments/assets/70abbde0-9751-43f9-a3d3-925f4907d8bd" />

```
PLAY RECAP
ec2-13-220-42-189.compute-1.amazonaws.com : ok=6  changed=5  unreachable=0  failed=0
ec2-3-83-255-173.compute-1.amazonaws.com  : ok=6  changed=5  unreachable=0  failed=0
```

**Outcomes**
- Full pipeline (provision + configure + deploy) completes in **~18 seconds** on average
- **Multiple consecutive green builds** in Jenkins
- **2 servers** configured in parallel with **0 failed / 0 unreachable** hosts
- New servers are picked up automatically through tags, with **no manual inventory edits**

---

## Author

**Firoj**
DevOps / Cloud Engineer

If you found this project useful, please give it a ⭐


