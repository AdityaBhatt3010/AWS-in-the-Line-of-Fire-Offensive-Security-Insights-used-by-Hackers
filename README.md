
# 🚨 AWS in the Line of Fire: Offensive Security Insights used by Hackers 🚨

A curated, practical guide to commonly exploited AWS misconfigurations — with commands, attack scenarios, and fixes.

---

## **Introduction**  
Cloud security in 2025 is no longer just about locking down S3 buckets — it's about understanding the attacker mindset and securing every IAM permission, API, and audit trail. AWS offers immense flexibility, but with that comes the opportunity for devastating misconfigurations. In this article, I’ll walk you through **the most exploited AWS misconfigurations today**, real-world case studies, **practical attack vectors**, and **how to fix them**.

---

## **1. S3 Buckets Still Leaking Gold**  
Despite all the red flags over the years, **misconfigured S3 buckets** continue to be a prime target for attackers.

### **Common Misconfigs:**  
- Public Access Enabled (`BlockPublicAcls` = False)  
- Authenticated Users Have Access (trusting "aws users" generally)  
- Bucket Policies with wildcard `"Principal": "*"`

### **Real-World Incident:**  
In early 2025, a fintech startup leaked over 80,000 customer documents via an S3 bucket with an open ACL meant for “testing”.

### **Practical Recon Command (Unauthenticated):**  
```bash
aws s3 ls s3://target-bucket-name --no-sign-request
```

### **Fix It Right:**  
```bash
aws s3api put-public-access-block \
  --bucket your-bucket-name \
  --public-access-block-configuration \
  BlockPublicAcls=true \
  IgnorePublicAcls=true \
  BlockPublicPolicy=true \
  RestrictPublicBuckets=true
```

Also, enable **Amazon Macie** for sensitive data detection.

---

## **2. IAM Privilege Escalation Chains**  
Attackers don’t need `AdminAccess` — they just need misconfigured IAM roles with indirect escalations.

### **Common Attack Vectors:**  
- `iam:PassRole` + `lambda:CreateFunction`  
- `iam:CreatePolicyVersion`  
- `ec2:RunInstances` with `user-data` containing attack scripts

### **Real Attack Simulation:**  
```bash
aws lambda create-function --function-name evilFunc \
  --runtime python3.8 \
  --handler lambda_function.lambda_handler \
  --role arn:aws:iam::account-id:role/AdminRole \
  --zip-file fileb://malicious_payload.zip
```

### **Fix It Right:**  
- Run **Pacu** to simulate privilege chains:
```bash
./pacu.py
> use iam__privesc_scan
> run
```
- Use **IAM Access Analyzer** to review trust relationships.
- Enforce **policy boundaries**.

---

## **3. Metadata Service Abuse (SSRF to Root Access)**  
When applications running on EC2 expose SSRF vulnerabilities, attackers can harvest metadata from the instance.

### **Old vs New:**  
- **IMDSv1:** Open to SSRF  
- **IMDSv2:** Requires session tokens

### **Payload:**  
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/RoleName
```

### **Fix It Right:**  
```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-endpoint enabled \
  --http-protocol-ipv6 disabled \
  --http-put-response-hop-limit 2 \
  --http-tokens required
```

---

## **4. Serverless Over-Permissioning (Lambda + API Gateway)**  
Lambda functions often have **broad IAM roles** or **APIs with no auth**, exposing critical backends.

### **Common Mistake:**  
- Lambda with `AdministratorAccess`  
- API Gateway route with no `API Key` or `JWT` check

### **Real Attack Flow:**  
1. Discover endpoint via recon tools  
2. Send crafted payload to invoke internal Lambda  
3. Exploit overprivileged Lambda to access S3, RDS, or assume roles

### **Fix It Right:**  
- Add IAM Conditions:
```json
"Condition": {
  "StringEqualsIfExists": {
    "aws:SourceVpce": "vpce-1234abcd"
  }
}
```
- Enforce auth at **API Gateway** with:
  - API Keys
  - JWT Authorizers
  - WAF Rate Limiting

---

## **5. Secrets Exposed via Code, Scripts, and GitHub**  
Leaked AWS keys or plaintext secrets are goldmines.

### **Real Case:**  
2025 saw an ed-tech company leak its root AWS keys on GitHub in a Python `.env` file. Result: EC2 mining, SES spamming, $22K bill.

### **Detection:**  
```bash
trufflehog github --repo=https://github.com/your-repo --branch=main
```

### **Fix It Right:**  
- Use **AWS Secrets Manager** or **SSM Parameter Store**
- Rotate secrets regularly:
```bash
aws secretsmanager rotate-secret --secret-id your-secret-name
```
- Use **GitHub Secret Scanning** and commit hooks.

---

## **6. Poor Logging and Monitoring Hygiene**  
Without CloudTrail and GuardDuty, attackers live rent-free.

### **Checklist:**  
- **CloudTrail:** Not enabled in all regions  
- **GuardDuty:** Disabled or misconfigured  
- **Config Rules:** Not enforced

### **Practical Enablement:**  
```bash
aws cloudtrail create-trail \
  --name orgTrail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail
```

Also, automate detection with **Prowler**:
```bash
./prowler -M html,csv,json -S -A <ACCOUNT_ID>
```

---

## **7. Cross-Account Role Trust Misuse**  
Attackers exploit **overly trusted roles** with wildcards in `Principal`.

### **Example:**  
```json
"Principal": {
  "AWS": "*"
}
```

### **Fix It Right:**  
- Use **explicit account IDs**  
- Attach **external ID** conditions
- Run **CloudSplaining** to detect bad policies:
```bash
cloudsplaining scan --input-file iam_policies.json
```

---

## **8. Open Security Groups (Inbound Hellfire)**  
Security groups are firewalls — and attackers love when they’re full of holes.

### **Common Misconfigurations:**  
- `0.0.0.0/0` allowed for SSH (port 22) or RDP (3389)  
- All inbound TCP/UDP ports open  
- Leaving critical services like MongoDB, Redis, MySQL exposed

### **Real-World Attack:**  
An attacker scanned port 3306 (MySQL) on an AWS IP block, brute-forced credentials, and dumped sensitive data.

### **Recon Example:**  
```bash
nmap -Pn -p- ec2-xx-xx-xx-xx.compute-1.amazonaws.com
```

### **Fix It Right:**  
```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-123abc \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

- Restrict access to **known IPs only**
- Use **AWS Systems Manager (SSM Session Manager)** instead of direct SSH

---

## **9. Default VPC Usage with Over-Permissive NACLs**  
The default VPC is often left unchanged, which means **lax NACL rules and public subnets**.

### **Exploitation Risk:**  
- Lateral movement from one EC2 to another  
- Unrestricted outbound rules used for **C2 channels** or **data exfiltration**

### **Practical Detection:**  
Use **AWS Config** or a script like:
```bash
aws ec2 describe-network-acls \
  --query 'NetworkAcls[*].Entries[?Egress==`false`]' \
  --output table
```

### **Fix It Right:**  
- Customize your own VPC with **tight NACLs**
- Use **flow logs** and monitor egress traffic:
```bash
aws ec2 create-flow-logs \
  --resource-ids vpc-123abc \
  --traffic-type ALL \
  --log-group-name vpc-flow-logs
```

---

## **10. Lambda Environment Variable Secrets**  
Developers often store DB passwords, API keys, or tokens directly in Lambda env vars for quick access.

### **Why It’s Dangerous:**  
Anyone with **Lambda read access** can dump secrets:
```bash
aws lambda get-function-configuration \
  --function-name myFunc
```

### **Real Attack Example:**  
Red teamers accessed the `.env` variables of a Lambda function and used the API key to pivot into a payment gateway’s sandbox environment.

### **Fix It Right:**  
- NEVER store secrets in environment variables  
- Use **AWS Secrets Manager**:
```bash
aws secretsmanager create-secret \
  --name MyDBSecret \
  --secret-string '{"username":"admin","password":"SuperSecret123"}'
```

- Fetch securely inside Lambda:
```python
import boto3
client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='MyDBSecret')
```

---

## **Conclusion: The 2025 AWS Misconfig Checklist**  

| Area | Misconfig | Fix |
|------|-----------|-----|
| S3 | Public buckets | Block public access, enable Macie |
| IAM | Escalation chains | Least privilege, use Pacu & Analyzer |
| EC2 | Metadata abuse, open ports | IMDSv2, restrict SGs |
| Lambda | Env secrets, over-privileged roles | Secrets Manager, policy minimization |
| VPC | Default use, open NACLs | Custom VPCs, monitor flow logs |
| API Gateway | No auth, weak keys | Use JWT/Auth, WAF |
| Secrets | Hardcoded or leaked | Use Secrets Manager, rotate |
| Logging | Missing audit trails | Enable CloudTrail, GuardDuty |
| Trust | Wildcard principals | Use ExternalId, lock trust relationships |

---

## **Final Word**  
Misconfigurations are inevitable — but **unmonitored** misconfigurations are **breaches in waiting**. Red teamers know where to look, and as defenders, so should you.

**Use tools like Pacu, ScoutSuite, and Prowler regularly.** Audit every policy. Rotate every secret. And above all — **don’t assume the default is secure.**

---

