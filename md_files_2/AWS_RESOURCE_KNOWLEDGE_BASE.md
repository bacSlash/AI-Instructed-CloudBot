# AWS Resource Knowledge Base

## Overview

This document contains comprehensive information about AWS services, their configurations, dependencies, and relationships. This knowledge base powers the intelligent question sequencing and dependency resolution in TerraConverse.

## Supported AWS Services

### Compute Services

#### EC2 (Elastic Compute Cloud)

**Service Type**: `aws_instance`  
**Category**: Compute  
**Description**: Virtual servers in the cloud

**Required Parameters**:
- `ami_id`: Amazon Machine Image ID
- `instance_type`: Instance size (t3.micro, t3.medium, etc.)

**Optional Parameters**:
- `key_name`: SSH key pair name
- `user_data`: Initialization script
- `iam_instance_profile`: IAM role for instance
- `monitoring`: Enable detailed monitoring
- `ebs_optimized`: Enable EBS optimization
- `root_block_device`: Root volume configuration
- `ebs_block_device`: Additional EBS volumes

**Network Parameters**:
- `subnet_id`: Subnet to launch in
- `vpc_security_group_ids`: List of security group IDs
- `associate_public_ip_address`: Assign public IP
- `private_ip`: Specific private IP address
- `source_dest_check`: Enable source/destination checking

**Dependencies**:
```python
{
    "required": ["vpc", "subnet"],
    "recommended": ["security_group", "key_pair"],
    "optional": ["elastic_ip", "iam_role", "ebs_volume"],
    "creates": []
}
```

**Common Use Cases**:
- Web servers (with ALB)
- Application servers
- Bastion hosts
- Build servers

**Security Best Practices**:
- Use IMDSv2 for instance metadata
- Enable EBS encryption
- Use Systems Manager instead of SSH
- Restrict security groups
- Use IAM roles instead of access keys

**Example Questions Flow**:
1. "What instance type do you need?" (suggest based on use case)
2. "Which AMI should I use?" (suggest Amazon Linux 2 or Ubuntu)
3. "Do you need a public IP address?" (explain implications)
4. "Should I create an SSH key pair?" (recommend Systems Manager instead)
5. "What size should the root volume be?" (default 20GB)

---

#### Lambda

**Service Type**: `aws_lambda_function`  
**Category**: Compute (Serverless)  
**Description**: Run code without managing servers

**Required Parameters**:
- `function_name`: Name of the Lambda function
- `runtime`: Execution environment (python3.11, nodejs18.x, etc.)
- `handler`: Entry point (e.g., index.handler)
- `role`: IAM role ARN
- `filename` or `s3_bucket`+`s3_key`: Code location

**Optional Parameters**:
- `memory_size`: Memory allocation (128-10240 MB)
- `timeout`: Execution timeout (1-900 seconds)
- `environment`: Environment variables
- `layers`: Lambda layers
- `vpc_config`: VPC configuration
- `dead_letter_config`: DLQ configuration
- `reserved_concurrent_executions`: Concurrency limit

**Dependencies**:
```python
{
    "required": ["iam_role"],
    "recommended": ["cloudwatch_log_group"],
    "optional": ["vpc", "subnet", "security_group", "kms_key", "s3_bucket"],
    "creates": ["cloudwatch_log_group"]
}
```

**IAM Role Permissions Needed**:
- Basic execution role (logs)
- VPC execution role (if in VPC)
- Service-specific permissions

---

### Networking Services

#### VPC (Virtual Private Cloud)

**Service Type**: `aws_vpc`  
**Category**: Network  
**Description**: Isolated virtual network

**Required Parameters**:
- `cidr_block`: IP address range (e.g., 10.0.0.0/16)

**Optional Parameters**:
- `enable_dns_hostnames`: Enable DNS hostnames (default: true)
- `enable_dns_support`: Enable DNS resolution (default: true)
- `instance_tenancy`: default or dedicated
- `ipv6_cidr_block`: IPv6 CIDR block
- `assign_generated_ipv6_cidr_block`: Auto-assign IPv6

**Dependencies**:
```python
{
    "required": [],
    "recommended": ["internet_gateway", "subnet"],
    "optional": ["dhcp_options", "vpc_endpoint"],
    "creates": ["default_security_group", "default_route_table", "default_nacl"]
}
```

**CIDR Block Guidelines**:
- Recommended ranges: 10.0.0.0/16, 172.16.0.0/16, 192.168.0.0/16
- Size: /16 to /28
- Cannot overlap with other VPCs if peering
- Consider future growth

**Example Questions Flow**:
1. "What CIDR block for your VPC?" (suggest 10.0.0.0/16)
2. "How many availability zones?" (recommend 2-3)
3. "Do you need public and private subnets?" (explain use cases)

---

#### Subnet

**Service Type**: `aws_subnet`  
**Category**: Network  
**Description**: Network segment within VPC

**Required Parameters**:
- `vpc_id`: Parent VPC
- `cidr_block`: Subnet IP range
- `availability_zone`: AZ placement

**Optional Parameters**:
- `map_public_ip_on_launch`: Auto-assign public IPs
- `ipv6_cidr_block`: IPv6 range
- `assign_ipv6_address_on_creation`: Auto-assign IPv6

**Dependencies**:
```python
{
    "required": ["vpc"],
    "recommended": ["route_table"],
    "optional": ["network_acl"],
    "creates": []
}
```

**Subnet Design Patterns**:
```
Pattern: Public + Private Subnets
- Public Subnet: 10.0.1.0/24 (web tier)
  - Has Internet Gateway route
  - Resources get public IPs
  
- Private Subnet: 10.0.2.0/24 (app tier)
  - Routes through NAT Gateway
  - No public IPs
  
- Private Subnet: 10.0.3.0/24 (data tier)
  - No internet access
  - Database instances
```

**CIDR Calculation**:
- /24 = 256 IPs (251 usable, AWS reserves 5)
- /25 = 128 IPs (123 usable)
- /26 = 64 IPs (59 usable)
- /27 = 32 IPs (27 usable)

---

#### Security Group

**Service Type**: `aws_security_group`  
**Category**: Network Security  
**Description**: Virtual firewall for instances

**Required Parameters**:
- `name`: Security group name
- `vpc_id`: Parent VPC
- `description`: Purpose description

**Optional Parameters**:
- `ingress`: Inbound rules
- `egress`: Outbound rules (default: allow all)

**Rule Structure**:
```python
{
    "from_port": 80,
    "to_port": 80,
    "protocol": "tcp",
    "cidr_blocks": ["0.0.0.0/0"],  # or security_groups or self
    "description": "Allow HTTP from anywhere"
}
```

**Dependencies**:
```python
{
    "required": ["vpc"],
    "recommended": [],
    "optional": [],
    "creates": []
}
```

**Security Rules Best Practices**:
- **Never allow SSH (22) from 0.0.0.0/0**
- **Never allow RDP (3389) from 0.0.0.0/0**
- Restrict database ports to app security group only
- Use security group references instead of CIDR when possible
- Add descriptions to all rules
- Follow principle of least privilege

**Common Patterns**:
```
Web Server SG:
- Ingress: 80, 443 from 0.0.0.0/0
- Ingress: 22 from bastion SG only
- Egress: All

App Server SG:
- Ingress: 8080 from web server SG
- Ingress: 22 from bastion SG only
- Egress: 443 to 0.0.0.0/0 (for API calls)
- Egress: 3306 to database SG

Database SG:
- Ingress: 3306 from app server SG only
- Egress: None (or minimal)
```

---

### Database Services

#### RDS (Relational Database Service)

**Service Type**: `aws_db_instance`  
**Category**: Database  
**Description**: Managed relational database

**Required Parameters**:
- `engine`: postgres, mysql, mariadb, oracle-ee, sqlserver-ex
- `engine_version`: Specific version
- `instance_class`: db.t3.micro, db.t3.medium, etc.
- `allocated_storage`: Storage size in GB (20-65536)
- `username`: Master username
- `password`: Master password (8+ chars)

**Optional Parameters**:
- `db_name`: Initial database name
- `port`: Database port (default based on engine)
- `db_subnet_group_name`: Subnet group
- `vpc_security_group_ids`: Security groups
- `publicly_accessible`: false (strongly recommended)
- `multi_az`: true for high availability
- `storage_type`: gp3, gp2, io1
- `iops`: For io1 storage
- `backup_retention_period`: Days (0-35)
- `backup_window`: Preferred backup time
- `maintenance_window`: Preferred maintenance time
- `storage_encrypted`: true (strongly recommended)
- `kms_key_id`: KMS key for encryption
- `deletion_protection`: true for production
- `enabled_cloudwatch_logs_exports`: Log types to export
- `performance_insights_enabled`: Enable Performance Insights
- `auto_minor_version_upgrade`: Auto upgrade minor versions

**Dependencies**:
```python
{
    "required": ["vpc", "db_subnet_group"],
    "recommended": ["security_group", "kms_key"],
    "optional": ["db_parameter_group", "db_option_group", "monitoring_role"],
    "creates": []
}
```

**DB Subnet Group**:
- Requires 2+ subnets in different AZs
- Use private subnets
- Separate from application subnets

**Engine-Specific Defaults**:
```python
ENGINE_DEFAULTS = {
    "postgres": {"port": 5432, "family": "postgres14"},
    "mysql": {"port": 3306, "family": "mysql8.0"},
    "mariadb": {"port": 3306, "family": "mariadb10.6"},
    "oracle-ee": {"port": 1521, "family": "oracle-ee-19"},
    "sqlserver-ex": {"port": 1433, "family": "sqlserver-ex-15.0"}
}
```

**Security Best Practices**:
- NEVER set publicly_accessible = true
- ALWAYS enable storage_encrypted
- ALWAYS enable deletion_protection for production
- Set backup_retention_period >= 7 days
- Use secrets manager for passwords
- Enable Performance Insights
- Enable Enhanced Monitoring

**Cost Optimization**:
- Use gp3 instead of gp2 (cheaper, better performance)
- Right-size instance class
- Use Reserved Instances for production
- Consider Aurora for scale

---

#### DynamoDB

**Service Type**: `aws_dynamodb_table`  
**Category**: Database (NoSQL)  
**Description**: Managed NoSQL database

**Required Parameters**:
- `name`: Table name
- `hash_key`: Partition key attribute
- `attribute`: List of attribute definitions

**Optional Parameters**:
- `range_key`: Sort key attribute
- `billing_mode`: PROVISIONED or PAY_PER_REQUEST
- `read_capacity`: For PROVISIONED mode
- `write_capacity`: For PROVISIONED mode
- `global_secondary_index`: GSI definitions
- `local_secondary_index`: LSI definitions
- `ttl`: Time to live configuration
- `stream_enabled`: Enable DynamoDB streams
- `stream_view_type`: Stream view type
- `server_side_encryption`: Encryption configuration
- `point_in_time_recovery`: Enable PITR

**Dependencies**:
```python
{
    "required": [],
    "recommended": ["kms_key"],
    "optional": ["iam_role"],
    "creates": []
}
```

---

### Storage Services

#### S3 (Simple Storage Service)

**Service Type**: `aws_s3_bucket`  
**Category**: Storage  
**Description**: Object storage

**Required Parameters**:
- `bucket`: Bucket name (globally unique)

**Important S3 Configurations** (separate resources in Terraform):
- `aws_s3_bucket_versioning`: Version control
- `aws_s3_bucket_encryption`: Server-side encryption
- `aws_s3_bucket_public_access_block`: Block public access
- `aws_s3_bucket_lifecycle_configuration`: Lifecycle rules
- `aws_s3_bucket_logging`: Access logging
- `aws_s3_bucket_cors_configuration`: CORS rules
- `aws_s3_bucket_policy`: Bucket policy

**Dependencies**:
```python
{
    "required": [],
    "recommended": [],
    "optional": ["kms_key", "cloudfront_distribution"],
    "creates": []
}
```

**Security Configuration** (CRITICAL):
```hcl
# ALWAYS block public access
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ALWAYS enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.example.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"  # or aws:kms with kms_master_key_id
    }
  }
}

# ALWAYS enable versioning for important data
resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.example.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

**Bucket Naming Rules**:
- 3-63 characters
- Lowercase letters, numbers, hyphens
- Must start and end with letter or number
- No uppercase, no underscores
- Globally unique across all AWS accounts

**Use Cases**:
- Static website hosting
- Application data storage
- Backup and archive
- Data lake
- CloudFront origin

---

### Container Services

#### ECS (Elastic Container Service)

**Service Type**: `aws_ecs_cluster`  
**Category**: Compute (Containers)  
**Description**: Container orchestration

**Components**:
1. **Cluster**: Logical grouping
2. **Task Definition**: Container specifications
3. **Service**: Runs and maintains tasks
4. **Task**: Running container instances

**Task Definition Parameters**:
- `family`: Task definition family name
- `container_definitions`: JSON container specs
- `cpu`: Task-level CPU (Fargate)
- `memory`: Task-level memory (Fargate)
- `network_mode`: awsvpc, bridge, host
- `requires_compatibilities`: EC2, FARGATE
- `execution_role_arn`: IAM role for ECS agent
- `task_role_arn`: IAM role for containers

**ECS Service Parameters**:
- `cluster`: ECS cluster ARN
- `task_definition`: Task definition ARN
- `desired_count`: Number of tasks
- `launch_type`: FARGATE or EC2
- `network_configuration`: VPC config (Fargate)
- `load_balancer`: ALB configuration
- `deployment_configuration`: Rolling update config

**Dependencies**:
```python
{
    "required": ["ecs_cluster", "task_definition", "iam_role"],
    "recommended": ["alb", "target_group", "security_group", "subnet"],
    "optional": ["cloudwatch_log_group", "service_discovery"],
    "creates": []
}
```

**Fargate vs EC2 Launch Type**:
- **Fargate**: Serverless, pay per task, easier to manage
- **EC2**: More control, potentially cheaper at scale, requires instance management

---

## Resource Relationship Patterns

### Pattern: 3-Tier Web Application

```
Resources Needed:
├── Network Layer
│   ├── VPC
│   ├── Public Subnets (2 AZs)
│   ├── Private Subnets (2 AZs)
│   ├── Internet Gateway
│   ├── NAT Gateway (per AZ)
│   └── Route Tables
│
├── Security Layer
│   ├── ALB Security Group (80, 443 from internet)
│   ├── Web Security Group (80, 443 from ALB SG)
│   ├── App Security Group (8080 from Web SG)
│   └── DB Security Group (3306 from App SG)
│
├── Compute Layer
│   ├── Application Load Balancer (public subnets)
│   ├── EC2 Instances or ECS (private subnets)
│   └── Auto Scaling Group
│
└── Data Layer
    ├── RDS Instance (private subnets)
    ├── DB Subnet Group
    └── S3 Bucket (static assets)
```

### Pattern: Serverless API

```
Resources Needed:
├── API Gateway (REST API)
├── Lambda Function(s)
├── DynamoDB Table
├── IAM Roles (Lambda execution + DynamoDB access)
├── CloudWatch Log Groups
└── Optional: VPC for Lambda (if accessing RDS)
```

### Pattern: Container Application

```
Resources Needed:
├── Network (VPC, Subnets, Security Groups)
├── ECS Cluster
├── ECS Task Definition
├── ECS Service
├── Application Load Balancer
├── Target Group
├── IAM Roles (execution + task)
├── CloudWatch Log Groups
└── Optional: ECR Repository
```

## Parameter Validation Rules

### CIDR Block Validation

```python
def validate_cidr(cidr: str, resource_type: str) -> ValidationResult:
    """
    VPC: /16 to /28
    Subnet: Must be subset of VPC, /16 to /28
    Security Group: Any valid CIDR
    """
    import ipaddress
    
    try:
        network = ipaddress.ip_network(cidr)
        prefix_len = network.prefixlen
        
        if resource_type == "vpc":
            if not (16 <= prefix_len <= 28):
                return ValidationResult(
                    is_valid=False,
                    errors=["VPC CIDR must be between /16 and /28"]
                )
            # Check RFC 1918 private ranges
            if not network.is_private:
                return ValidationResult(
                    is_valid=False,
                    errors=["VPC CIDR should use private IP ranges"]
                )
                
        return ValidationResult(is_valid=True)
        
    except ValueError as e:
        return ValidationResult(
            is_valid=False,
            errors=[f"Invalid CIDR format: {str(e)}"]
        )
```

### Instance Type Validation

```python
def validate_instance_type(instance_type: str, region: str) -> ValidationResult:
    """Check if instance type available in region"""
    # This would call AWS API in real implementation
    # boto3.client('ec2').describe_instance_type_offerings()
    
    COMMON_TYPES = [
        "t2.micro", "t2.small", "t3.micro", "t3.small", "t3.medium",
        "t3.large", "m5.large", "m5.xlarge", "c5.large", "r5.large"
    ]
    
    if instance_type not in COMMON_TYPES:
        return ValidationResult(
            is_valid=True,  # Don't block, but warn
            warnings=[f"Uncommon instance type: {instance_type}. Verify availability in {region}"]
        )
    
    return ValidationResult(is_valid=True)
```

---