# Connect Amazon RDS PostgreSQL from EC2 (Two Secure Methods)

This repository documents **two production-grade ways** to connect an **Amazon RDS PostgreSQL** database from an **EC2 (Ubuntu)** instance:

1. **Using AWS Secrets Manager (username/password)**
2. **Using IAM Database Authentication (passwordless, token-based)**

Both methods assume:
- EC2 and RDS are in the **same VPC**
- RDS is **not publicly accessible**
- EC2 uses an **IAM Role** (no access keys)

---

## Part 1: Connect RDS from EC2 using AWS Secrets Manager

### Architecture
EC2 (IAM Role) → Secrets Manager → RDS PostgreSQL

### Step 1: Attach IAM Policy to EC2 Role

Attach the following policy to the **EC2 IAM Role**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:SECRET_NAME*"
    }
  ]
}
```

### Step 2: Security Group Configuration

- RDS Security Group:
  - Inbound: PostgreSQL (5432)
  - Source: **EC2 Security Group**
- EC2 outbound: allow all (default)

---

### Step 3: Install Required Packages on EC2

```bash
sudo apt update
sudo apt install -y awscli jq postgresql-client
```

Set region once:

```bash
aws configure set region ap-south-1
```

---

### Step 4: Fetch Secret from Secrets Manager

```bash
SECRET=$(aws secretsmanager get-secret-value   --secret-id "rds!db-xxxxxxxxxxxxxxxx"   --query SecretString   --output text)
```

---

### Step 5: Extract Credentials

```bash
DB_USER=$(echo "$SECRET" | jq -r .username)
DB_PASS=$(echo "$SECRET" | jq -r .password)
```

---

### Step 6: Export RDS Connection Details

```bash
export DB_HOST="your-rds-endpoint.ap-south-1.rds.amazonaws.com"
export DB_PORT=5432
export DB_NAME=postgres
```

---

### Step 7: Connect to PostgreSQL

```bash
PGPASSWORD="$DB_PASS" psql   -h "$DB_HOST"   -U "$DB_USER"   -d "$DB_NAME"   -p "$DB_PORT"
```

Successful output:

```text
postgres=>
```

---

## Part 2: Connect RDS using IAM Database Authentication (Passwordless)

This method **eliminates static DB passwords**.

### Architecture
EC2 IAM Role → IAM Auth Token → RDS PostgreSQL

---

### Step 1: Enable IAM DB Authentication on RDS

- RDS → Databases → Modify
- Enable **IAM DB authentication**
- Apply immediately

---

### Step 2: Create IAM-enabled DB User

Login using admin user and run:

```sql
CREATE USER iam_user WITH LOGIN;
GRANT CONNECT ON DATABASE postgres TO iam_user;
GRANT USAGE ON SCHEMA public TO iam_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO iam_user;
GRANT rds_iam TO iam_user;
```

---

### Step 3: Attach IAM Policy to EC2 Role

Get **DB Resource ID** from RDS → Configuration.

Attach this policy to EC2 IAM Role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:ap-south-1:ACCOUNT_ID:dbuser:DB_RESOURCE_ID/iam_user"
    }
  ]
}
```

---

### Step 4: Download RDS SSL Certificate

```bash
wget https://truststore.pki.rds.amazonaws.com/ap-south-1/ap-south-1-bundle.pem
```

---

### Step 5: Generate IAM Auth Token

```bash
export DB_HOST="your-rds-endpoint.ap-south-1.rds.amazonaws.com"
export DB_PORT=5432
export DB_USER="iam_user"
export DB_NAME="postgres"

TOKEN=$(aws rds generate-db-auth-token   --region ap-south-1   --hostname "$DB_HOST"   --port 5432   --username "$DB_USER")
```

---

### Step 6: Connect to RDS Using IAM Token

```bash
psql   "host=$DB_HOST   port=5432   dbname=$DB_NAME   user=$DB_USER   sslmode=verify-full   sslrootcert=ap-south-1-bundle.pem   password=$TOKEN"
```

Successful output:

```text
postgres=>
```

---

## Notes & Best Practices

- IAM tokens are valid for **15 minutes**
- Always use **SSL**
- Prefer IAM DB Auth for applications
- Keep password-based access only for break-glass scenarios

---

## Summary

| Method | Secrets Stored | Rotation | Security |
|------|----------------|----------|----------|
| Secrets Manager | Yes | Automatic | High |
| IAM DB Auth | No | Not needed | Very High |

---

This setup follows AWS security best practices and is suitable for production workloads.
