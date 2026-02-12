# Data Models and Schemas

## Overview

This document defines all data structures, schemas, and models used throughout the TerraConverse application. These models ensure type safety, validation, and consistency across components.

## Core Data Models

### 1. Message Model

Represents a single message in the conversation.

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Optional

class MessageRole(Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"

class MessageType(Enum):
    TEXT = "text"
    QUESTION = "question"
    CONFIRMATION = "confirmation"
    ERROR = "error"
    WARNING = "warning"
    INFO = "info"

@dataclass
class Message:
    """Represents a single conversation message"""
    role: MessageRole
    content: str
    timestamp: datetime
    message_type: MessageType = MessageType.TEXT
    metadata: Optional[dict] = None
    
    # For questions
    parameter_name: Optional[str] = None
    suggested_value: Optional[str] = None
    explanation: Optional[str] = None
    
    # For validations
    is_valid: Optional[bool] = None
    validation_errors: Optional[list[str]] = None
```

**Usage Example**:
```python
message = Message(
    role=MessageRole.ASSISTANT,
    content="What AWS region should I deploy to?",
    timestamp=datetime.now(),
    message_type=MessageType.QUESTION,
    parameter_name="region",
    suggested_value="us-east-1",
    explanation="The region determines where your resources will be located."
)
```

---

### 2. Resource Intent Model

Represents identified AWS resources from user's initial request.

```python
from pydantic import BaseModel, Field
from typing import List, Optional, Dict

class ResourceIntent(BaseModel):
    """Identified AWS resource from user intent"""
    resource_type: str = Field(..., description="AWS resource type (e.g., 'ec2_instance')")
    resource_category: str = Field(..., description="Category: compute, network, storage, database")
    purpose: str = Field(..., description="User's intended use for this resource")
    confidence_score: float = Field(ge=0.0, le=1.0, description="LLM confidence in identification")
    mentioned_explicitly: bool = Field(default=False, description="User explicitly mentioned this resource")
    
    # Metadata
    priority: int = Field(default=5, ge=1, le=10, description="Collection priority (1=highest)")
    requires_user_input: bool = Field(default=True, description="Needs user configuration")
    
    class Config:
        json_schema_extra = {
            "example": {
                "resource_type": "ec2_instance",
                "resource_category": "compute",
                "purpose": "web server",
                "confidence_score": 0.95,
                "mentioned_explicitly": True,
                "priority": 1
            }
        }
```

---

### 3. Parameter Schema Model

Defines schema for each configurable parameter.

```python
from pydantic import BaseModel, Field, validator
from typing import Any, List, Optional, Union
from enum import Enum

class ParameterType(str, Enum):
    STRING = "string"
    INTEGER = "integer"
    BOOLEAN = "boolean"
    LIST = "list"
    DICT = "dict"
    CIDR = "cidr"
    ARN = "arn"

class ParameterSchema(BaseModel):
    """Schema definition for a configuration parameter"""
    name: str = Field(..., description="Parameter name")
    display_name: str = Field(..., description="Human-readable name")
    type: ParameterType
    required: bool = Field(default=True)
    
    # Validation
    min_length: Optional[int] = None
    max_length: Optional[int] = None
    min_value: Optional[Union[int, float]] = None
    max_value: Optional[Union[int, float]] = None
    pattern: Optional[str] = Field(None, description="Regex pattern for validation")
    allowed_values: Optional[List[Any]] = Field(None, description="Enum of allowed values")
    
    # Documentation
    description: str = Field(..., description="What this parameter configures")
    explanation: str = Field(..., description="Why this parameter is needed")
    default_value: Optional[Any] = None
    example_values: List[str] = Field(default_factory=list)
    
    # Dependencies
    depends_on: List[str] = Field(default_factory=list, description="Parameters that must be set first")
    affects: List[str] = Field(default_factory=list, description="Parameters affected by this one")
    
    # AWS-specific
    aws_constraint: Optional[str] = Field(None, description="AWS-specific limitation")
    region_dependent: bool = Field(default=False)
    
    # Security
    security_sensitive: bool = Field(default=False)
    security_warning: Optional[str] = None
    
    @validator('pattern')
    def validate_pattern(cls, v):
        if v:
            import re
            try:
                re.compile(v)
            except re.error:
                raise ValueError(f"Invalid regex pattern: {v}")
        return v
```

**Example Parameter Schemas**:

```python
# EC2 Instance Type Parameter
instance_type_schema = ParameterSchema(
    name="instance_type",
    display_name="Instance Type",
    type=ParameterType.STRING,
    required=True,
    description="The EC2 instance type (defines CPU, memory, and network capacity)",
    explanation="Instance type determines the computing resources available to your server",
    default_value="t3.medium",
    example_values=["t3.micro", "t3.medium", "t3.large", "m5.large"],
    allowed_values=["t2.micro", "t2.small", "t3.micro", "t3.small", "t3.medium", "t3.large", 
                    "m5.large", "m5.xlarge", "c5.large", "r5.large"],
    region_dependent=True,
    aws_constraint="Not all instance types available in all regions"
)

# VPC CIDR Block Parameter
vpc_cidr_schema = ParameterSchema(
    name="vpc_cidr",
    display_name="VPC CIDR Block",
    type=ParameterType.CIDR,
    required=True,
    description="IP address range for the VPC in CIDR notation",
    explanation="Defines the private IP address space for your entire virtual network",
    default_value="10.0.0.0/16",
    example_values=["10.0.0.0/16", "172.16.0.0/16", "192.168.0.0/16"],
    pattern=r"^(\d{1,3}\.){3}\d{1,3}/\d{1,2}$",
    aws_constraint="Must be between /16 and /28, from RFC 1918 private ranges",
    affects=["subnet_cidr_public", "subnet_cidr_private"]
)

# Security Group Rule Parameter
sg_ingress_schema = ParameterSchema(
    name="sg_ingress_rules",
    display_name="Security Group Ingress Rules",
    type=ParameterType.LIST,
    required=True,
    description="Inbound traffic rules for the security group",
    explanation="Controls which external traffic can reach your resources",
    default_value=[{"port": 443, "protocol": "tcp", "cidr": "0.0.0.0/0"}],
    example_values=["22 from 10.0.0.0/16", "443 from anywhere", "3306 from sg-xxx"],
    security_sensitive=True,
    security_warning="Opening ports to 0.0.0.0/0 may expose resources to attacks"
)
```

---

### 4. Resource Definition Model

Complete definition of an AWS resource with all parameters.

```python
from pydantic import BaseModel, Field
from typing import Dict, List, Optional

class ResourceDefinition(BaseModel):
    """Complete AWS resource definition"""
    resource_id: str = Field(..., description="Unique identifier for this resource instance")
    resource_type: str = Field(..., description="AWS resource type")
    resource_name: str = Field(..., description="Terraform resource name")
    
    # Parameters
    parameters: Dict[str, Any] = Field(default_factory=dict, description="Collected parameter values")
    required_parameters: List[str] = Field(default_factory=list)
    optional_parameters: List[str] = Field(default_factory=list)
    
    # Completion status
    is_complete: bool = Field(default=False)
    completion_percentage: float = Field(default=0.0, ge=0.0, le=100.0)
    missing_parameters: List[str] = Field(default_factory=list)
    
    # Dependencies
    depends_on_resources: List[str] = Field(default_factory=list)
    referenced_by_resources: List[str] = Field(default_factory=list)
    
    # Validation
    validation_status: str = Field(default="pending")  # pending, valid, invalid
    validation_errors: List[str] = Field(default_factory=list)
    security_warnings: List[str] = Field(default_factory=list)
    
    # Metadata
    category: str
    description: str
    terraform_resource_type: str  # e.g., "aws_instance"
    
    def mark_parameter_collected(self, param_name: str, value: Any):
        """Mark a parameter as collected"""
        self.parameters[param_name] = value
        if param_name in self.missing_parameters:
            self.missing_parameters.remove(param_name)
        self._update_completion()
    
    def _update_completion(self):
        """Update completion status"""
        total_required = len(self.required_parameters)
        collected_required = sum(1 for p in self.required_parameters if p in self.parameters)
        self.completion_percentage = (collected_required / total_required * 100) if total_required > 0 else 100
        self.is_complete = collected_required == total_required
        self.missing_parameters = [p for p in self.required_parameters if p not in self.parameters]
```

---

### 5. Session State Model

Complete application state for a user session.

```python
from pydantic import BaseModel, Field
from typing import Dict, List, Optional
from datetime import datetime
from enum import Enum

class ConversationStage(str, Enum):
    INIT = "init"
    INTENT_PARSING = "intent_parsing"
    DEPENDENCY_ANALYSIS = "dependency_analysis"
    PARAMETER_COLLECTION = "parameter_collection"
    VALIDATION = "validation"
    GENERATION = "generation"
    COMPLETE = "complete"
    ERROR = "error"

class SessionState(BaseModel):
    """Complete session state"""
    # Session metadata
    session_id: str
    created_at: datetime
    last_updated: datetime
    
    # Conversation
    messages: List[Message] = Field(default_factory=list)
    conversation_stage: ConversationStage = ConversationStage.INIT
    
    # Intent and resources
    user_initial_request: Optional[str] = None
    identified_resources: List[ResourceIntent] = Field(default_factory=list)
    resource_definitions: Dict[str, ResourceDefinition] = Field(default_factory=dict)
    
    # Current processing
    current_resource_id: Optional[str] = None
    current_parameter: Optional[str] = None
    dependency_graph: Dict[str, List[str]] = Field(default_factory=dict)
    collection_order: List[str] = Field(default_factory=list)
    
    # Progress tracking
    total_parameters: int = 0
    collected_parameters: int = 0
    completion_percentage: float = 0.0
    
    # Validation and errors
    has_errors: bool = False
    error_messages: List[str] = Field(default_factory=list)
    has_warnings: bool = False
    warning_messages: List[str] = Field(default_factory=list)
    
    # Generated output
    terraform_code: Dict[str, str] = Field(default_factory=dict)  # filename -> content
    generation_timestamp: Optional[datetime] = None
    code_validated: bool = False
    
    # User preferences
    aws_region: Optional[str] = None
    preferred_defaults: Dict[str, Any] = Field(default_factory=dict)
    
    def add_message(self, message: Message):
        """Add message to conversation"""
        self.messages.append(message)
        self.last_updated = datetime.now()
    
    def update_progress(self):
        """Recalculate completion percentage"""
        if self.total_parameters > 0:
            self.completion_percentage = (self.collected_parameters / self.total_parameters) * 100
    
    def transition_stage(self, new_stage: ConversationStage):
        """Transition to new conversation stage"""
        self.conversation_stage = new_stage
        self.last_updated = datetime.now()
```

---

### 6. AWS Resource Schemas

Pre-defined schemas for all supported AWS resources.

```python
from typing import Dict
from pydantic import BaseModel, Field

class EC2InstanceResource(BaseModel):
    """EC2 Instance resource definition"""
    # Required
    instance_type: str
    ami_id: str
    
    # Network
    vpc_id: Optional[str] = None
    subnet_id: Optional[str] = None
    security_group_ids: List[str] = Field(default_factory=list)
    associate_public_ip: bool = False
    
    # Storage
    root_volume_size: int = Field(default=8, ge=8, le=16384)
    root_volume_type: str = Field(default="gp3")
    ebs_optimized: bool = True
    
    # Access
    key_name: Optional[str] = None
    iam_instance_profile: Optional[str] = None
    
    # Configuration
    user_data: Optional[str] = None
    monitoring: bool = True
    
    # Tags
    name: str
    environment: str = "development"
    managed_by: str = "TerraConverse"
    
    # Metadata
    depends_on: List[str] = Field(default_factory=lambda: ["vpc", "subnet", "security_group"])
    category: str = "compute"

class RDSInstanceResource(BaseModel):
    """RDS Database Instance resource definition"""
    # Required
    engine: str  # postgres, mysql, mariadb, oracle-ee, sqlserver-ex
    engine_version: str
    instance_class: str
    allocated_storage: int = Field(ge=20, le=65536)
    
    # Database
    db_name: Optional[str] = None
    master_username: str
    master_password: str = Field(min_length=8)
    port: Optional[int] = None
    
    # Network
    vpc_id: Optional[str] = None
    db_subnet_group_name: Optional[str] = None
    security_group_ids: List[str] = Field(default_factory=list)
    publicly_accessible: bool = False
    
    # Backup
    backup_retention_period: int = Field(default=7, ge=0, le=35)
    backup_window: Optional[str] = None
    maintenance_window: Optional[str] = None
    
    # Performance
    multi_az: bool = False
    storage_type: str = Field(default="gp3")
    iops: Optional[int] = None
    
    # Security
    storage_encrypted: bool = True
    kms_key_id: Optional[str] = None
    deletion_protection: bool = True
    
    # Monitoring
    enabled_cloudwatch_logs_exports: List[str] = Field(default_factory=list)
    monitoring_interval: int = Field(default=60)
    
    # Tags
    name: str
    environment: str = "development"
    
    # Metadata
    depends_on: List[str] = Field(default_factory=lambda: ["vpc", "db_subnet_group", "security_group"])
    category: str = "database"

class VPCResource(BaseModel):
    """VPC resource definition"""
    cidr_block: str = Field(pattern=r"^(\d{1,3}\.){3}\d{1,3}/\d{1,2}$")
    enable_dns_hostnames: bool = True
    enable_dns_support: bool = True
    
    # Optional
    instance_tenancy: str = Field(default="default")  # default, dedicated
    ipv6_cidr_block: Optional[str] = None
    
    # Tags
    name: str
    environment: str = "development"
    
    # Metadata
    depends_on: List[str] = Field(default_factory=list)
    category: str = "network"

class S3BucketResource(BaseModel):
    """S3 Bucket resource definition"""
    bucket_name: str = Field(pattern=r"^[a-z0-9][a-z0-9-]*[a-z0-9]$")
    
    # Access control
    acl: str = Field(default="private")
    block_public_acls: bool = True
    block_public_policy: bool = True
    ignore_public_acls: bool = True
    restrict_public_buckets: bool = True
    
    # Versioning
    versioning_enabled: bool = True
    
    # Encryption
    sse_algorithm: str = Field(default="AES256")  # AES256 or aws:kms
    kms_key_id: Optional[str] = None
    
    # Lifecycle
    lifecycle_rules: List[Dict] = Field(default_factory=list)
    
    # Logging
    logging_enabled: bool = False
    logging_target_bucket: Optional[str] = None
    logging_target_prefix: Optional[str] = None
    
    # Tags
    name: str
    environment: str = "development"
    
    # Metadata
    depends_on: List[str] = Field(default_factory=list)
    category: str = "storage"

# Additional resource schemas...
class LambdaFunctionResource(BaseModel): pass
class ECSClusterResource(BaseModel): pass
class ALBResource(BaseModel): pass
# ... etc
```

---

### 7. Validation Models

Models for validation results and errors.

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum

class ValidationSeverity(str, Enum):
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"

class ValidationResult(BaseModel):
    """Result of parameter validation"""
    is_valid: bool
    parameter_name: str
    value: Any
    
    # Issues
    errors: List[str] = Field(default_factory=list)
    warnings: List[str] = Field(default_factory=list)
    info: List[str] = Field(default_factory=list)
    
    # Suggestions
    suggested_fixes: List[str] = Field(default_factory=list)
    alternative_values: List[Any] = Field(default_factory=list)
    
    # Metadata
    validation_type: str  # format, constraint, aws_api, security
    validated_at: datetime = Field(default_factory=datetime.now)

class SecurityValidation(BaseModel):
    """Security-specific validation result"""
    severity: ValidationSeverity
    issue_type: str  # encryption, access_control, network, compliance
    message: str
    remediation: str
    affected_resources: List[str]
    can_auto_fix: bool = False
    auto_fix_description: Optional[str] = None
```

---

### 8. Terraform Generation Models

Models for Terraform code generation.

```python
from pydantic import BaseModel, Field
from typing import Dict, List

class TerraformResource(BaseModel):
    """Terraform resource block"""
    resource_type: str  # aws_instance, aws_vpc, etc.
    resource_name: str  # my_instance, main_vpc, etc.
    attributes: Dict[str, Any]
    depends_on: List[str] = Field(default_factory=list)
    lifecycle: Optional[Dict[str, Any]] = None
    provisioners: List[Dict] = Field(default_factory=list)

class TerraformVariable(BaseModel):
    """Terraform variable definition"""
    name: str
    type: str  # string, number, bool, list, map, object
    description: str
    default: Optional[Any] = None
    sensitive: bool = False
    validation: Optional[Dict] = None

class TerraformOutput(BaseModel):
    """Terraform output definition"""
    name: str
    value: str  # HCL expression
    description: str
    sensitive: bool = False

class TerraformCode(BaseModel):
    """Complete Terraform configuration"""
    provider_config: Dict[str, Any]
    resources: List[TerraformResource]
    variables: List[TerraformVariable]
    outputs: List[TerraformOutput]
    data_sources: List[Dict] = Field(default_factory=list)
    locals: Dict[str, Any] = Field(default_factory=dict)
```

---

## Dependency Graph Structure

```python
# Complete dependency graph for AWS resources
RESOURCE_DEPENDENCY_GRAPH: Dict[str, Dict[str, List[str]]] = {
    "ec2_instance": {
        "required": ["vpc", "subnet"],
        "recommended": ["security_group", "key_pair"],
        "optional": ["elastic_ip", "iam_role", "ebs_volume"]
    },
    "rds_instance": {
        "required": ["vpc", "db_subnet_group"],
        "recommended": ["security_group", "db_parameter_group"],
        "optional": ["kms_key", "iam_role", "monitoring_role"]
    },
    "lambda_function": {
        "required": ["iam_role"],
        "recommended": ["security_group", "subnet"],
        "optional": ["kms_key", "cloudwatch_log_group", "s3_bucket"]
    },
    "ecs_service": {
        "required": ["ecs_cluster", "task_definition", "subnet"],
        "recommended": ["security_group", "load_balancer", "iam_role"],
        "optional": ["service_discovery", "auto_scaling"]
    },
    "alb": {
        "required": ["vpc", "subnet"],
        "recommended": ["security_group", "target_group"],
        "optional": ["acm_certificate", "waf_web_acl"]
    }
}
```

---

## Enumerations and Constants

```python
# AWS Regions
AWS_REGIONS = [
    "us-east-1", "us-east-2", "us-west-1", "us-west-2",
    "eu-west-1", "eu-west-2", "eu-west-3", "eu-central-1",
    "ap-south-1", "ap-northeast-1", "ap-northeast-2",
    "ap-southeast-1", "ap-southeast-2",
    "ca-central-1", "sa-east-1"
]

# Instance Types by Category
INSTANCE_TYPES = {
    "general_purpose": ["t2.micro", "t2.small", "t3.micro", "t3.small", "t3.medium", "t3.large"],
    "compute_optimized": ["c5.large", "c5.xlarge", "c5.2xlarge"],
    "memory_optimized": ["r5.large", "r5.xlarge", "r5.2xlarge"],
    "storage_optimized": ["i3.large", "i3.xlarge", "d2.xlarge"]
}

# RDS Engines
RDS_ENGINES = {
    "postgres": ["12.15", "13.11", "14.8", "15.3"],
    "mysql": ["5.7.42", "8.0.33"],
    "mariadb": ["10.6.13", "10.11.4"],
    "oracle-ee": ["19.0.0.0"],
    "sqlserver-ex": ["15.00.4236.7.v1"]
}

# EBS Volume Types
EBS_VOLUME_TYPES = ["gp2", "gp3", "io1", "io2", "sc1", "st1"]
```

---
