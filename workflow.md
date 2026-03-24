# Workflow: remote state (S3 + DynamoDB) and VPC

This repository uses an **S3 bucket** for Terraform state and a **DynamoDB table** for state locking. The main VPC stack expects backend settings supplied at `terraform init` time.

## Optional: validate without configuring S3 yet

If you only want to check configuration syntax before the bucket exists:

```powershell
terraform init -backend=false
terraform validate
```

A real `plan` / `apply` for the root stack still requires a configured remote backend (next sections) unless you temporarily remove or change the backend block.

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) `>= 1.0`
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured, or equivalent credentials (environment variables, IAM role, SSO profile)
- IAM permissions to create S3 buckets, DynamoDB tables, and VPC networking resources in your target account and region

## Step 1 (alternative) — Create S3 and DynamoDB manually

Use this if you are **not** using the `bootstrap/` Terraform stack. The Terraform S3 backend is picky about the **DynamoDB** table schema; S3 is standard.

### DynamoDB table (state locking)

Terraform’s S3 backend expects **exactly** this primary key (names and types are fixed by Terraform):

| Setting | Required value |
|--------|----------------|
| **Partition key** | Name: `LockID` — **must match exactly** (case-sensitive) |
| **Partition key type** | **String** |
| **Sort key** | **None** (do not add a sort key) |
| **Capacity** | On-demand (**PAY_PER_REQUEST**) is typical |

No other attributes or global secondary indexes are required. Terraform stores one lock item per state file when you run `plan` / `apply`.

**AWS Console:** DynamoDB → Create table → Table name (your choice, e.g. `terraform-state-lock-recvue`) → Partition key `LockID`, type **String** → Table settings: **On-demand** → Create.

**AWS CLI** (replace table name and region):

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock-recvue \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Wait until the table status is **ACTIVE** (`aws dynamodb describe-table --table-name terraform-state-lock-recvue --query 'Table.TableStatus'`).

### S3 bucket (state storage)

1. **Name:** Globally unique across all of AWS (same region you will use in `backend.hcl`).
2. **Versioning:** **Enable** (recommended so you can recover older state objects).
3. **Encryption:** Enable default encryption (SSE-S3 is enough for many teams; SSE-KMS is also supported with extra backend options).
4. **Block Public Access:** Turn **on** for all four settings.

**AWS Console:** S3 → Create bucket → choose Region → name bucket → under “Block Public Access” keep defaults (block all) → enable versioning (Properties after create, or wizard if shown) → default encryption (Properties).

**AWS CLI** examples for `us-east-1` (replace bucket name; PowerShell):

```powershell
$BUCKET = "my-company-terraform-state-123456789012"
$REGION = "us-east-1"

aws s3api create-bucket --bucket $BUCKET --region $REGION

aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled

aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"},"BucketKeyEnabled":true}]}'
```

For regions **other than** `us-east-1`, create the bucket with a location constraint, for example:

```powershell
aws s3api create-bucket --bucket $BUCKET --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
```

### IAM (what Terraform needs)

The identity running Terraform needs, at minimum:

- **S3:** `s3:ListBucket` on the bucket ARN, and `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on `arn:aws:s3:::BUCKET_NAME/*` (and often `s3:GetObjectVersion`, `s3:PutObject` for locking/versioning behavior depending on policy).
- **DynamoDB:** `dynamodb:GetItem`, `PutItem`, `DeleteItem`, `DescribeTable` on the lock table ARN.

Use AWS managed policies only as a starting point; tighten to your bucket and table ARNs in production.

### Match `config/backend.hcl` to what you created

After manual creation, `config/backend.hcl` must line up with **your** names and region:

| Key in `backend.hcl` | Set to |
|---------------------|--------|
| `bucket` | Your S3 bucket name |
| `region` | Same AWS region as the bucket **and** the DynamoDB table |
| `dynamodb_table` | Your DynamoDB **table name** (not the partition key name) |
| `key` | Object path for this stack’s state file (e.g. `recvue/terraform.tfstate` or `recvue/prod/terraform.tfstate`) |
| `encrypt` | `true` (writes state with encryption in transit to S3; aligns with encrypted bucket) |

Example after manual setup:

```hcl
bucket         = "my-company-terraform-state-123456789012"
key            = "recvue/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-lock-recvue"
```

If your table name differs, change only `dynamodb_table` — **do not** rename the partition key away from `LockID`.

Then continue with **Step 3** (`terraform init -backend-config=config/backend.hcl`) and **Step 4** in this document.

---

## Step 1 — One-time bootstrap (state bucket + lock table)

The `bootstrap/` configuration creates the remote backend resources. It uses **local state** by default (a `terraform.tfstate` file inside `bootstrap/`). That is normal for a one-off bootstrap.

1. Choose a **globally unique** S3 bucket name (for example `mycompany-terraform-state-ACCOUNTID`).

2. From the `bootstrap` directory, initialize and apply:

   ```powershell
   cd bootstrap
   terraform init
   terraform plan -var="state_bucket_name=YOUR_UNIQUE_BUCKET_NAME" -var="aws_region=us-east-1"
   terraform apply -var="state_bucket_name=YOUR_UNIQUE_BUCKET_NAME" -var="aws_region=us-east-1"
   ```

   Optional: set `dynamodb_table_name` if the default `terraform-state-lock-recvue` is not suitable:

   ```powershell
   terraform apply `
     -var="state_bucket_name=YOUR_UNIQUE_BUCKET_NAME" `
     -var="aws_region=us-east-1" `
     -var="dynamodb_table_name=your-lock-table-name"
   ```

3. Note the outputs (`state_bucket_name`, `dynamodb_table_name`, `aws_region`). You will paste them into the root backend config in the next step.

## Step 2 — Configure the root backend

1. From the repository root, copy the example backend file:

   ```powershell
   cd ..
   copy config\backend.hcl.example config\backend.hcl
   ```

   On macOS or Linux: `cp config/backend.hcl.example config/backend.hcl`

2. Edit `config/backend.hcl` and set:

   - `bucket` — output `state_bucket_name` from bootstrap
   - `dynamodb_table` — output `dynamodb_table_name` from bootstrap
   - `region` — same region as bootstrap (must match the bucket and table)
   - `key` — path to the state object inside the bucket (change if you use multiple stacks or environments)

3. **Do not commit secrets** in `backend.hcl` if your policy treats bucket names or ARNs as sensitive. Add `config/backend.hcl` to `.gitignore` if your team keeps it local only.

## Step 3 — Initialize the main stack with the remote backend

From the **repository root** (not `bootstrap/`):

```powershell
terraform init -backend-config=config/backend.hcl
```

- **First time** (no existing remote state): Terraform creates the state object in S3 on the first successful apply.
- **Migrating from local state**: If you previously applied without this backend, Terraform will ask whether to **copy existing state** to S3. Answer **yes** if you want to keep the same resources under remote state.

To reconfigure the backend later (for example after changing the bucket name):

```powershell
terraform init -backend-config=config/backend.hcl -reconfigure
```

## Step 4 — Plan and apply the VPC module

Still from the repository root:

```powershell
terraform plan
terraform apply
```

Optional: set root variables (region, CIDRs, tags), for example:

```powershell
terraform apply `
  -var="aws_region=us-east-1" `
  -var='common_tags={"Environment"="dev","Project"="recvue"}'
```

## Step 5 — Verify

- In the AWS console (or CLI), confirm the VPC, subnets, NAT Gateway, and related resources exist in the expected region.
- Run `terraform output` to read IDs and the NAT public IP.

## Operational notes

- **Locking**: While `terraform plan` or `apply` runs, the DynamoDB table holds a lock row. If a run is interrupted, you may need `terraform force-unlock <LOCK_ID>` (use only when no other Terraform process is running).
- **State backup**: S3 versioning is enabled on the bootstrap bucket so you can recover older state objects if needed.
- **CI/CD**: Run `terraform init -backend-config=config/backend.hcl` in the pipeline; inject `backend.hcl` or use `-backend-config="key=value"` pairs as secrets/variables your platform supports.

## Summary order

| Order | Where        | Action |
|------:|--------------|--------|
| 1a | `bootstrap/` OR **manual** | Create S3 + DynamoDB (`bootstrap/` **or** console/CLI per **Step 1 (alternative)**) |
| 2 | root         | Create `config/backend.hcl` from the example; **DynamoDB partition key must stay `LockID` (String)** |
| 3 | root         | `terraform init -backend-config=config/backend.hcl` |
| 4 | root         | `terraform apply` (VPC module) |
