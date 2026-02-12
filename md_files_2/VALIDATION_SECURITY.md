# Validation and Security Specifications

## Parameter Validation Rules

### CIDR Block Validation
- VPC: Must be /16 to /28
- Must be from RFC 1918 private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- Subnets must be subsets of VPC CIDR
- No overlapping subnets

### AWS Resource Naming
- S3 buckets: 3-63 chars, lowercase, numbers, hyphens only, globally unique
- Resource names: alphanumeric, hyphens, underscores
- Tags: 128 chars max per key/value

### Instance Types
- Validate availability in selected region via AWS API
- Warn about expensive instance types

### Database Passwords
- Minimum 8 characters
- Recommend using AWS Secrets Manager

## Security Best Practices

### Critical Warnings
- SSH (22) or RDP (3389) from 0.0.0.0/0
- Databases publicly accessible
- S3 buckets without public access blocks
- Unencrypted storage (RDS, S3, EBS)

### Recommendations
- Use IMDSv2 for EC2
- Enable VPC Flow Logs
- Enable CloudWatch monitoring
- Use private subnets for databases
- Enable deletion protection for critical resources

---
