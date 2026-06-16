


# AWS Networking — Terraform

Infrastructure as Code for AWS networking using Terraform. Builds a production-style VPC with public and private subnets, NAT Gateway, Internet Gateway, route tables, and security groups.

## Architecture

<img width="1292" height="874" alt="image" src="https://github.com/user-attachments/assets/ee40400b-b11c-4583-98fc-afc49fae4668" />

## What gets created

| Resource | Name | Purpose |
|---|---|---|
| VPC | `mlops-dev-vpc` | Network boundary, CIDR 10.0.0.0/16 |
| Public subnet | `mlops-dev-subnet-public-1a` | Internet-facing resources, NAT Gateway |
| Private subnet | `mlops-dev-subnet-private-1b` | Workloads, no direct internet access |
| Internet Gateway | `mlops-dev-igw` | Public ingress and egress |
| Elastic IP | `mlops-dev-nat-eip` | Static IP for NAT Gateway |
| NAT Gateway | `mlops-dev-nat-gateway` | Outbound-only internet for private subnet |
| Route table | `mlops-dev-rt-public` | Routes public subnet traffic to IGW |
| Route table | `mlops-dev-rt-private` | Routes private subnet traffic to NAT |
| Security group | `mlops-dev-sg-private` | Allows 443 and 22 inbound from VPC only |

## Project structure

```
aws-networking-terraform/
├── .gitignore
├── README.md
└── aws/
    ├── main.tf               ← root module, calls networking module
    ├── provider.tf           ← AWS provider config
    ├── variable.tf           ← root input variables
    ├── outputs.tf            ← root outputs
    └── modules/
        └── networking/
            ├── main.tf       ← VPC, subnets, IGW, NAT, SG resources
            ├── variables.tf  ← networking module inputs
            └── outputs.tf    ← exposes vpc_id, subnet_ids, sg_id
```

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.5
- [AWS CLI](https://aws.amazon.com/cli/) configured with credentials
- An AWS account

```bash
# verify terraform is installed
terraform -version

# verify AWS CLI is configured
aws sts get-caller-identity
```

## Usage

### 1. Clone the repo

```bash
git clone https://github.com/<your-username>/aws-networking-terraform.git
cd aws-networking-terraform/aws
```

### 2. Review variables

Default values are set in `variable.tf`. Override them by creating a `dev.tfvars` file:

```hcl
# dev.tfvars  (gitignored — never commit this)
aws_region   = "eu-west-2"
project_name = "mlops"
env          = "dev"
vpc_cidr     = "10.0.0.0/16"
```

### 3. Initialise

```bash
terraform init
```

Downloads the AWS provider. Only needed once, or after changing provider versions.

### 4. Plan

```bash
terraform plan
# or with a tfvars file
terraform plan -var-file="dev.tfvars"
```

Review the output. You should see ~13 resources to add, 0 to change, 0 to destroy.

### 5. Apply

```bash
terraform apply
# or with a tfvars file
terraform apply -var-file="dev.tfvars"
```

Type `yes` when prompted.

### 6. Verify

After apply, check the outputs:

```bash
terraform output
```

Verify in AWS Console:
- **VPC** → Your VPCs → find `mlops-dev-vpc`
- **Subnets** → confirm two subnets in different AZs
- **Route Tables** → public RT routes to IGW, private RT routes to NAT
- **NAT Gateways** → status should be `Available`

### 7. Destroy

Always destroy when done to avoid unnecessary charges.

```bash
terraform destroy
```

## Inputs

| Variable | Description | Default |
|---|---|---|
| `aws_region` | AWS region to deploy into | `eu-west-2` |
| `project_name` | Prefix applied to all resource names | `mlops` |
| `env` | Environment name (dev/staging/prod) | `dev` |
| `vpc_cidr` | CIDR block for the VPC | `10.0.0.0/16` |

## Outputs

| Output | Description |
|---|---|
| `vpc_id` | VPC ID |
| `public_subnet_id` | Public subnet ID |
| `private_subnet_id` | Private subnet ID |
| `private_sg_id` | Security group ID for private workloads |
| `private_route_table_id` | Private route table ID |

## Key concepts

**Why two subnets?**
Public subnet resources can receive inbound internet connections. Private subnet resources can only make outbound connections (via NAT) — nothing from the internet can reach them directly.

**Why is the NAT Gateway in the public subnet?**
The NAT Gateway needs a public IP (Elastic IP) to communicate with the internet on behalf of private resources. It must therefore sit in the public subnet which has a route to the Internet Gateway.

**What is route propagation?**
Each subnet's route table controls where traffic goes. Without the correct routes — `0.0.0.0/0 → IGW` on the public side and `0.0.0.0/0 → NAT` on the private side — traffic has nowhere to go and packets drop silently.

**Why no defaults in the networking module variables?**
The root module (`aws/variable.tf`) is the single source of truth for all values. The child module (`modules/networking/variables.tf`) declares what it accepts but sets no defaults — this prevents values getting out of sync across modules.

## Cost awareness

| Resource | Approx cost |
|---|---|
| NAT Gateway | ~$0.045/hr + $0.045/GB data |
| Elastic IP (unattached) | ~$0.005/hr |
| VPC, subnets, IGW, route tables, SG | Free |

Estimated: ~$1.10/day if left running. Run `terraform destroy` when not in use.

## Terraform commands reference

```bash
terraform init        # download providers
terraform validate    # check syntax
terraform fmt         # format code
terraform plan        # preview changes
terraform apply       # deploy
terraform output      # show outputs
terraform destroy     # tear down all resources
```

## Contributing

1. Create a feature branch: `git checkout -b feat/your-change`
2. Make changes and test with `terraform plan`
3. Commit using conventional commits: `git commit -m "feat: add second private subnet"`
4. Push and open a PR: `git push origin feat/your-change`

## Managed by

Terraform — do not modify resources manually in the AWS Console. Manual changes will be overwritten on the next `terraform apply`.
