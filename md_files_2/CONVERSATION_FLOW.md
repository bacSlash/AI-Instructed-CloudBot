# Conversation Flow and Dialog Management

## Overview

This document defines the conversation logic, question sequencing, and LLM prompt strategies for TerraConverse. The conversation flow is designed to feel natural while efficiently collecting all necessary parameters.

## Conversation Stages

### Stage 1: Initialization (INIT)

**Purpose**: Welcome user and collect initial infrastructure description

**System Behavior**:
- Display welcome message with example requests
- Wait for user's initial infrastructure description
- No LLM calls yet

**User Input**: Free-form description of desired infrastructure

**Example**:
```
User: "I need a web server with a PostgreSQL database"
```

**Transition**: → INTENT_PARSING

---

### Stage 2: Intent Parsing (INTENT_PARSING)

**Purpose**: Analyze user request to identify required AWS resources

**System Behavior**:
1. Send user description to LLM with intent parsing prompt
2. Extract list of required AWS resources
3. Identify user's purpose for each resource
4. Calculate confidence scores

**LLM Prompt Template**:
```python
INTENT_PARSING_PROMPT = """You are an expert AWS cloud architect. A user has described their infrastructure needs. Your task is to identify which AWS resources are required.

User Request: "{user_request}"

Analyze this request and identify ALL AWS resources needed, including dependencies. Return a JSON array of resources with this structure:

[
  {{
    "resource_type": "ec2_instance",
    "category": "compute",
    "purpose": "web server",
    "confidence": 0.95,
    "mentioned_explicitly": true
  }},
  ...
]

Consider:
1. Explicit resources (mentioned directly by user)
2. Implicit resources (required dependencies like VPC, subnets, security groups)
3. Best practice resources (recommended but optional like CloudWatch, backups)

Be comprehensive but don't over-engineer. For a basic request, stick to essentials.
"""
```

**Expected Output**:
```json
[
  {
    "resource_type": "ec2_instance",
    "category": "compute",
    "purpose": "web server",
    "confidence": 0.95,
    "mentioned_explicitly": true
  },
  {
    "resource_type": "rds_instance",
    "category": "database",
    "purpose": "postgresql database",
    "confidence": 0.98,
    "mentioned_explicitly": true
  },
  {
    "resource_type": "vpc",
    "category": "network",
    "purpose": "network isolation",
    "confidence": 1.0,
    "mentioned_explicitly": false
  },
  {
    "resource_type": "subnet",
    "category": "network",
    "purpose": "network segmentation",
    "confidence": 1.0,
    "mentioned_explicitly": false
  },
  {
    "resource_type": "security_group",
    "category": "security",
    "purpose": "firewall rules",
    "confidence": 1.0,
    "mentioned_explicitly": false
  }
]
```

**Validation**:
- Verify all required fields present
- Check confidence scores are reasonable (0.0-1.0)
- Validate resource types against known AWS services

**Transition**: → DEPENDENCY_ANALYSIS

---

### Stage 3: Dependency Analysis (DEPENDENCY_ANALYSIS)

**Purpose**: Build dependency graph and determine optimal question order

**System Behavior**:
1. Load dependency graph for each identified resource
2. Build directed acyclic graph (DAG) of dependencies
3. Identify shared resources (e.g., single VPC for multiple services)
4. Perform topological sort to determine collection order
5. Group related parameters together

**Dependency Resolution Algorithm**:
```python
def resolve_dependencies(resources: List[ResourceIntent]) -> List[str]:
    """
    Build dependency graph and return collection order
    
    Returns: Ordered list of resource IDs to configure
    """
    graph = defaultdict(list)
    in_degree = defaultdict(int)
    all_resources = set()
    
    # Add all resources and their dependencies
    for resource in resources:
        all_resources.add(resource.resource_type)
        dependencies = DEPENDENCY_GRAPH[resource.resource_type]
        
        for dep in dependencies['required']:
            graph[dep].append(resource.resource_type)
            in_degree[resource.resource_type] += 1
            all_resources.add(dep)
    
    # Topological sort (Kahn's algorithm)
    queue = [r for r in all_resources if in_degree[r] == 0]
    result = []
    
    while queue:
        current = queue.pop(0)
        result.append(current)
        
        for neighbor in graph[current]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    
    return result
```

**Example Output**:
```
Collection Order:
1. vpc (no dependencies)
2. subnet (depends on vpc)
3. security_group (depends on vpc)
4. ec2_instance (depends on vpc, subnet, security_group)
5. db_subnet_group (depends on vpc, subnet)
6. rds_instance (depends on vpc, db_subnet_group, security_group)
```

**Shared Resource Detection**:
```python
# If both EC2 and RDS identified, share the VPC
if 'ec2_instance' in resources and 'rds_instance' in resources:
    # Create one VPC, used by both
    shared_vpc = True
    # Create separate security groups for each
```

**Transition**: → PARAMETER_COLLECTION

---

### Stage 4: Parameter Collection (PARAMETER_COLLECTION)

**Purpose**: Collect configuration parameters for each resource in order

**System Behavior**:
Main loop that iterates through resources in dependency order, collecting parameters for each.

**For Each Resource**:
1. Load parameter schema
2. Identify required vs optional parameters
3. For each parameter:
   - Generate contextual question
   - Present to user
   - Validate response
   - Store if valid, re-ask if invalid
4. Mark resource as complete

**Question Generation Process**:

```python
def generate_question(
    resource_type: str,
    parameter_name: str,
    context: dict
) -> str:
    """
    Generate contextual question for parameter
    
    Args:
        resource_type: Type of AWS resource
        parameter_name: Parameter to collect
        context: Previously collected parameters and resource info
    
    Returns:
        Formatted question with explanation and suggestions
    """
    schema = get_parameter_schema(resource_type, parameter_name)
    
    # Build LLM prompt
    prompt = f"""Generate a clear, concise question to collect the following parameter:

Resource: {resource_type}
Parameter: {parameter_name}
Description: {schema.description}
Type: {schema.type}
Required: {schema.required}

Context:
{json.dumps(context, indent=2)}

Generate a question that:
1. Is clear and specific
2. Explains WHY this parameter matters (1 sentence)
3. Suggests a good default value with reasoning
4. Provides an example of valid input
5. References previously collected parameters when relevant

Format:
{{
  "question": "What ... ?",
  "explanation": "This determines ...",
  "suggested_value": "...",
  "suggestion_reasoning": "This is recommended because ...",
  "example": "Example: ..."
}}
"""
    
    response = llm_client.generate(prompt)
    return format_question(response)
```

**Question Formatting**:
```
Main Question: What AWS region should I deploy to?

ℹ️  Explanation: The region determines where your resources will be 
   physically located and affects latency, compliance, and pricing.

💡 Suggested: us-east-1 (N. Virginia)
   This is recommended because it has the most AWS services available 
   and lowest latency for US East Coast users.

Example: us-east-1, us-west-2, eu-west-1, ap-southeast-1
```

**Parameter Collection Loop**:
```python
async def collect_parameters(resource_def: ResourceDefinition):
    """Collect all parameters for a resource"""
    
    # Required parameters first
    for param_name in resource_def.required_parameters:
        while param_name not in resource_def.parameters:
            # Generate question
            question = await generate_question(
                resource_def.resource_type,
                param_name,
                get_context()
            )
            
            # Present to user
            await display_message(question)
            
            # Get user response
            user_response = await get_user_input()
            
            # Validate
            validation = validate_parameter(
                param_name,
                user_response,
                resource_def.resource_type
            )
            
            if validation.is_valid:
                resource_def.mark_parameter_collected(param_name, user_response)
                await display_message(f"✅ {param_name} set to {user_response}")
            else:
                await display_error(validation.errors)
                # Loop continues, will ask again
    
    # Optional parameters (ask if recommended)
    for param_name in resource_def.optional_parameters:
        if is_recommended(param_name):
            # Similar flow but allow skip
            pass
```

**Validation During Collection**:
```python
def validate_parameter(
    param_name: str,
    value: Any,
    resource_type: str
) -> ValidationResult:
    """
    Multi-layer validation
    """
    schema = get_parameter_schema(resource_type, param_name)
    
    # Layer 1: Type and format validation
    result = validate_format(value, schema)
    if not result.is_valid:
        return result
    
    # Layer 2: Constraint validation
    result = validate_constraints(value, schema)
    if not result.is_valid:
        return result
    
    # Layer 3: AWS API validation (if applicable)
    if schema.aws_constraint:
        result = validate_with_aws_api(value, schema)
        if not result.is_valid:
            return result
    
    # Layer 4: Security validation
    security_result = validate_security(value, schema)
    if security_result.has_warnings:
        result.warnings.extend(security_result.warnings)
    
    return result
```

**Progress Tracking**:
```python
def update_progress():
    """Update session progress indicators"""
    total_params = sum(
        len(r.required_parameters) 
        for r in session.resource_definitions.values()
    )
    collected_params = sum(
        len(r.parameters)
        for r in session.resource_definitions.values()
    )
    
    session.total_parameters = total_params
    session.collected_parameters = collected_params
    session.completion_percentage = (collected_params / total_params * 100)
```

**Transition**: → VALIDATION

---

### Stage 5: Validation (VALIDATION)

**Purpose**: Final validation of all collected parameters

**System Behavior**:
1. Validate completeness (all required params collected)
2. Cross-resource validation (e.g., subnet CIDR fits in VPC CIDR)
3. Security validation (comprehensive check)
4. Generate warnings and recommendations

**Validation Checks**:

```python
def final_validation(session: SessionState) -> ValidationReport:
    """Comprehensive validation before generation"""
    
    report = ValidationReport()
    
    # 1. Completeness check
    for resource in session.resource_definitions.values():
        if not resource.is_complete:
            report.add_error(
                f"Resource {resource.resource_name} incomplete: "
                f"missing {resource.missing_parameters}"
            )
    
    # 2. Cross-resource validation
    if 'vpc' in resources and 'subnet' in resources:
        vpc_cidr = resources['vpc'].parameters['cidr_block']
        for subnet in get_subnets():
            subnet_cidr = subnet.parameters['cidr_block']
            if not is_subnet_of(subnet_cidr, vpc_cidr):
                report.add_error(
                    f"Subnet CIDR {subnet_cidr} not within VPC CIDR {vpc_cidr}"
                )
    
    # 3. Security validation
    security_issues = comprehensive_security_check(session)
    report.security_warnings.extend(security_issues)
    
    # 4. Best practice recommendations
    recommendations = generate_recommendations(session)
    report.recommendations.extend(recommendations)
    
    return report
```

**Security Validation**:
```python
def comprehensive_security_check(session: SessionState) -> List[SecurityWarning]:
    """Check all security best practices"""
    
    warnings = []
    
    # Check security groups
    for sg in get_security_groups():
        for rule in sg.ingress_rules:
            if rule['cidr'] == '0.0.0.0/0':
                if rule['port'] == 22:
                    warnings.append(SecurityWarning(
                        severity='CRITICAL',
                        message="SSH (port 22) open to the internet",
                        remediation="Restrict to specific IP or use Systems Manager"
                    ))
                elif rule['port'] == 3389:
                    warnings.append(SecurityWarning(
                        severity='CRITICAL',
                        message="RDP (port 3389) open to the internet",
                        remediation="Restrict to specific IP or use bastion host"
                    ))
    
    # Check RDS
    for rds in get_rds_instances():
        if rds.parameters.get('publicly_accessible'):
            warnings.append(SecurityWarning(
                severity='CRITICAL',
                message="RDS instance is publicly accessible",
                remediation="Set publicly_accessible = false"
            ))
        
        if not rds.parameters.get('storage_encrypted'):
            warnings.append(SecurityWarning(
                severity='HIGH',
                message="RDS storage not encrypted",
                remediation="Enable storage encryption with KMS"
            ))
    
    # Check S3
    for s3 in get_s3_buckets():
        if not s3.parameters.get('block_public_access'):
            warnings.append(SecurityWarning(
                severity='CRITICAL',
                message="S3 bucket public access not blocked",
                remediation="Enable all block public access settings"
            ))
    
    return warnings
```

**Display Validation Results**:
```
✅ Validation Complete

📊 Summary:
- 6 resources configured
- 18 parameters collected
- 0 errors
- 2 warnings

⚠️  Security Warnings:

HIGH: RDS storage not encrypted
  Recommendation: Enable storage encryption with KMS for data protection
  [Auto-fix available]

MEDIUM: CloudWatch monitoring not configured for EC2
  Recommendation: Enable detailed monitoring for better observability
  [Add to configuration]

Would you like to:
[Apply Auto-fixes]  [Continue Anyway]  [Review Parameters]
```

**Transition**: → GENERATION (if validated) or → PARAMETER_COLLECTION (if errors)

---

### Stage 6: Generation (GENERATION)

**Purpose**: Generate Terraform HCL code from collected parameters

**System Behavior**:
1. Organize resources by dependency order
2. Generate HCL for each resource using templates
3. Generate variables and outputs
4. Format and validate generated code
5. Package files for download

**Generation Process**:
```python
async def generate_terraform_code(session: SessionState) -> Dict[str, str]:
    """Generate all Terraform files"""
    
    files = {}
    
    # 1. Generate main.tf
    main_tf = generate_main_tf(session)
    files['main.tf'] = main_tf
    
    # 2. Generate variables.tf
    variables_tf = generate_variables_tf(session)
    files['variables.tf'] = variables_tf
    
    # 3. Generate outputs.tf
    outputs_tf = generate_outputs_tf(session)
    files['outputs.tf'] = outputs_tf
    
    # 4. Generate terraform.tfvars.example
    tfvars_example = generate_tfvars_example(session)
    files['terraform.tfvars.example'] = tfvars_example
    
    # 5. Generate README.md
    readme = generate_readme(session)
    files['README.md'] = readme
    
    # 6. Format all files
    for filename, content in files.items():
        if filename.endswith('.tf'):
            files[filename] = terraform_fmt(content)
    
    # 7. Validate syntax
    validation_result = validate_terraform_syntax(files)
    if not validation_result.is_valid:
        raise GenerationError(validation_result.errors)
    
    return files
```

**Transition**: → COMPLETE

---

### Stage 7: Complete (COMPLETE)

**Purpose**: Present generated code to user

**System Behavior**:
1. Display success message
2. Show code preview
3. Provide download options
4. Display deployment instructions
5. Offer to start new configuration

**Display Format**:
```
✅ Code Generation Complete!

I've generated your Terraform configuration with:
- 1 VPC (10.0.0.0/16)
- 2 Subnets (public & private)
- 2 Security Groups
- 1 EC2 Instance (t3.medium)
- 1 RDS Instance (PostgreSQL 14.7, db.t3.micro)

Files ready for download:
📄 main.tf (156 lines)
📄 variables.tf (24 lines)
📄 outputs.tf (12 lines)
📄 terraform.tfvars.example (18 lines)
📄 README.md (deployment instructions)

[Download All (.zip)]  [Preview Code]

---

Next steps:
1. Download the files
2. Review the configuration
3. Run: terraform init
4. Run: terraform plan
5. Run: terraform apply

⚠️  Important: Review security warnings in README.md before deploying!

[Start New Configuration]  [Download Again]
```

---

## LLM Prompt Engineering

### System Prompt Template

```python
SYSTEM_PROMPT = """You are an expert AWS cloud architect and Terraform specialist. You help users define and configure AWS infrastructure through natural conversation.

Your responsibilities:
1. Understand user infrastructure needs from plain English descriptions
2. Ask clear, contextual questions to collect necessary parameters
3. Explain technical concepts in accessible language
4. Suggest sensible defaults with reasoning
5. Warn about security and cost implications
6. Guide users toward AWS best practices

Guidelines:
- Be conversational but professional
- Ask one question at a time
- Always explain WHY information is needed
- Provide examples and suggestions
- Be security-conscious
- Don't assume user's technical level
- Reference previously collected information for context

Current conversation context:
{context}
"""
```

### Few-Shot Examples

```python
FEW_SHOT_EXAMPLES = [
    {
        "user": "I need a web server",
        "assistant": {
            "intent": ["ec2_instance", "vpc", "subnet", "security_group"],
            "question": "Great! I'll help you set up a web server. First, what AWS region would you like to deploy to? This determines where your server will be physically located.\n\n💡 Suggested: us-east-1 (N. Virginia) - Most services available, good for US East Coast.\n\nExample: us-east-1, us-west-2, eu-west-1"
        }
    },
    {
        "user": "10.0.0.0/16",
        "assistant": {
            "validation": "valid",
            "response": "✅ VPC CIDR set to 10.0.0.0/16 (65,536 IP addresses available)\n\nNow, let's configure a public subnet for your web server. What CIDR block should I use?\n\nℹ️ The subnet should be a subset of your VPC CIDR and will host resources accessible from the internet.\n\n💡 Suggested: 10.0.1.0/24 (256 IP addresses)\n\nExample: 10.0.1.0/24"
        }
    }
]
```

### Function Calling Schema

```python
FUNCTION_SCHEMA = {
    "name": "collect_parameter",
    "description": "Collect and validate a configuration parameter",
    "parameters": {
        "type": "object",
        "properties": {
            "parameter_name": {
                "type": "string",
                "description": "Name of the parameter to collect"
            },
            "parameter_value": {
                "type": "string",
                "description": "User-provided value"
            },
            "resource_type": {
                "type": "string",
                "description": "AWS resource type"
            }
        },
        "required": ["parameter_name", "parameter_value", "resource_type"]
    }
}
```

---

## Error Handling

### User Input Errors

```python
def handle_user_error(error_type: str, context: dict):
    """Generate helpful error message"""
    
    messages = {
        "invalid_cidr": """
        ❌ That doesn't look like a valid CIDR block.
        
        CIDR format: X.X.X.X/Y
        Example: 10.0.0.0/16
        
        The number after / is the prefix length (16-28 for VPC).
        
        Would you like to use the suggested default of {suggested}?
        """,
        
        "instance_type_unavailable": """
        ⚠️  The instance type '{value}' may not be available in {region}.
        
        Common instance types:
        - t3.micro (2 vCPU, 1 GB RAM) - Good for testing
        - t3.medium (2 vCPU, 4 GB RAM) - Good for small apps
        - t3.large (2 vCPU, 8 GB RAM) - Good for medium apps
        
        What instance type would you like to use?
        """,
        
        "security_group_too_permissive": """
        ⚠️  SECURITY WARNING
        
        You're about to allow SSH (port 22) from the entire internet (0.0.0.0/0).
        This is a security risk as it exposes your server to brute force attacks.
        
        Recommendations:
        1. Restrict to your IP: {user_ip}/32
        2. Use AWS Systems Manager (no SSH needed)
        3. Use a bastion host
        
        How would you like to proceed?
        [Use My IP] [Systems Manager] [Continue Anyway]
        """
    }
    
    return messages[error_type].format(**context)
```

### LLM Errors

```python
def handle_llm_error(error: Exception):
    """Fallback when LLM unavailable"""
    
    # Use template-based questions instead
    fallback_questions = load_template_questions()
    
    # Continue with reduced functionality
    return {
        "mode": "template",
        "questions": fallback_questions
    }
```

---