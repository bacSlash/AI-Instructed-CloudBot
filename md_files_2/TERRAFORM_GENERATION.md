# Terraform Code Generation

## Overview

This document specifies the Terraform HCL generation logic, templates, formatting rules, and file organization for TerraConverse.

## Generation Pipeline

```
1. Resource Organization
   ↓
2. Template Selection
   ↓
3. HCL Generation
   ↓
4. Variable Extraction
   ↓
5. Output Generation
   ↓
6. File Organization
   ↓
7. Code Formatting
   ↓
8. Validation
```

## Jinja2 Template System

### Template Directory Structure

```
templates/
├── base/
│   ├── provider.tf.j2
│   ├── terraform_block.tf.j2
│   └── locals.tf.j2
├── compute/
│   ├── ec2_instance.tf.j2
│   ├── autoscaling_group.tf.j2
│   ├── launch_template.tf.j2
│   ├── lambda_function.tf.j2
│   └── ecs_cluster.tf.j2
├── network/
│   ├── vpc.tf.j2
│   ├── subnet.tf.j2
│   ├── security_group.tf.j2
│   ├── route_table.tf.j2
│   ├── internet_gateway.tf.j2
│   └── nat_gateway.tf.j2
├── database/
│   ├── rds_instance.tf.j2
│   ├── db_subnet_group.tf.j2
│   ├── dynamodb_table.tf.j2
│   └── elasticache_cluster.tf.j2
└── storage/
    ├── s3_bucket.tf.j2
    ├── ebs_volume.tf.j2
    └── efs_filesystem.tf.j2
```

### Base Template: Provider Configuration

**File**: `templates/base/provider.tf.j2`

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "{{ project_name }}"
      Environment = "{{ environment }}"
      ManagedBy   = "TerraConverse"
      CreatedAt   = "{{ timestamp }}"
    }
  }
}
```

### VPC Template

**File**: `templates/network/vpc.tf.j2`

```hcl
# VPC
resource "aws_vpc" "{{ vpc_name }}" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = {{ enable_dns_hostnames|lower }}
  enable_dns_support   = {{ enable_dns_support|lower }}
  instance_tenancy     = "{{ instance_tenancy }}"
  
  tags = {
    Name = "{{ vpc_name }}"
  }
}

{% if create_internet_gateway %}
# Internet Gateway
resource "aws_internet_gateway" "{{ vpc_name }}" {
  vpc_id = aws_vpc.{{ vpc_name }}.id
  
  tags = {
    Name = "{{ vpc_name }}-igw"
  }
}
{% endif %}

{% if create_flow_logs %}
# VPC Flow Logs
resource "aws_flow_log" "{{ vpc_name }}" {
  vpc_id          = aws_vpc.{{ vpc_name }}.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.vpc_flow_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn
  
  tags = {
    Name = "{{ vpc_name }}-flow-logs"
  }
}

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/{{ vpc_name }}/flow-logs"
  retention_in_days = 7
}

resource "aws_iam_role" "vpc_flow_logs" {
  name = "{{ vpc_name }}-flow-logs-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "vpc-flow-logs.amazonaws.com"
      }
    }]
  })
}
{% endif %}
```

### EC2 Instance Template

**File**: `templates/compute/ec2_instance.tf.j2`

```hcl
# EC2 Instance: {{ instance_name }}
resource "aws_instance" "{{ instance_name }}" {
  ami           = var.{{ instance_name }}_ami_id
  instance_type = var.{{ instance_name }}_instance_type
  
  subnet_id              = aws_subnet.{{ subnet_name }}.id
  vpc_security_group_ids = [aws_security_group.{{ security_group_name }}.id]
  
  {% if key_name %}
  key_name = var.{{ instance_name }}_key_name
  {% endif %}
  
  {% if associate_public_ip %}
  associate_public_ip_address = true
  {% endif %}
  
  {% if iam_instance_profile %}
  iam_instance_profile = aws_iam_instance_profile.{{ instance_name }}.name
  {% endif %}
  
  {% if user_data %}
  user_data = base64encode(var.{{ instance_name }}_user_data)
  {% endif %}
  
  root_block_device {
    volume_size           = var.{{ instance_name }}_root_volume_size
    volume_type           = "{{ root_volume_type }}"
    encrypted             = true
    delete_on_termination = true
  }
  
  monitoring    = {{ monitoring|lower }}
  ebs_optimized = {{ ebs_optimized|lower }}
  
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2
    http_put_response_hop_limit = 1
  }
  
  tags = {
    Name = "{{ instance_name }}"
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

{% if create_elastic_ip %}
# Elastic IP
resource "aws_eip" "{{ instance_name }}" {
  instance = aws_instance.{{ instance_name }}.id
  domain   = "vpc"
  
  tags = {
    Name = "{{ instance_name }}-eip"
  }
  
  depends_on = [aws_internet_gateway.{{ vpc_name }}]
}
{% endif %}
```

### RDS Instance Template

**File**: `templates/database/rds_instance.tf.j2`

```hcl
# DB Subnet Group
resource "aws_db_subnet_group" "{{ db_name }}" {
  name       = "{{ db_name }}-subnet-group"
  subnet_ids = [
    aws_subnet.{{ subnet_name_1 }}.id,
    aws_subnet.{{ subnet_name_2 }}.id
  ]
  
  tags = {
    Name = "{{ db_name }}-subnet-group"
  }
}

# RDS Instance
resource "aws_db_instance" "{{ db_name }}" {
  identifier     = "{{ db_name }}"
  engine         = "{{ engine }}"
  engine_version = "{{ engine_version }}"
  instance_class = var.{{ db_name }}_instance_class
  
  allocated_storage     = var.{{ db_name }}_allocated_storage
  storage_type          = "{{ storage_type }}"
  storage_encrypted     = true
  {% if kms_key_id %}
  kms_key_id            = aws_kms_key.{{ db_name }}.arn
  {% endif %}
  
  db_name  = var.{{ db_name }}_database_name
  username = var.{{ db_name }}_master_username
  password = var.{{ db_name }}_master_password  # Use AWS Secrets Manager in production
  port     = {{ port }}
  
  db_subnet_group_name   = aws_db_subnet_group.{{ db_name }}.name
  vpc_security_group_ids = [aws_security_group.{{ security_group_name }}.id]
  publicly_accessible    = false  # Never make databases public
  
  multi_az               = {{ multi_az|lower }}
  backup_retention_period = {{ backup_retention_period }}
  backup_window          = "{{ backup_window }}"
  maintenance_window     = "{{ maintenance_window }}"
  
  deletion_protection    = {{ deletion_protection|lower }}
  skip_final_snapshot    = false
  final_snapshot_identifier = "{{ db_name }}-final-snapshot-${timestamp()}"
  
  enabled_cloudwatch_logs_exports = [
    {% for log_type in cloudwatch_log_exports %}
    "{{ log_type }}"{{ "," if not loop.last else "" }}
    {% endfor %}
  ]
  
  monitoring_interval = {{ monitoring_interval }}
  {% if monitoring_interval > 0 %}
  monitoring_role_arn = aws_iam_role.{{ db_name }}_monitoring.arn
  {% endif %}
  
  performance_insights_enabled = {{ performance_insights_enabled|lower }}
  {% if performance_insights_enabled %}
  performance_insights_retention_period = 7
  {% endif %}
  
  tags = {
    Name = "{{ db_name }}"
  }
}

{% if monitoring_interval > 0 %}
# Enhanced Monitoring IAM Role
resource "aws_iam_role" "{{ db_name }}_monitoring" {
  name = "{{ db_name }}-enhanced-monitoring"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "monitoring.rds.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "{{ db_name }}_monitoring" {
  role       = aws_iam_role.{{ db_name }}_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
{% endif %}
```

### Security Group Template

**File**: `templates/network/security_group.tf.j2`

```hcl
# Security Group: {{ sg_name }}
resource "aws_security_group" "{{ sg_name }}" {
  name        = "{{ sg_name }}"
  description = "{{ description }}"
  vpc_id      = aws_vpc.{{ vpc_name }}.id
  
  {% for rule in ingress_rules %}
  ingress {
    description      = "{{ rule.description }}"
    from_port        = {{ rule.from_port }}
    to_port          = {{ rule.to_port }}
    protocol         = "{{ rule.protocol }}"
    {% if rule.cidr_blocks %}
    cidr_blocks      = [{% for cidr in rule.cidr_blocks %}"{{ cidr }}"{{ "," if not loop.last else "" }}{% endfor %}]
    {% endif %}
    {% if rule.security_groups %}
    security_groups  = [{% for sg in rule.security_groups %}aws_security_group.{{ sg }}.id{{ "," if not loop.last else "" }}{% endfor %}]
    {% endif %}
  }
  {% endfor %}
  
  {% for rule in egress_rules %}
  egress {
    description      = "{{ rule.description }}"
    from_port        = {{ rule.from_port }}
    to_port          = {{ rule.to_port }}
    protocol         = "{{ rule.protocol }}"
    {% if rule.cidr_blocks %}
    cidr_blocks      = [{% for cidr in rule.cidr_blocks %}"{{ cidr }}"{{ "," if not loop.last else "" }}{% endfor %}]
    {% endif %}
  }
  {% endfor %}
  
  tags = {
    Name = "{{ sg_name }}"
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

## Variable Generation

### Variables File Template

**File**: Generated `variables.tf`

```hcl
# General Configuration
variable "aws_region" {
  description = "AWS region for resource deployment"
  type        = string
  default     = "{{ aws_region }}"
}

variable "project_name" {
  description = "Project name for resource tagging"
  type        = string
  default     = "{{ project_name }}"
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
  default     = "{{ environment }}"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}

# Network Configuration
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "{{ vpc_cidr }}"
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block"
  }
}

# EC2 Configuration
variable "{{ instance_name }}_instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "{{ instance_type }}"
}

variable "{{ instance_name }}_ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  default     = "{{ ami_id }}"
}

# RDS Configuration
variable "{{ db_name }}_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "{{ instance_class }}"
}

variable "{{ db_name }}_master_username" {
  description = "Master username for RDS"
  type        = string
  default     = "{{ master_username }}"
  sensitive   = true
}

variable "{{ db_name }}_master_password" {
  description = "Master password for RDS (use AWS Secrets Manager in production)"
  type        = string
  sensitive   = true
  
  validation {
    condition     = length(var.{{ db_name }}_master_password) >= 8
    error_message = "Password must be at least 8 characters"
  }
}
```

## Output Generation

### Outputs File Template

**File**: Generated `outputs.tf`

```hcl
# VPC Outputs
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.{{ vpc_name }}.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.{{ vpc_name }}.cidr_block
}

# Subnet Outputs
output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = [
    {% for subnet in public_subnets %}
    aws_subnet.{{ subnet }}.id{{ "," if not loop.last else "" }}
    {% endfor %}
  ]
}

# EC2 Outputs
output "{{ instance_name }}_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.{{ instance_name }}.id
}

output "{{ instance_name }}_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.{{ instance_name }}.public_ip
}

output "{{ instance_name }}_private_ip" {
  description = "Private IP of the EC2 instance"
  value       = aws_instance.{{ instance_name }}.private_ip
}

# RDS Outputs
output "{{ db_name }}_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.{{ db_name }}.endpoint
  sensitive   = true
}

output "{{ db_name }}_address" {
  description = "RDS instance address"
  value       = aws_db_instance.{{ db_name }}.address
  sensitive   = true
}

output "{{ db_name }}_port" {
  description = "RDS instance port"
  value       = aws_db_instance.{{ db_name }}.port
}
```

## Code Formatting

### Terraform Formatting

```python
import subprocess

def terraform_fmt(hcl_code: str) -> str:
    """
    Format HCL code using terraform fmt
    """
    try:
        # Write to temp file
        with tempfile.NamedTemporaryFile(mode='w', suffix='.tf', delete=False) as f:
            f.write(hcl_code)
            temp_file = f.name
        
        # Run terraform fmt
        result = subprocess.run(
            ['terraform', 'fmt', temp_file],
            capture_output=True,
            text=True
        )
        
        # Read formatted content
        with open(temp_file, 'r') as f:
            formatted_code = f.read()
        
        # Cleanup
        os.unlink(temp_file)
        
        return formatted_code
        
    except Exception as e:
        # Fallback to manual formatting
        return manual_format_hcl(hcl_code)

def manual_format_hcl(hcl_code: str) -> str:
    """Fallback HCL formatting"""
    lines = hcl_code.split('\n')
    formatted_lines = []
    indent_level = 0
    
    for line in lines:
        stripped = line.strip()
        
        # Decrease indent for closing braces
        if stripped.startswith('}'):
            indent_level = max(0, indent_level - 1)
        
        # Add indentation
        if stripped:
            formatted_lines.append('  ' * indent_level + stripped)
        else:
            formatted_lines.append('')
        
        # Increase indent for opening braces
        if stripped.endswith('{'):
            indent_level += 1
    
    return '\n'.join(formatted_lines)
```

## Code Validation

```python
def validate_terraform_syntax(files: Dict[str, str]) -> ValidationResult:
    """
    Validate generated Terraform code
    """
    with tempfile.TemporaryDirectory() as temp_dir:
        # Write all files
        for filename, content in files.items():
            filepath = os.path.join(temp_dir, filename)
            with open(filepath, 'w') as f:
                f.write(content)
        
        # Run terraform init
        init_result = subprocess.run(
            ['terraform', 'init'],
            cwd=temp_dir,
            capture_output=True,
            text=True
        )
        
        if init_result.returncode != 0:
            return ValidationResult(
                is_valid=False,
                errors=[f"Terraform init failed: {init_result.stderr}"]
            )
        
        # Run terraform validate
        validate_result = subprocess.run(
            ['terraform', 'validate'],
            cwd=temp_dir,
            capture_output=True,
            text=True
        )
        
        if validate_result.returncode != 0:
            return ValidationResult(
                is_valid=False,
                errors=[f"Terraform validate failed: {validate_result.stderr}"]
            )
        
        return ValidationResult(is_valid=True)
```

## README Generation

```python
def generate_readme(session: SessionState) -> str:
    """Generate comprehensive README.md"""
    
    resources_list = "\n".join([
        f"- {r.resource_type}: {r.description}"
        for r in session.resource_definitions.values()
    ])
    
    security_warnings = "\n".join([
        f"- {w.severity}: {w.message}"
        for w in session.warning_messages
        if 'security' in w.lower()
    ])
    
    readme = f"""# TerraConverse Generated Infrastructure

## Overview

This Terraform configuration was generated by TerraConverse on {session.generation_timestamp}.

## Resources

{resources_list}

## Prerequisites

- Terraform >= 1.0
- AWS CLI configured with appropriate credentials
- IAM permissions for all resources being created

## Deployment

### 1. Initialize Terraform

```bash
terraform init
```

### 2. Review the Plan

```bash
terraform plan
```

### 3. Apply the Configuration

```bash
terraform apply
```

## Security Warnings

{security_warnings if security_warnings else "No security warnings"}

## Important Notes

- Review all generated code before applying
- Update sensitive values in terraform.tfvars
- Consider using AWS Secrets Manager for passwords
- Enable additional monitoring and logging for production

## Cleanup

To destroy all resources:

```bash
terraform destroy
```

## Support

Generated by TerraConverse - Terraform AI Assistant
"""
    
    return readme
```

---