# System Architecture

## Architecture Overview

TerraConverse follows a modular, layered architecture designed for maintainability, extensibility, and clear separation of concerns.

```
┌─────────────────────────────────────────────────────────────┐
│                     Streamlit UI Layer                      │
│  (Chat Interface, Parameter Display, Download Controls)     │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                 Conversation Orchestrator                   │
│     (Session Management, Dialog Flow, State Machine)        │
└─────┬─────────────────┬─────────────────┬──────────────────┘
      │                 │                 │
┌─────▼─────┐  ┌────────▼────────┐  ┌────▼──────────────────┐
│    LLM    │  │   AWS Resource  │  │  Terraform Generator  │
│  Service  │  │  Knowledge Base │  │      Engine           │
└─────┬─────┘  └────────┬────────┘  └────┬──────────────────┘
      │                 │                 │
┌─────▼─────────────────▼─────────────────▼──────────────────┐
│              Parameter Collection & Validation              │
│        (Schema Validators, AWS Constraints Checker)         │
└─────────────────────────────────────────────────────────────┘
```

## System Components

### 1. Streamlit UI Layer
**Responsibility**: User interaction and presentation

**Components**:
- `ui/chat_interface.py`: Main chat component using st.chat_message
- `ui/sidebar.py`: Parameter summary, progress indicators, settings
- `ui/download_manager.py`: Terraform file download functionality
- `ui/state_manager.py`: Streamlit session state wrapper

**Key Features**:
- Real-time chat interface with message history
- Live parameter summary panel
- Progress tracking for parameter collection
- Terraform code preview before download
- Security warnings display

**Session State Management**:

CRITICAL: Streamlit reruns the entire script on every interaction. ALL data must be stored in `st.session_state` to persist between reruns.

```python
# Initialize in a dedicated function called at app start
def initialize_session_state():
    """MUST be called before any other Streamlit code"""
    if 'messages' not in st.session_state:
        st.session_state.messages = []
    if 'collected_parameters' not in st.session_state:
        st.session_state.collected_parameters = {}
    if 'conversation_stage' not in st.session_state:
        st.session_state.conversation_stage = 'init'
    if 'identified_resources' not in st.session_state:
        st.session_state.identified_resources = []
    if 'current_parameter_index' not in st.session_state:
        st.session_state.current_parameter_index = 0
    if 'processing' not in st.session_state:
        st.session_state.processing = False
    if 'terraform_code' not in st.session_state:
        st.session_state.terraform_code = {}
    if 'error_message' not in st.session_state:
        st.session_state.error_message = None

# Session State Structure:
st.session_state = {
    'messages': [],                    # Chat history
    'collected_parameters': {},        # Gathered parameters
    'conversation_stage': 'init',      # Current conversation phase
    'identified_resources': [],        # Parsed AWS resources
    'current_parameter_index': 0,      # Which parameter we're collecting
    'processing': False,               # Prevent duplicate API calls
    'terraform_code': {},              # Generated files
    'error_message': None              # Current error state
}
```

**Critical Pattern: Use Callbacks, Not Direct Processing**
```python
# WRONG - This calls LLM on every rerun ❌
user_input = st.chat_input("Message")
if user_input:
    response = llm.generate(user_input)  # Called on EVERY rerun!

# CORRECT - Use callbacks ✅
def handle_user_input():
    if st.session_state.processing:
        return  # Prevent duplicate calls
    st.session_state.processing = True
    user_input = st.session_state.user_input_widget
    response = llm.generate(user_input)  # Called ONCE
    st.session_state.messages.append({'role': 'assistant', 'content': response})
    st.session_state.processing = False

st.chat_input("Message", key="user_input_widget", on_submit=handle_user_input)
```

### 2. Conversation Orchestrator
**Responsibility**: Dialog management and conversation flow control

**Components**:
- `orchestrator/dialog_manager.py`: Main conversation controller
- `orchestrator/intent_parser.py`: Parse user intent from initial request
- `orchestrator/question_sequencer.py`: Determine next question to ask
- `orchestrator/dependency_resolver.py`: Manage resource dependency chains
- `orchestrator/completion_checker.py`: Verify all required params collected

**Core Logic Flow**:
```
1. User provides initial infrastructure description
2. Intent Parser identifies required AWS resources
3. Dependency Resolver builds resource dependency graph
4. Question Sequencer determines optimal question order
5. For each required parameter:
   a. Generate contextual question via LLM
   b. Validate user response
   c. Store validated parameter
   d. Check if dependencies triggered
6. Completion Checker verifies all params collected
7. Trigger Terraform generation
```

**State Machine**:
```
INIT → INTENT_PARSING → DEPENDENCY_ANALYSIS → PARAMETER_COLLECTION → 
VALIDATION → GENERATION → COMPLETE
```

### 3. LLM Service
**Responsibility**: Natural language understanding and generation

**Components**:
- `llm/groq_client.py`: Groq API client wrapper
- `llm/prompt_templates.py`: Structured prompts for different scenarios
- `llm/response_parser.py`: Extract structured data from LLM responses
- `llm/context_manager.py`: Manage conversation context window

**Integration Pattern**:
```python
class GroqLLMService:
    def parse_infrastructure_intent(self, user_description: str) -> List[ResourceIntent]
    def generate_question(self, resource_type: str, parameter: str, context: dict) -> str
    def validate_response(self, question: str, response: str) -> ValidationResult
    def suggest_default_value(self, resource_type: str, parameter: str) -> str
```

**Prompt Strategy**:
- System prompts define AI role as AWS infrastructure expert
- Few-shot examples for consistent output format
- Function calling for structured parameter extraction
- Context includes previously collected parameters

### 4. AWS Resource Knowledge Base
**Responsibility**: AWS service definitions, dependencies, and constraints

**Components**:
- `knowledge/resource_schemas.py`: Pydantic models for each AWS resource
- `knowledge/dependency_graph.py`: Resource dependency relationships
- `knowledge/service_catalog.py`: AWS service metadata and descriptions
- `knowledge/parameter_constraints.py`: AWS-specific validation rules

**Schema Structure**:
```python
class EC2InstanceSchema(BaseModel):
    # Required parameters
    instance_type: str  # t2.micro, t3.medium, etc.
    ami_id: str
    
    # Optional parameters
    key_name: Optional[str]
    subnet_id: Optional[str]
    security_group_ids: Optional[List[str]]
    user_data: Optional[str]
    
    # Dependencies
    depends_on: List[str] = ['vpc', 'subnet', 'security_group']
    
    # Metadata
    category: str = 'compute'
    description: str = 'EC2 virtual machine instance'
```

**Dependency Graph Example**:
```python
DEPENDENCY_GRAPH = {
    'ec2_instance': {
        'required': ['vpc', 'subnet'],
        'recommended': ['security_group', 'key_pair'],
        'optional': ['elastic_ip', 'iam_role']
    },
    'rds_instance': {
        'required': ['vpc', 'db_subnet_group'],
        'recommended': ['security_group', 'parameter_group'],
        'optional': ['kms_key', 'iam_role']
    }
}
```

### 5. Parameter Collection & Validation
**Responsibility**: Validate and store configuration parameters

**Components**:
- `validation/schema_validator.py`: Validate against Pydantic schemas
- `validation/aws_validator.py`: AWS API-based validation (boto3)
- `validation/security_checker.py`: Security best practice validation
- `validation/constraint_checker.py`: Check AWS service limits and constraints

**Validation Layers**:
1. **Type Validation**: Ensure correct data types (string, int, bool, list)
2. **Format Validation**: Regex patterns for ARNs, CIDRs, naming conventions
3. **Constraint Validation**: Min/max values, allowed values, string lengths
4. **AWS API Validation**: Check region availability, AMI existence, instance type availability
5. **Security Validation**: Warn about public access, unencrypted storage, overly permissive policies

**Example Validators**:
```python
def validate_cidr_block(cidr: str) -> ValidationResult:
    """Validate CIDR notation for VPC/subnet"""
    
def validate_instance_type(instance_type: str, region: str) -> ValidationResult:
    """Check if instance type available in region"""
    
def validate_security_group_rule(rule: dict) -> SecurityValidationResult:
    """Check for overly permissive security rules"""
```

### 6. Terraform Generator Engine
**Responsibility**: Generate HCL code from collected parameters

**Components**:
- `terraform/generator.py`: Main HCL generation orchestrator
- `terraform/templates/`: Jinja2 templates for each resource type
- `terraform/formatter.py`: HCL formatting and organization
- `terraform/file_organizer.py`: Split code into appropriate .tf files

**Generation Pipeline**:
```
1. Organize resources by type and dependencies
2. Generate resource blocks from templates
3. Generate variable definitions for parameterized values
4. Generate output blocks for important resource attributes
5. Format HCL code (proper indentation, spacing)
6. Organize into separate files (main.tf, variables.tf, outputs.tf)
7. Validate generated code syntax
```

**Template Structure** (Jinja2):
```hcl
# templates/ec2_instance.tf.j2
resource "aws_instance" "{{ resource_name }}" {
  ami           = var.{{ resource_name }}_ami_id
  instance_type = var.{{ resource_name }}_instance_type
  
  {% if subnet_id %}
  subnet_id = aws_subnet.{{ subnet_name }}.id
  {% endif %}
  
  {% if security_group_ids %}
  vpc_security_group_ids = [
    {% for sg in security_group_ids %}
    aws_security_group.{{ sg }}.id{{ "," if not loop.last else "" }}
    {% endfor %}
  ]
  {% endif %}
  
  tags = {
    Name = "{{ resource_name }}"
    ManagedBy = "TerraConverse"
  }
}
```

## Data Flow

### End-to-End Flow

```
1. User Input (UI)
   ↓
2. Intent Parsing (LLM Service)
   → Identify: "EC2 instance" + "RDS database"
   ↓
3. Dependency Analysis (Knowledge Base)
   → EC2 needs: VPC, Subnet, Security Group
   → RDS needs: VPC, DB Subnet Group, Security Group
   ↓
4. Question Sequence Planning (Orchestrator)
   → Order: VPC → Subnet → Security Group → EC2 → DB Subnet Group → RDS
   ↓
5. Parameter Collection Loop (UI + LLM + Validation)
   For each resource:
     a. Generate contextual question (LLM)
     b. Display to user with defaults (UI)
     c. Collect response (UI)
     d. Validate response (Validation)
     e. Store parameter (Session State)
   ↓
6. Completion Check (Orchestrator)
   → Verify all required params for all resources
   ↓
7. Terraform Generation (Generator)
   → Apply templates
   → Format HCL
   → Organize files
   ↓
8. Download (UI)
   → Present files to user
```

### Parameter Flow Example

```
User: "I need a web server with a database"

Intent Parser Output:
{
    "resources": [
        {"type": "ec2_instance", "purpose": "web_server"},
        {"type": "rds_instance", "purpose": "database"}
    ]
}

Dependency Resolver Output:
{
    "execution_order": [
        "vpc",
        "subnet_public",
        "subnet_private", 
        "internet_gateway",
        "route_table",
        "security_group_web",
        "security_group_db",
        "ec2_instance",
        "db_subnet_group",
        "rds_instance"
    ]
}

Question Sequence:
1. "What AWS region should I deploy to? (default: us-east-1)"
2. "What CIDR block for your VPC? (default: 10.0.0.0/16)"
3. "What CIDR for public subnet? (default: 10.0.1.0/24)"
... and so on
```

## Component Interactions

### LLM Service ↔ Orchestrator
- Orchestrator calls LLM to parse intent and generate questions
- LLM returns structured responses (JSON/function calls)
- Orchestrator manages context and conversation history

### Orchestrator ↔ Knowledge Base
- Orchestrator queries dependency graph for resource relationships
- Retrieves parameter schemas for validation
- Gets default values and descriptions

### Validation ↔ AWS API (boto3)
- Real-time checks for resource availability
- Region-specific validation
- Quota and limit verification

### Generator ↔ Knowledge Base
- Retrieves resource schemas for template selection
- Gets proper Terraform resource names and syntax
- Accesses parameter metadata for variable generation

## Error Handling Strategy

### LLM Service Errors
- Retry with exponential backoff for API failures
- Fallback to template-based questions if LLM unavailable
- Validate LLM outputs against expected schema

### Validation Errors
- Clear error messages with remediation suggestions
- Allow user to retry with different values
- Skip optional parameters if validation fails repeatedly

### Generation Errors
- Validate Terraform syntax before presenting to user
- Log generation errors for debugging
- Provide partial output if some resources fail

## Security Considerations

### API Key Management
- Groq API key stored in environment variables
- No hardcoded credentials
- Secure session state management

### User Input Sanitization
- Validate all user inputs against schemas
- Prevent injection attacks in generated Terraform
- Sanitize resource names and tags

### Generated Code Security
- Automatic security checks on generated Terraform
- Warnings for public access configurations
- Encryption enabled by default where applicable

## Performance Optimization

### Caching Strategy
- Cache AWS API responses (AMI lists, instance types)
- Cache LLM responses for common questions
- Session-level caching for dependency graphs

### Async Operations
- Parallel validation for independent parameters
- Async LLM calls where possible
- Background Terraform formatting

### Resource Limits
- Limit conversation history to last 20 messages
- Truncate long user inputs
- Implement request timeouts

## Deployment Architecture

```
┌──────────────────────────────────────┐
│         Streamlit Container          │
│  ┌────────────────────────────────┐  │
│  │     TerraConverse App          │  │
│  │  (streamlit run app.py)        │  │
│  └────────────┬───────────────────┘  │
│               │                      │
│  ┌────────────▼───────────────────┐  │
│  │   Environment Variables        │  │
│  │   - GROQ_API_KEY              │  │
│  │   - AWS_REGION (optional)     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
           │
           │ HTTPS
           ▼
┌──────────────────────────────────────┐
│        Groq API (External)           │
│    (Llama 3.1 70B Inference)         │
└──────────────────────────────────────┘
```

## Scalability Considerations

### Current Design (Single User)
- Streamlit session-based architecture
- In-memory state management
- Single-threaded processing

### Future Scaling (Multi-User)
- External session store (Redis)
- Message queue for async processing
- Database for conversation persistence
- Load balancing across instances

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Status**: Active Development
