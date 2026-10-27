# CY Production AWS Infrastructure (CloudFormation)

**3-AZ VPC + private Ubuntu EC2 behind an Application Load Balancer, single NAT Gateway, SSM-managed access.**

Template: [`cy-production-vpc-alb-ec2.yaml`](cy-production-vpc-alb-ec2.yaml) · Region: `ap-south-1` · Deploy: CloudFormation console

## What it builds

| Layer | Resources |
|---|---|
| Network | VPC `10.0.0.0/16`, 3 public + 3 private subnets across 3 AZs, 1 Internet Gateway |
| Egress | **1 shared NAT Gateway** (cost-optimised), 1 Elastic IP, separate private route table per AZ |
| Subnet firewall | `CY-PUBLIC-NACL` and `CY-PRIVATE-NACL` (stateless, explicit in/out rules) |
| Load balancer | Internet-facing ALB `CY-APP-ALB` across the 3 public subnets, target group `CY-APP-TG` (port 80) |
| Compute | 1x Ubuntu 24.04 LTS, `t3a.medium`, 50 GB gp3, private subnet, no public IP, key pair `CY-PROD` |
| Access | IAM role + instance profile with `AmazonSSMManagedInstanceCore` (Session Manager, no bastion) |
| Security groups | `CY-ALB-SG` (80/443 from internet), `CY-APP-PRODUCTION-SG` (80 from ALB only) |

## Traffic flow

```
Internet -> ALB (public subnets) -> EC2 :80 (private subnet)
EC2 -> NAT Gateway (public subnet) -> Internet Gateway   (updates, SSM, SMTP, external DB)
```

## Deploy

1. CloudFormation -> Create stack -> With new resources -> Upload `cy-production-vpc-alb-ec2.yaml`
2. Stack name: `CY-PRODUCTION-ENV`
3. Parameters: defaults are fine. `KeyPairName` must already exist in the region.
   Set `CertificateArn` (ACM, same region) to enable HTTPS:443 and redirect HTTP to HTTPS. Leave empty for HTTP only.
4. Tick the IAM acknowledgement -> Submit (about 5 minutes)
5. Outputs -> `AlbDnsName` -> point a CNAME at it (e.g. `lms-v2.cyberyami.com`)

Connect to the instance: EC2 -> instance -> Connect -> Session Manager.

## Design notes / lessons learned

- **Private NACL must allow inbound TCP 1024-65535 from `0.0.0.0/0`.** NACLs are stateless; reply packets
  from internet hosts (e.g. the SSM endpoint) arrive with the *remote* IP, not the NAT's IP. Allowing only the
  VPC CIDR inbound causes SSM `i/o timeout`. This was the root cause of the original SSM failure.
- **Outbound ports must be opened on both NACLs** (public one too - the NAT Gateway lives there):
  443, 80, 587 / 465 (SMTP submission), 25060 (DigitalOcean managed PostgreSQL).
- Port 25 outbound is throttled by AWS by default - use 587/465.
- A single NAT Gateway is zonal: if its AZ fails, private subnets in all AZs lose outbound internet. Accepted trade-off for cost.
- EC2 `DependsOn` the private route, so the SSM agent never boots before its internet path exists.
- Cross-account DNS: use a plain CNAME to the ALB DNS name (Alias records only list same-account ELBs).
- Do not hand-edit resources in the console; update the stack instead, or drift will break later updates.

## Not managed by this template (done manually)

- Route 53 records and ACM certificate validation record (other AWS account)
- DigitalOcean trusted-source entry for the NAT Gateway Elastic IP
- Application install on the instance (nginx/app listening on port 80)
- Backup policy (AWS Backup plan was removed)
