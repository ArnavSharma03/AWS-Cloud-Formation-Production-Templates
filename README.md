# Arnav AWS Production Infrastructure (CloudFormation)

Reusable CloudFormation template: **3-AZ VPC, single NAT Gateway, public/private NACLs, internet-facing ALB, and a private Ubuntu 24.04 EC2 managed through SSM.**

- Template: [`arnav-production-vpc-alb-ec2.yaml`](arnav-production-vpc-alb-ec2.yaml)
- Tested region: `ap-south-1` (works in any region with 3 AZs)
- Deploy method: AWS CloudFormation console (no CLI needed)

## What it builds

| Layer | Resources |
|---|---|
| Network | VPC `10.0.0.0/16`, 3 public + 3 private subnets across 3 AZs, Internet Gateway |
| Egress | 1 shared NAT Gateway + Elastic IP, one private route table per AZ |
| Subnet firewall | `<prefix>-PUBLIC-NACL` and `<prefix>-PRIVATE-NACL` with explicit in/out rules |
| Load balancer | Internet-facing ALB, target group on port 80, optional HTTPS listener |
| Compute | 1x Ubuntu 24.04 LTS, `t3a.medium`, 50 GB gp3, private subnet, no public IP |
| Access | IAM role + instance profile (`AmazonSSMManagedInstanceCore`) for Session Manager, no bastion |
| Security groups | ALB SG (80/443 from internet), app SG (80 from ALB SG only) |

All resource names start with the `NamePrefix` parameter (default `Arnav`).

## Traffic flow

```
Internet -> ALB (public subnets) -> EC2 :80 (private subnet)
EC2 -> NAT Gateway (public subnet) -> Internet Gateway   (updates, SSM, SMTP, external DB)
```

## Parameters

| Parameter | Default | Notes |
|---|---|---|
| `NamePrefix` | `Arnav` | Prefix for every resource name and tag. Change per project/environment. |
| `VpcCIDR` | `10.0.0.0/16` | Change if it overlaps another VPC you plan to peer. |
| `Public/PrivateSubnet1-3CIDR` | `10.0.1-3.0/24`, `10.0.11-13.0/24` | |
| `InstanceType` | `t3a.medium` | |
| `RootVolumeSizeGiB` | `50` | gp3 |
| `KeyPairName` | *(required)* | Existing EC2 key pair in the same region. |
| `CertificateArn` | *(empty)* | ACM cert ARN (same region). Empty = HTTP only. Set = HTTPS:443 listener + HTTP redirect. |
| `UbuntuAmiParameter` | Canonical SSM path | Always resolves to the latest Ubuntu 24.04 LTS AMI. Do not edit. |

## Deploy

1. CloudFormation -> **Create stack** -> With new resources -> **Upload a template file**
2. Pick a stack name, fill `KeyPairName` (and optionally `CertificateArn`), keep other defaults
3. Tick the IAM acknowledgement -> **Submit** (about 5 minutes)
4. Outputs -> `AlbDnsName` -> create a CNAME for your domain pointing to it
5. Connect to the instance: EC2 -> instance -> Connect -> **Session Manager**

To reuse for another project: upload the same template as a **new stack** with a different `NamePrefix` (and non-overlapping `VpcCIDR` if you will peer VPCs).

## Design notes / lessons learned

- **Private NACL must allow inbound TCP 1024-65535 from `0.0.0.0/0`.** NACLs are stateless and reply packets
  from internet hosts arrive with the remote IP, not the NAT's. Allowing only the VPC CIDR inbound causes
  SSM `i/o timeout` errors.
- **Open outbound ports on both NACLs** (the NAT Gateway sits in the public subnet): 443, 80, 587 and 465
  (SMTP submission), 25060 (managed PostgreSQL, e.g. DigitalOcean).
- AWS throttles outbound port 25 by default. Use 587 or 465.
- One NAT Gateway is zonal: if its AZ fails, private subnets in all AZs lose outbound internet. Accepted
  trade-off for cost. For full HA, add a NAT Gateway per AZ and point each private route table at its own.
- The EC2 instance `DependsOn` the private route, so the SSM agent never boots before its internet path exists.
- If the domain's DNS is in a different AWS account, use a plain CNAME to the ALB DNS name (Alias records
  only list same-account load balancers).
- Do not hand-edit stack resources in the console. Update the stack instead, otherwise drift will break later updates.
- Hand-added security group rules (e.g. SSH from a personal IP) are not tracked by the stack. Prefer Session Manager.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| SSM "Not connected", agent logs `i/o timeout` | Private NACL missing ephemeral inbound rule, or NAT route missing |
| ALB returns 504 | Nothing listening on port 80 on the instance, or target unhealthy |
| ALB returns 502 | App closes connections early / crashes on health check |
| Outbound SMTP/DB times out | Port not opened on **both** NACLs |
| Stack update fails after console edits | Drift - run "Detect drift" and reconcile |

## Not managed by this template

- DNS records and ACM certificate validation records
- Allow-list entries on external services (e.g. whitelisting the NAT Gateway Elastic IP)
- Application install on the instance (web server / app on port 80)
- Backups (add an AWS Backup plan if needed)
