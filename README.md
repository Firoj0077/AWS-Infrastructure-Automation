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

## Setup Guide

STEP-1:	Launch an instance with key-pair, 8080 port and IAM Admin role
STEP-2:	install terraform and write a terraform code to launch an instance with same key-pair.
STEP-3: push that code into GitHub. 
STEP-4: setup Jenkins in our server. 
STEP-5: create a job in Jenkins (Automate terraform) to launch an instance. 
STEP-6: install Ansible and its dependencies(python-pip, level, openssl and pip install boto3)
STEP-7: write a plugin for getting prod server details
STEP-8: allow all permissions for Ansible configuration on prod servers
STEP-9: make ssh connection b/w the server through pem file
STEP-10: write a script to automate the configuration on prod servers.


STEP-2:	INSTALL TERRAFORM

yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
 yum install terraform -y


STEP-5:

1. Choice parameters
2. Integrate GitHub repo
3. Shell:
	#!/bin/bash
	set -xe

	cd /var/lib/jenkins/workspace /MyDeployment

	terraform init
	terraform plan
	terraform $action --auto-approve

	sleep 20

	if [ $action = "destroy" ]; then
		exit 0
	else
		cd ../Ansible
		ansible-playbook -i /opt/ansible/inventory/ /var/lib/jenkins/workspace/MyDeployment/ansible/deployment.yml
	fi

STEP-6,7,8,9:
amazon-linux-extras install ansible2 -y
yum install python python-pip python-level openssl -y
pip install boto3
vi /etc/ansible/ansible.cfg
[defaults]
inventory = /opt/ansible/inventory/aws_ec2.yml
host_key_checking = False
[plugins]
enable_plugins = aws_ec2

Mkdir -p /opt/ansible/inventory
Vim aws_ec2.yml		
========================
---
plugin: aws_ec2
regions:
  - us-east-1
filters:
  tag:aws:autoscaling:groupName: web-server-asg

##Jenkins File

pipeline {
    agent any

    stages {
        stage("infra-code") {
            steps {
                git branch: 'main', url: 'https://github.com/Firoj0077/AWS-Infrastructure-Automation.git'
            }
        }
        stage("init"){
            steps{
                sh 'echo -e "yes\n" | terraform init'
            }
        }
        stage("plan"){
            steps{
                sh 'terraform plan'
            }
        }
        stage("Action"){
            steps{
                sh 'terraform $action --auto-approve'
            }
        }
        stage("Deploy"){
            steps{
                sh 'ansible-playbook -i /opt/ansible/inventory/ /var/lib/jenkins/workspace/MyDeployment/ansible/deployment.yml'
            }
        }
    }
}

## Results

### Jenkins Stage View: consecutive successful builds
<img width="940" alt="Jenkins Stage View" src="https://github.com/user-attachments/assets/b6393186-785f-4887-adf1-2fa2a673770d" />

### Ansible deployment logs: 2 servers configured, 0 failures
<img width="940" alt="Ansible Deploy Logs" src="https://github.com/user-attachments/assets/70abbde0-9751-43f9-a3d3-925f4907d8bd" />>

PLAY RECAP
ec2-13-220-42-189.compute-1.amazonaws.com : ok=6  changed=5  unreachable=0  failed=0
ec2-3-83-255-173.compute-1.amazonaws.com  : ok=6  changed=5  unreachable=0  failed=0
```

**Outcomes**
- Full pipeline (provision + configure + deploy) completes in **~18 seconds** on average
- **Multiple consecutive green builds** in Jenkins
- **2 servers** configured in parallel with **0 failed / 0 unreachable** hosts
- New servers are picked up automatically through tags, with **no manual inventory edits**

## Author

**Firoj**
DevOps / Cloud Engineer

If you found this project useful, please give it a ⭐


