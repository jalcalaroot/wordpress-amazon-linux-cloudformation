# WordPress on Amazon Linux 2 EC2 (CloudFormation)

CloudFormation template that launches a single Amazon Linux 2 EC2 instance running WordPress with a local MariaDB database (installed via user data), behind an HTTP Application Load Balancer with a Route 53 alias record. (The template file name `...-cloudformarion.yml` contains a typo.)

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `wordpress-amazon-linux-cloudformarion.yml`

```
Route 53 A alias (DNSNombre) --> ALB :80 --> target group (expects HTTP 302) --> EC2 (Apache + PHP 7.2 + MariaDB + WordPress)
```

- EC2 instance (Amazon Linux 2, x86_64 or arm64 AMI from SSM) with a 30 GiB encrypted gp3 root volume
- User data installs Apache, PHP 7.2 and MariaDB, creates the WordPress database and user, downloads the latest WordPress and writes `wp-config.php`
- Instance security group allowing 22, 80 and 443 from `0.0.0.0/0`
- IAM role/instance profile with `AmazonSSMFullAccess`, `CloudWatchFullAccess`, `AmazonEC2ReadOnlyAccess` and an inline `sts:*` policy
- Internet-facing ALB with an HTTP:80 listener and a target group containing the instance (health check matches `302`)
- Route 53 A alias record pointing to the ALB
- Outputs: instance ID (exported as `<Owner>-InstanceId`) and instance security group ID

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `KeyPair` | Existing EC2 key pair (declared but not attached to the instance) | `<KEY_PAIR_NAME>` |
| `VpcId` | Existing VPC | (required) |
| `LoadBalancerSubnets` | Public subnets for the ALB | (required) |
| `SubnetId` | Subnet for the instance | (required) |
| `InstanceType` | e.g. `t3a.medium` (x86_64) or `m7g.medium` (arm64) | (required) |
| `AmiID` | SSM parameter, e.g. `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` | (required) |
| `Owner` | Name prefix and `Owner` tag | (required) |
| `DBName` / `DBUsername` | WordPress database name / user | `wordpress` / `admin` |
| `DBPassword` | WordPress database password | `<DB_PASSWORD>` (template has a weak hardcoded default) |
| `DNSNombre` | Record name, e.g. `blog.<DOMAIN>` | `<SUBDOMAIN>` |
| `ACMARN` | ACM certificate ARN (declared but not used) | `<CERTIFICATE_ARN>` |
| `HostedZoneId` | Route 53 hosted zone | `<HOSTED_ZONE_ID>` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file wordpress-amazon-linux-cloudformarion.yml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    VpcId=<VPC_ID> SubnetId=<SUBNET_ID> LoadBalancerSubnets=<PUBLIC_SUBNET_1>,<PUBLIC_SUBNET_2> \
    InstanceType=t3a.medium AmiID=/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
    KeyPair=<KEY_PAIR_NAME> Owner=<OWNER> DBPassword=<DB_PASSWORD> \
    DNSNombre=blog.<DOMAIN> HostedZoneId=<HOSTED_ZONE_ID>
```

## Security notes

- `DBPassword` has a weak hardcoded default, is not `NoEcho`, and is embedded in user data (readable from instance metadata and the console). Always override it, or move it to Secrets Manager/SSM.
- SSH, HTTP and HTTPS on the instance are open to `0.0.0.0/0`; the instance can be reached directly, bypassing the ALB.
- The site is served over plain HTTP only: `ACMARN` is never used and there is no HTTPS listener.
- The instance role is broad (`AmazonSSMFullAccess`, `CloudWatchFullAccess`, `sts:*` on `*`).
- Amazon Linux 2 and PHP 7.2 are end-of-life; the database runs on the same instance with no backups.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

The root volume has `DeleteOnTermination: true`, so the WordPress content and database are deleted with the instance; back up first if needed.

## License

MIT — see [LICENSE](LICENSE).
