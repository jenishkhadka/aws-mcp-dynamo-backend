# Serverless DynamoDB MCP Backend — Full Setup Guide
**Use this to reproduce the project from scratch**

---

## Prerequisites

- AWS Free Tier account
- Ubuntu Linux (VM or native)
- Windows PC (optional, for Kiro IDE)

---

## Tools Required

```bash
# Check versions
aws --version        # aws-cli 2.31.35
terraform -v         # 1.9.0
jq --version         # 1.8.1
envsubst --version   # 0.23.2
git --version        # latest
curl --version       # any
```

### Install missing tools on Ubuntu

```bash
# curl
sudo apt-get install curl -y

# jq
sudo apt install jq -y

# gettext (for envsubst)
sudo apt install gettext -y

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y
```

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/jenish/aws-mcp-dynamo-backend.git
cd aws-mcp-dynamo-backend
```

---

## Step 2: Make Scripts Executable

```bash
chmod +x validate.sh check_env.sh apply.sh destroy.sh
```

---

## Step 3: Configure AWS Credentials

### Get Access Keys from AWS Console
1. Go to console.aws.amazon.com
2. Top right → your account name → Security credentials
3. Access keys → Create access key → CLI → Create
4. Copy Access Key ID and Secret Access Key

### Run aws configure

```bash
aws configure
```

Fill in:
```
AWS Access Key ID: YOUR_ACCESS_KEY_ID
AWS Secret Access Key: YOUR_SECRET_ACCESS_KEY
Default region name: us-east-1
Default output format: json
```

### Verify credentials

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "YOUR_ACCOUNT_ID",
    "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/YOUR_USERNAME"
}
```

---

## Step 4: Check Environment

```bash
./check_env.sh
```

Expected output:
```
===================================================================
  DynamoDB MCP — Environment Check
===================================================================
Checking for required tools...
  ✓ aws
  ✓ terraform
  ✓ jq
  ✓ envsubst
Checking AWS credentials...
  ✓ AWS Account: YOUR_ACCOUNT_ID
  ✓ Identity: arn:aws:iam::YOUR_ACCOUNT_ID:user/YOUR_USERNAME
===================================================================
  Environment check passed!
===================================================================
```

---

## Step 5: Deploy Everything

```bash
./apply.sh
```

Type `yes` when prompted. Takes 2-5 minutes.

This creates:
- DynamoDB table
- 11 Lambda functions
- API Gateway
- IAM user and roles
- Secrets Manager secret

---

## Step 6: Get Your Credentials

### Find your API endpoint

```bash
aws apigatewayv2 get-apis --region us-east-1
```

Note the `ApiEndpoint` value.

### Get your MCP secret key

```bash
aws secretsmanager get-secret-value --secret-id dynamodb-mcp-proxy --region us-east-1
```

Note all values from `SecretString`:
- `access_key_id`
- `secret_access_key`
- `api_endpoint`
- `region`

---

## Step 7: Create MCP Config File

```bash
cat > ~/aws-mcp-dynamo-backend/02-proxy/mcp_config.json << 'EOF'
{
  "mcpServers": {
    "dynamodb": {
      "command": "bash",
      "args": ["/home/YOUR_USERNAME/aws-mcp-dynamo-backend/02-proxy/proxy.sh"],
      "env": {
        "MCP_ACCESS_KEY_ID": "YOUR_ACCESS_KEY_ID",
        "MCP_SECRET_ACCESS_KEY": "YOUR_SECRET_ACCESS_KEY",
        "MCP_API_ENDPOINT": "https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/",
        "MCP_REGION": "us-east-1"
      }
    }
  }
}
EOF
```

---

## Step 8: Test Proxy Manually

```bash
MCP_ACCESS_KEY_ID="YOUR_ACCESS_KEY_ID" \
MCP_SECRET_ACCESS_KEY="YOUR_SECRET_ACCESS_KEY" \
MCP_API_ENDPOINT="https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/" \
MCP_REGION="us-east-1" \
bash 02-proxy/proxy.sh
```

Expected output:
```
NOTE: DynamoDB MCP proxy started.
NOTE: Endpoint: https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/  Region: us-east-1
NOTE: Discovering tools from https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/tools ...
NOTE: Discovered 10 tool(s).
```

---

## Step 9: Install Kiro IDE on Ubuntu

```bash
# Download
wget https://desktop-release.kiro.dev/latest/linux/kiro.deb -O kiro.deb

# Install
sudo dpkg -i kiro.deb
sudo apt-get install -f

# Launch
kiro
```

---

## Step 10: Configure MCP in Kiro

1. Open Kiro IDE
2. Click MCP in sidebar → Enable
3. Click Edit Config (mcp.json)
4. Paste your mcp_config.json contents
5. Save → Restart Kiro
6. Check bottom left → `dynamodb Connected (10 tools)` ✅

---

## Step 11: Test with AI

In Kiro Vibe chat type:

```
List all DynamoDB tables
```

```
Add an item to the Users table with userId "user1" and name "Your Name"
```

```
Get the item from Users table where userId is "user1"
```

```
Scan all items in the Users table
```

---

## Useful Commands

### Check what's deployed on AWS

```bash
# List Lambda functions
aws lambda list-functions --region us-east-1

# List DynamoDB tables
aws dynamodb list-tables --region us-east-1

# List API Gateways
aws apigatewayv2 get-apis --region us-east-1

# List IAM users
aws iam list-users
```

### Validate deployment

```bash
./validate.sh
```

### Destroy everything (cleanup)

```bash
./destroy.sh
```

⚠️ This deletes ALL AWS resources created by Terraform.

---

## Project File Structure

```
aws-mcp-dynamo-backend/
├── apply.sh              # Deploy everything
├── destroy.sh            # Delete everything
├── check_env.sh          # Verify tools and credentials
├── validate.sh           # Test Lambda functions
├── 01-lambdas/
│   ├── code/
│   │   └── dynamodb_ops.py    # All Lambda handlers
│   ├── main.tf                # AWS provider config
│   ├── api.tf                 # API Gateway
│   ├── iam-proxy-user.tf      # IAM user and roles
│   ├── sample-table.tf        # DynamoDB table
│   ├── lambda-get-item.tf
│   ├── lambda-put-item.tf
│   ├── lambda-update-item.tf
│   ├── lambda-delete-item.tf
│   ├── lambda-query.tf
│   ├── lambda-scan.tf
│   ├── lambda-batch-get.tf
│   ├── lambda-list-tables.tf
│   ├── lambda-describe-table.tf
│   ├── lambda-count-items.tf
│   └── lambda-tools.tf
└── 02-proxy/
    ├── proxy.sh               # MCP stdio proxy (SigV4 signing)
    └── mcp_config.json        # MCP server config for Kiro/Claude
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `Permission denied` on .sh files | `chmod +x *.sh` |
| `curl: command not found` | `sudo apt-get install curl -y` |
| `MCP_ACCESS_KEY_ID is required` | Pass env vars or check mcp.json config |
| `Output not found` in terraform | Run `./apply.sh` again |
| `Items: []` in API Gateway | Use `apigatewayv2` not `apigateway` |
| Kiro MCP connection closed | Check proxy.sh runs manually first |

---

## VMware Ubuntu VM Recommended Settings

For 16GB RAM Windows PC with AMD Ryzen 6900H:

| Setting | Value |
|---------|-------|
| Memory | 8192 MB (8GB) |
| Processors | 2 |
| Cores per processor | 4 |
| Total cores | 8 |