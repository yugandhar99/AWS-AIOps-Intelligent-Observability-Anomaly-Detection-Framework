# AWS Intelligent Observability & Anomaly Detection Framework

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Terraform](https://img.shields.io/badge/terraform->=1.0-blue.svg)
![AWS](https://img.shields.io/badge/AWS-Production%20Ready-orange.svg)
![Python](https://img.shields.io/badge/python-3.11-blue.svg)

> An enterprise-grade, AI-powered observability and automated anomaly detection framework for AWS using CloudWatch, Lambda-based statistical anomaly scoring, Amazon Bedrock for generative log reasoning, and multi-channel incident alerting.

---

## 📋 Table of Contents

- [Project Description](#-project-description)
- [Architecture](#-architecture)
- [Project Requirements](#-project-requirements)
- [Database & State Storage Setup](#-database--state-storage-setup)
- [How to Run & Test the Project Locally](#-how-to-run--test-the-project-locally)
- [Cloud Deployment Guide](#-cloud-deployment-guide)
- [Configuration Reference](#-configuration-reference)
- [Dashboards & Visualization](#-dashboards--visualization)
- [License](#-license)

---

## 🎯 Project Description

The **Intelligent Observability & Anomaly Detection Framework** is a fully automated, Infrastructure-as-Code (Terraform) solution designed to enhance Cloud, DevOps, and Site Reliability Engineering (SRE) workflows. 

Traditional monitoring depends on static, hardcoded thresholds that trigger alert fatigue. This framework augments AWS observability by running scheduled Lambda analytical workers that perform statistical time-series anomaly detection on CloudWatch metrics and analyze logs using pattern clustering and Amazon Bedrock foundation models (Anthropic Claude).

### Key Capabilities

* **Statistical Anomaly Scoring**: Evaluates metrics dynamically using rolling Z-score baselines, percentile distribution, and moving averages (via NumPy/SciPy).
* **Intelligent Log Analysis**: Ingests CloudWatch log streams every 5 minutes to detect recurring error patterns, track burst frequencies, and calculate log anomaly scores.
* **Generative AI Root Cause Analysis**: Leverages Amazon Bedrock to extract root causes, evaluate blast radius/impact, and generate remediation suggestions directly within alert payloads.
* **AWS Cost Anomaly Detection**: Integrates with AWS Cost Explorer to track spending irregularities and publish budget breach alerts.
* **Multi-Channel Alert Dispatch**: Formats and dispatches enriched alerts to Slack, PagerDuty, and SNS email subscribers.
* **Production-Ready Dashboards**: Ships with curated CloudWatch dashboards and pre-configured Grafana JSON dashboards.

---

## 🏗️ Architecture

```
┌──────────────────┐
│ CloudWatch Logs  │
│ & AWS Metrics    │
└────────┬─────────┘
         │
         ├── Scheduled EventBridge Triggers (Every 5 mins)
         │
         ▼
┌────────────────────────────────────────────────────────┐
│                   Serverless Analytics                 │
│                                                        │
│  ┌───────────────────────┐   ┌──────────────────────┐  │
│  │   Log Analyzer        │   │  Anomaly Scorer      │  │
│  │   Lambda (Python)     │   │  Lambda (Python)     │  │
│  └──────────┬────────────┘   └──────────┬───────────┘  │
└─────────────┼───────────────────────────┼──────────────┘
              │                           │
              │ (Optional GenAI Analysis) │
              ▼                           │
       ┌───────────────┐                  │
       │ Amazon Bedrock│                  │
       │ (Claude Model)│                  │
       └──────┬────────┘                  │
              │                           │
              └─────────────┬─────────────┘
                            ▼
                   ┌─────────────────┐
                   │  SNS Topic      │
                   └────────┬────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
       ┌─────────────┐             ┌─────────────┐
       │    Slack    │             │  PagerDuty  │
       │ Integration │             │ Integration │
       └─────────────┘             └─────────────┘
```

---

## 📦 Project Requirements

### 1. Local Workstation Tools
* **Terraform** >= 1.0 (tested with Terraform 1.5+)
* **AWS CLI** v2 (configured with valid AWS credentials)
* **Python** 3.11+ (for local Lambda execution and testing)
* **Git** 2.x+
* *(Optional)* **Docker** (if testing Grafana dashboards locally)

### 2. AWS Permissions
The deploying IAM identity requires permissions for:
* **CloudWatch**: `cloudwatch:*`, `logs:*`
* **AWS Lambda**: `lambda:CreateFunction`, `lambda:UpdateFunctionCode`, `lambda:InvokeFunction`
* **Amazon SNS**: `sns:CreateTopic`, `sns:Subscribe`, `sns:Publish`
* **IAM**: `iam:CreateRole`, `iam:PutRolePolicy`, `iam:PassRole`
* **EventBridge**: `events:PutRule`, `events:PutTargets`
* **Cost Explorer**: `ce:CreateAnomalyMonitor`, `ce:CreateAnomalySubscription`
* *(Optional)* **Amazon Bedrock**: `bedrock:InvokeModel`

---

## 🗄️ Database & State Storage Setup

This framework leverages serverless cloud-native storage patterns rather than a traditional relational database:

### 1. Terraform Remote State & Distributed Locking (S3 + DynamoDB)
For collaborative and production deployments, store your Terraform state remotely in an S3 bucket with DynamoDB state locking:

```bash
# 1. Create S3 Bucket for Terraform State
aws s3api create-bucket \
  --bucket intelligent-observability-tfstate \
  --region us-east-1

# 2. Enable Bucket Versioning
aws s3api put-bucket-versioning \
  --bucket intelligent-observability-tfstate \
  --versioning-configuration Status=Enabled

# 3. Create DynamoDB Table for Distributed State Locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Configure your `versions.tf` backend:
```hcl
terraform {
  backend "s3" {
    bucket         = "intelligent-observability-tfstate"
    key            = "observability/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
  }
}
```

### 2. Time-Series Telemetry & Metric Storage
* **CloudWatch Log Groups**: Acts as the immutable, high-throughput log store with customizable retention (e.g., 14 to 90 days).
* **CloudWatch Custom Metrics**: The Lambda workers publish computed anomaly metrics under the custom namespaces `AIOps/LogAnalysis` and `AIOps/AnomalyScores`.

---

## 💻 How to Run & Test the Project Locally

### Step 1: Validate Terraform Syntax & Configuration
Verify Terraform configuration files without creating AWS resources:

```bash
# Initialize Terraform provider plugins
terraform init

# Validate syntax and module structure
terraform validate

# Format HCL files
terraform fmt -check
```

### Step 2: Test Lambda Handlers Locally with Python

Both Lambda functions (`lambdas/log-analyzer` and `lambdas/anomaly-scorer`) can be executed and unit-tested locally using Python:

```bash
# Create and activate a Python virtual environment
python -m venv venv

# Windows PowerShell:
.\venv\Scripts\Activate.ps1
# Linux/macOS:
# source venv/bin/activate

# Install required numerical analysis and AWS SDK dependencies
pip install boto3 numpy scipy
```

Test invoking the anomaly scoring logic locally:
```bash
# Set dummy environment variables
$env:LOG_LEVEL="DEBUG"
$env:ANOMALY_THRESHOLD="0.7"

# Run a quick interactive test
python -c "
from lambdas.anomaly_scorer.handler import AnomalyScorer
scorer = AnomalyScorer()
print('AnomalyScorer initialized successfully!')
"
```

### Step 3: Run Grafana Locally (Optional)
To test and visualize the included dashboards before deploying to Amazon Managed Grafana:

```bash
# Run Grafana container locally
docker run -d -p 3000:3000 --name grafana-local grafana/grafana

# Open http://localhost:3000 in your browser (Default: admin / admin)
# Navigate to Dashboards -> Import -> Upload files from:
# - dashboards/grafana/infrastructure-overview.json
# - dashboards/grafana/cost-analysis.json
```

---

## 🚀 Cloud Deployment Guide

### 1. Prepare Configuration
Navigate to the root or complete example directory:

```bash
cd examples/complete
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
region      = "us-east-1"
environment = "prod"

# Monitoring targets
log_groups = [
  "/aws/lambda/payment-service",
  "/aws/ecs/order-api"
]

# Notifications
slack_webhook_url         = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
pagerduty_integration_key = "YOUR_PAGERDUTY_KEY"
sns_email_addresses       = ["sre-alerts@company.com"]

# Optional GenAI Insights
enable_bedrock   = true
bedrock_model_id = "anthropic.claude-3-sonnet-20240229-v1:0"
```

### 2. Plan and Deploy
```bash
# Preview infrastructure changes
terraform plan -out=tfplan

# Apply infrastructure changes
terraform apply tfplan
```

### 3. Verify Deployed Resources
Once deployed, the terminal outputs will supply:
* SNS Alert Topic ARN
* Main CloudWatch Dashboard URL
* Cost Anomaly Dashboard URL

---

## ⚙️ Configuration Reference

| Variable | Description | Type | Default |
| :--- | :--- | :---: | :---: |
| `project_name` | Name prefix for all provisioned resources | `string` | `"aiops-monitoring"` |
| `environment` | Deployment environment (`dev`, `staging`, `prod`) | `string` | `"prod"` |
| `region` | Target AWS Region | `string` | `"us-east-1"` |
| `enable_anomaly_detection` | Enable statistical metric anomaly scoring Lambda | `bool` | `true` |
| `enable_bedrock` | Enable Bedrock LLM for automated log root-cause analysis | `bool` | `false` |
| `bedrock_model_id` | Foundation model ID for log analysis | `string` | `"anthropic.claude-3-sonnet-20240229-v1:0"` |
| `enable_cost_anomaly_detection` | Enable AWS Cost Anomaly Monitor | `bool` | `true` |
| `log_groups` | Target CloudWatch log groups to scan | `list(string)` | `[]` |
| `slack_webhook_url` | Webhook URL for Slack incident alerts | `string` | `""` |
| `pagerduty_integration_key` | Integration key for PagerDuty | `string` | `""` |
| `sns_email_addresses` | Email list for SNS alert distribution | `list(string)` | `[]` |

---

## 📊 Dashboards & Visualization

* **AWS CloudWatch Dashboard**: Automatically deployed and visible in the AWS Console under `aiops-monitoring-main-{environment}`.
* **Grafana Dashboards**: Importable JSON models located in `dashboards/grafana/`:
  * `infrastructure-overview.json`: Metric anomaly scores, Lambda invocations, and error rates.
  * `cost-analysis.json`: AWS service cost breakdowns and daily budget burn rates.

---

## 📝 License

This project is distributed under the MIT License. See the [LICENSE](LICENSE) file for complete details.
