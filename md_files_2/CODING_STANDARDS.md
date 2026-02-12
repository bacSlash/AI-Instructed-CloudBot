# Coding Standards and Conventions

## Python Code Style

### General Guidelines
- Follow PEP 8
- Use type hints for all function signatures
- Maximum line length: 100 characters
- Use docstrings for all public functions and classes

### Example
```python
from typing import List, Dict, Optional
from pydantic import BaseModel

class ResourceManager:
    """Manages AWS resource definitions"""
    
    def create_resource(
        self,
        resource_type: str,
        parameters: Dict[str, Any]
    ) -> ResourceDefinition:
        """
        Create a new resource definition
        
        Args:
            resource_type: AWS resource type (e.g., 'ec2_instance')
            parameters: Configuration parameters
            
        Returns:
            ResourceDefinition instance
            
        Raises:
            ValidationError: If parameters are invalid
        """
        # Implementation
        pass
```

## Project Structure

```
terraconverse/
├── app.py                      # Streamlit main entry point
├── requirements.txt
├── .env.example
├── README.md
│
├── src/
│   ├── __init__.py
│   │
│   ├── ui/                     # Streamlit UI components
│   │   ├── __init__.py
│   │   ├── chat_interface.py
│   │   ├── sidebar.py
│   │   └── download_manager.py
│   │
│   ├── orchestrator/           # Conversation management
│   │   ├── __init__.py
│   │   ├── dialog_manager.py
│   │   ├── intent_parser.py
│   │   ├── question_sequencer.py
│   │   └── dependency_resolver.py
│   │
│   ├── llm/                    # LLM integration
│   │   ├── __init__.py
│   │   ├── groq_client.py
│   │   ├── prompt_templates.py
│   │   └── context_manager.py
│   │
│   ├── knowledge/              # AWS knowledge base
│   │   ├── __init__.py
│   │   ├── resource_schemas.py
│   │   ├── dependency_graph.py
│   │   └── service_catalog.py
│   │
│   ├── validation/             # Validation logic
│   │   ├── __init__.py
│   │   ├── schema_validator.py
│   │   ├── aws_validator.py
│   │   └── security_checker.py
│   │
│   ├── terraform/              # Terraform generation
│   │   ├── __init__.py
│   │   ├── generator.py
│   │   ├── formatter.py
│   │   └── templates/
│   │       ├── base/
│   │       ├── compute/
│   │       ├── network/
│   │       ├── database/
│   │       └── storage/
│   │
│   └── models/                 # Data models
│       ├── __init__.py
│       ├── message.py
│       ├── resource.py
│       ├── session.py
│       └── validation.py
│
└── tests/                      # Tests (future)
    ├── __init__.py
    └── test_*.py
```

## Naming Conventions

- **Classes**: PascalCase (e.g., `ResourceDefinition`)
- **Functions/Methods**: snake_case (e.g., `generate_question`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_RETRIES`)
- **Private methods**: Prefix with underscore (e.g., `_internal_method`)
- **Files**: snake_case (e.g., `dialog_manager.py`)

## Error Handling

```python
# Define custom exceptions
class TerraConverseError(Exception):
    """Base exception for TerraConverse"""
    pass

class ValidationError(TerraConverseError):
    """Validation failed"""
    pass

class LLMError(TerraConverseError):
    """LLM API error"""
    pass

# Use try-except blocks appropriately
try:
    result = validate_parameter(value)
except ValidationError as e:
    logger.error(f"Validation failed: {e}")
    display_error_message(str(e))
```

---