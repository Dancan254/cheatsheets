# AWS

Reach for this for the ~20 services you actually touch and the CLI patterns around them.
Not exhaustive - the working set.

## The services that matter (by job)

| Need | Service | One-liner |
|---|---|---|
| Run a VM | **EC2** | Raw compute. You manage the OS. |
| Run containers | **ECS** / **EKS** | ECS = AWS-native orchestration; EKS = managed Kubernetes |
| Run a function | **Lambda** | Event-driven, no server, pay per invoke |
| Object storage | **S3** | Files, static sites, backups, data lake |
| Managed SQL | **RDS** / **Aurora** | Postgres/MySQL you don't patch |
| Managed NoSQL | **DynamoDB** | Key-value, single-digit-ms, serverless |
| Cache | **ElastiCache** | Managed Redis/Memcached |
| Queue | **SQS** | Managed message queue (at-least-once) |
| Pub/sub | **SNS** | Fan-out notifications/topics |
| Event bus | **EventBridge** | Route events between services |
| Networking | **VPC** | Your private network (subnets, routes, SGs) |
| DNS | **Route 53** | DNS + health-check routing |
| CDN | **CloudFront** | Edge cache in front of S3/ALB |
| Load balancer | **ELB / ALB** | ALB = L7 HTTP routing |
| Identity | **IAM** | Users, roles, policies (who can do what) |
| Secrets | **Secrets Manager** / **SSM Parameter Store** | Secrets Mgr rotates; SSM is cheaper |
| Logs/metrics | **CloudWatch** | Logs, metrics, alarms, dashboards |
| IaC | **CloudFormation** / **CDK** | AWS-native infra as code |
| Registry | **ECR** | Private Docker registry |

## IAM - the mental model

- **User** = a human/long-lived identity. **Role** = temporary identity that *anything* can assume (EC2, Lambda, another account).
- **Policy** = JSON doc granting/denying actions on resources. Attached to users/roles.
- **Prefer roles over access keys.** An EC2 instance or Lambda gets a role → no keys to leak.
- Everything is **deny by default**. An explicit `Deny` always wins over any `Allow`.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

## CLI setup

```bash
aws configure                       # or configure a named profile:
aws configure --profile prod
export AWS_PROFILE=prod             # use it for the session
aws sts get-caller-identity         # "who am I right now?" - always run when confused
aws configure list                  # which creds/region are active
```

## S3

```bash
aws s3 ls
aws s3 ls s3://my-bucket/path/
aws s3 cp file.txt s3://my-bucket/path/
aws s3 cp s3://my-bucket/path/file.txt .        # download
aws s3 sync ./dist s3://my-bucket --delete      # mirror a dir (deploy static site)
aws s3 rm s3://my-bucket/path/ --recursive
aws s3 presign s3://my-bucket/file.txt --expires-in 3600   # temp download URL
```

## EC2

```bash
aws ec2 describe-instances \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,IP:PublicIpAddress}" \
  --output table
aws ec2 start-instances --instance-ids i-0abc123
aws ec2 stop-instances  --instance-ids i-0abc123
aws ec2 describe-security-groups --group-ids sg-0abc123
```

## Logs (CloudWatch)

```bash
aws logs tail /aws/lambda/my-fn --follow          # live tail, like docker logs -f
aws logs tail /ecs/my-app --since 1h --format short
aws logs filter-log-events --log-group-name /ecs/my-app \
  --filter-pattern "ERROR" --start-time $(date -d '1 hour ago' +%s000)
```

## Lambda / ECR / SSM (common one-offs)

```bash
aws lambda invoke --function-name my-fn --payload '{"key":"val"}' out.json
aws lambda update-function-code --function-name my-fn --zip-file fileb://fn.zip

# ECR: log Docker into the registry, then push
aws ecr get-login-password --region eu-west-1 \
  | docker login --username AWS --password-stdin <acct>.dkr.ecr.eu-west-1.amazonaws.com

aws ssm get-parameter --name /prod/db/password --with-decryption --query Parameter.Value --output text
```

## Query & output (jq without jq)

```bash
--output table|json|text        # table for humans, text for scripts
--query "Reservations[].Instances[].InstanceId"    # JMESPath filter, server-side
```

## Gotchas / things I always forget

- **`aws sts get-caller-identity`** first whenever something's denied - you're probably in the wrong profile/account.
- Region matters for almost everything. A "missing" resource is usually in another region. Set `AWS_REGION` / `--region`.
- **S3 is global-namespace but region-homed.** Bucket names are globally unique; data lives in one region.
- IAM changes are eventually consistent - a new policy can take seconds to apply.
- Never commit access keys. Use roles (EC2/Lambda/ECS task roles) or SSO. Rotate anything that leaks immediately.
- Security Groups are **stateful allow-lists** (return traffic auto-allowed); NACLs are stateless. Most "can't connect" issues are a missing SG inbound rule.
- SQS is **at-least-once** → design idempotent consumers. Standard queues aren't ordered; FIFO queues are (lower throughput).
- Stopping an EC2 instance still bills for EBS storage; you only stop compute charges. Terminate to stop all charges.
- `s3 sync --delete` deletes files at the destination not present in source - great for deploys, dangerous if you point it wrong.

## Quick reference

| Task | Command |
|---|---|
| Who am I | `aws sts get-caller-identity` |
| Tail logs | `aws logs tail /path --follow` |
| Deploy static site | `aws s3 sync ./dist s3://bucket --delete` |
| Temp download link | `aws s3 presign s3://.../f --expires-in 3600` |
| ECR docker login | `aws ecr get-login-password \| docker login ...` |
| Read a secret | `aws ssm get-parameter --name N --with-decryption` |
| Switch account | `export AWS_PROFILE=name` |
