# API Specifications and Function Signatures

## Core Service Interfaces

### DialogManager

```python
class DialogManager:
    """Orchestrates conversation flow"""
    
    def start_conversation(self) -> None:
        """Initialize new conversation"""
        
    def process_user_input(self, user_input: str) -> Response:
        """Process user message and generate response"""
        
    def get_next_question(self) -> Question:
        """Determine and return next question to ask"""
        
    def is_complete(self) -> bool:
        """Check if all parameters collected"""
```

### LLMService

```python
class LLMService:
    """LLM integration service"""
    
    def parse_intent(self, user_request: str) -> List[ResourceIntent]:
        """Parse user's infrastructure request"""
        
    def generate_question(
        self,
        resource_type: str,
        parameter_name: str,
        context: Dict[str, Any]
    ) -> Question:
        """Generate contextual question"""
        
    def validate_response(
        self,
        question: str,
        response: str
    ) -> ValidationResult:
        """Validate user's response"""
```

### ValidationService

```python
class ValidationService:
    """Parameter validation service"""
    
    def validate_parameter(
        self,
        param_name: str,
        value: Any,
        resource_type: str
    ) -> ValidationResult:
        """Validate single parameter"""
        
    def validate_security(
        self,
        session: SessionState
    ) -> List[SecurityWarning]:
        """Run security checks"""
```

### TerraformGenerator

```python
class TerraformGenerator:
    """Terraform code generation"""
    
    def generate_code(
        self,
        session: SessionState
    ) -> Dict[str, str]:
        """Generate all Terraform files"""
        
    def format_hcl(self, code: str) -> str:
        """Format HCL code"""
        
    def validate_syntax(self, files: Dict[str, str]) -> ValidationResult:
        """Validate Terraform syntax"""
```

---