# builder - exam task (EC2 "builder" with Docker + Docker Compose)

Separate Terraform configuration (own state) for the exam task. It does **not** touch the lab in `../` (k3s, Jenkins, Argo CD).

## What was implemented
- Region `us-east-1`, an **existing VPC** (no VPC is created): by default the default VPC of the region, or any VPC given with `-var vpc_id=vpc-...` (the instructor's one is `vpc-044604d0bfb707142`), and its first public subnet.
- EC2 instance named **`builder`**, `t3.medium`, Ubuntu 24.04.
- SSH key: `tls_private_key` -> `local_file` `builder_key.pem` (mode 0600, git-ignored) -> `aws_key_pair` `builder-key`.
- Security group: ports **22** and **5001**, only from **my IP** (found with checkip.amazonaws.com), all outbound allowed.
- **Docker + Docker Compose installed by Terraform** with `remote-exec` (bonus). `install_docker = false` skips it (then install manually, Option 1).
- IAM role `builder-role` (read-only: EC2, VPC, AMI, ELB describe) + instance profile: the app on the builder needs **no AWS keys** (boto3 uses the instance role). IMDSv2 with hop limit 2 so the Docker container can reach it.
- Outputs: `public_ip`, `ssh_private_key_path`, `ssh_key_name`, `security_group_id`, `ssh_command`, `app_url`.

## Missing / blockers
- **`us-east-1` is blocked in my AWS account** by an organization Service Control Policy (explicit deny, e.g. `ec2:DescribeVpcs`, `ec2:DescribeImages`), so the builder runs in **`eu-north-1`** (`-var region=eu-north-1`), in the default VPC. The code is unchanged: the default region of the configuration stays `us-east-1`, as in the exam.
- The account is Free Tier only: `t3.medium` is rejected (`not eligible for Free Tier`), so I use `-var instance_type=m7i-flex.large`.
- The VPC `vpc-044604d0bfb707142` exists only in the instructor's AWS account, so by default the builder uses the default VPC of `us-east-1` in my account. With the instructor's account: `terraform apply -var vpc_id=vpc-044604d0bfb707142`.
- Jenkins is not installed here; the Jenkins + Argo CD setup is in `../` (see `../README.md`).

## Run
```bash
cd builder
aws configure                      # or export AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_DEFAULT_REGION=us-east-1
terraform init
terraform plan  -var region=eu-north-1 -var instance_type=m7i-flex.large
terraform apply -var region=eu-north-1 -var instance_type=m7i-flex.large
terraform output
```
(In the instructor's account use the defaults, or `-var vpc_id=vpc-044604d0bfb707142`.)

Connect (the key file is created next to the code):
```bash
ssh -i builder_key.pem ubuntu@<public_ip>
docker --version && docker compose version
```
Windows: if ssh says "bad permissions", run `icacls builder_key.pem /inheritance:r /grant:r "%USERNAME%:R"`.

Run the app on the builder (Section 3):
```bash
git clone https://github.com/shaharc20-bit/project1finel.git && cd project1finel/app
export AWS_REGION=eu-north-1      # the region of the builder; credentials come from the instance role
sudo -E docker compose up -d --build
```
Open `http://<public_ip>:5001/`.

Destroy when finished: `terraform destroy -var region=eu-north-1 -var instance_type=m7i-flex.large` (the same `-var` values as in `apply`).

## Validation done
`terraform validate`, `terraform fmt -check`, and an apply against a mocked AWS API (instance named `builder`, SG with 22 and 5001 from one /32, outputs present).
