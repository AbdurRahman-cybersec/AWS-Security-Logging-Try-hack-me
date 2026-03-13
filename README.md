# aws-security-logging-soc-guide

> A practical SOC analyst guide to AWS security logging — CloudTrail, GuardDuty, VPC Flow Logs, CloudFront, and S3 data events with Splunk query examples.

---

## Overview

This repository documents key concepts, Splunk queries, and detection strategies for monitoring AWS environments in a Security Operations Center (SOC). It covers the three core security areas every AWS SOC analyst must address:

- **Control Plane** — Monitor actions in the AWS console and infrastructure management
- **Workloads** — Monitor files and processes on EC2 instances
- **Managed Services** — Monitor access to RDS, S3, CloudFront, and other AWS services

---

## Contents

| Section | Description |
|---|---|
| CloudTrail | Control plane logging, key fields, Splunk queries |
| GuardDuty | Automated threat detection, finding types, limitations |
| VPC Flow Logs | Network traffic visibility in AWS |
| CloudFront Logs | Web traffic logging, WAF-style detections |
| S3 Data Events | Object-level logging with CloudTrail |
| Workloads & Containers | EC2 monitoring, Fargate, Lambda considerations |

---

## Key AWS Log Sources

### CloudTrail
Records all AWS API calls. Essential fields:

```
eventSource      # AWS service (e.g. signin.amazonaws.com)
eventName        # API action (ConsoleLogin, RunInstances, CreateBucket)
sourceIPAddress  # Origin of the API call
userIdentity.*   # Actor ARN and identity details
readOnly         # false = write/risky action
recipientAccountId # Target AWS account
```

**Splunk queries:**
```splunk
index=aws eventName=ConsoleLogin
index=aws sourcetype=aws:cloudtrail eventName=RunInstances
index=aws sourcetype=aws:cloudtrail readOnly=false
```

### GuardDuty
Automated threat detection with built-in finding types:

| Finding Type | Description |
|---|---|
| `Exfiltration:IAMUser/AnomalousBehavior` | Suspicious control plane actions during data exfiltration |
| `Policy:S3/BucketAnonymousAccessGranted` | S3 bucket made publicly accessible |
| `UnauthorizedAccess:EC2/RDPBruteForce` | RDP brute force against exposed EC2 |
| `Impact:EC2/BitcoinDomainRequest.Reputation` | EC2 querying cryptocurrency mining domains |

**Splunk query:**
```splunk
index=aws sourcetype=aws:cloudwatch:guardduty
```

> Note: GuardDuty does not replace raw log collection. It lacks company-specific context, may miss advanced threats, and cannot reliably detect logging tampering.

### VPC Flow Logs
Network-level visibility across subnets:

```
srcaddr / dstaddr   # Source and destination IPs
srcport / dstport   # Network ports
protocol            # 6=TCP, 17=UDP, 1=ICMP
action              # ACCEPT or REJECT
```

### CloudFront Logs
Web traffic logs following the W3C standard:
- `c/cs` prefix = client-side fields
- `s/sc` prefix = server-side fields

```splunk
index=aws source="cloudfront.log" cs_uri_stem="/admin/login"
```

### S3 Data Events
Object-level operations (requires CloudTrail Data Events to be enabled):
```splunk
index=aws source="s3.json" eventName=GetObject
```

---

## Detection Examples

### Detect console logins from unusual IPs
```splunk
index=aws eventName=ConsoleLogin | stats count by sourceIPAddress, userIdentity.userName
```

### Find write events (high risk)
```splunk
index=aws sourcetype=aws:cloudtrail readOnly=false
```

### Detect public S3 bucket creation
```splunk
index=aws eventName=PutBucketAcl
```

### Hunt for new EC2 instances
```splunk
index=aws sourcetype=aws:cloudtrail eventName=RunInstances
```

---

## Control Plane Threat Areas

- **IAM**: Changes in user privileges, access keys, unusual logins
- **EC2**: Insecure security group changes (exposed RDP/SSH)
- **S3**: Buckets made public accidentally

For production-ready detection rules, refer to:
- [Sigma Rules](https://github.com/SigmaHQ/sigma) — search `AWS`
- [Elastic Detection Rules](https://github.com/elastic/detection-rules) — search `AWS`

---

## Managed vs Unmanaged Services

| Type | Monitoring Approach |
|---|---|
| AWS-Managed (S3, RDS, Aurora) | Enable AWS service logs via API |
| Unmanaged (self-hosted on EC2) | Install SIEM agent + Auditd/Sysmon |
| Containers (ECS/Fargate) | Use cloud-native tools like Falco or CWPP |
| Serverless (Lambda) | Use CWPP or AWS-specific security guidelines |

---

## Resources

- [AWS CloudTrail Documentation](https://docs.aws.amazon.com/cloudtrail/)
- [AWS GuardDuty Documentation](https://docs.aws.amazon.com/guardduty/)
- [Splunk AWS Add-on](https://splunkbase.splunk.com/app/1876/)
- [Sigma Rules for AWS](https://github.com/SigmaHQ/sigma)

---

## License

MIT
