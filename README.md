# project1-terraform - AWS infrastructure for the DevOps final project

Two separate Terraform configurations (each with its own state):

| Folder | What | Region |
|---|---|---|
| `/` (this folder) | **The lab**: one `terraform apply` builds a VPC, an Ubuntu EC2 server, Docker, k3s, Helm, Jenkins and Argo CD | `eu-north-1` |
| `builder/` | **Exam task**: EC2 named "builder" with Docker + Docker Compose, in the existing VPC of the instructor | `us-east-1` |

The other repositories: `project1finel` (code, Jenkinsfile, Helm chart) and `project1-gitops` (what Argo CD deploys).

## What was implemented (the lab)
| Part | Resources |
|---|---|
| `bootstrap/` | S3 bucket for the state (versioning, encryption), DynamoDB table for locking |
| `modules/network` | VPC `10.20.0.0/16`, public subnet, internet gateway, route table |
| `modules/server` | Security group (22, 5001, 8080, 30443, 30501, 30502 from my IP only), key pair, IAM role (read-only EC2/ELB), Ubuntu 24.04 EC2 with IMDSv2, Elastic IP |
| `scripts/install.sh` | user_data: Docker, git, k3s, Helm, Argo CD |
| `provision.tf` | `remote-exec`: Jenkins in Docker (configured as code), Argo CD Applications `flask-app` and `applicationsets` (from `project1-gitops`) |
| `jenkins/` | Dockerfile (docker CLI + helm), `casc.yaml` (credentials and the pipeline job that builds `project1finel`) |
| `argocd/` | Templates of the Argo CD Applications |

## Missing / blockers
- None known. Tokens are passed as variables (`terraform.tfvars`, git-ignored), never stored in git.
- Before `terraform apply`, `project1-gitops` and `project1finel` must have their content on `main` (Argo CD and Jenkins read it from there).

## Run the lab
```
ssh-keygen -t ed25519 -f ~/.ssh/aws-lab         # once
cd bootstrap && terraform init && terraform apply && cd ..      # once: remote state (skip if it already exists)
cp terraform.tfvars.example terraform.tfvars    # fill in the tokens and the password
terraform init
terraform plan
terraform apply
terraform output
```
Outputs: `ssh_command`, `app_url`, `jenkins_url`, `argocd_url`. Jenkins and Argo CD: user `admin`, password from `terraform.tfvars`.

Remove everything: `terraform destroy` in this folder (not in `bootstrap`).

## The exam task
See `builder/README.md`.
