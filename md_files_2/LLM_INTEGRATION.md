# LLM Integration Guide

## Overview

This document specifies the integration with Groq API for Llama 3.1 70B, including prompt engineering, function calling, and error handling strategies.

## Groq API Configuration

### API Client Setup

```python
from groq import Groq
import os
from typing import List, Dict, Any, Optional
import json

class GroqLLMClient:
    """Groq API client for Llama 3.1 70B"""
    
    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key or os.getenv('GROQ_API_KEY')
        if not self.api_key:
            raise ValueError("GROQ_API_KEY environment variable not set")
        
        self.client = Groq(api_key=self.api_key)
        self.model = "llama-3.1-70b-versatile"
        self.max_tokens = 2048
        self.temperature = 0.7
    
    def generate(
        self,
        messages: List[Dict[str, str]],
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None,
        json_mode: bool = False
    ) -> str:
        """Generate completion from messages"""
        
        params = {
            "model": self.model,
            "messages": messages,
            "temperature": temperature or self.temperature,
            "max_tokens": max_tokens or self.max_tokens,
        }
        
        if json_mode:
            params["response_format"] = {"type": "json_object"}
        
        try:
            response = self.client.chat.completions.create(**params)
            return response.choices[0].message.content
        
        except Exception as e:
            raise LLMError(f"Groq API error: {str(e)}")
```

### Environment Variables

```bash
# .env file
GROQ_API_KEY=your_groq_api_key_here
AWS_REGION=us-east-1  # Optional default
```

## Prompt Engineering

### System Prompt

```python
SYSTEM_PROMPT = """You are an expert AWS cloud architect and Terraform specialist helping users configure cloud infrastructure through conversation.

Your Role:
- Parse user infrastructure requests to identify required AWS resources
- Ask clear, contextual questions to collect configuration parameters
- Explain technical concepts in accessible language
- Suggest sensible defaults with reasoning
- Warn about security risks and cost implications
- Guide users toward AWS best practices

Guidelines:
1. Ask ONE question at a time
2. Always explain WHY information is needed (1-2 sentences)
3. Provide a suggested default value with reasoning
4. Give examples of valid inputs
5. Reference previously collected parameters for context
6. Be conversational but professional
7. Prioritize security and best practices

Response Format for Questions:
{{
  "question": "Clear, specific question?",
  "explanation": "Why this matters...",
  "suggested_value": "recommended value",
  "suggestion_reasoning": "Why this is recommended...",
  "examples": ["example1", "example2"]
}}

Current State:
- Conversation Stage: {stage}
- Collected Parameters: {collected_count}
- Remaining Parameters: {remaining_count}
"""
```

### Intent Parsing Prompt

```python
def build_intent_parsing_prompt(user_request: str) -> List[Dict[str, str]]:
    """Build messages for intent parsing"""
    
    return [
        {
            "role": "system",
            "content": """You are an AWS infrastructure expert. Parse user requests to identify required AWS resources.

Output valid JSON only, no additional text. Format:
{
  "resources": [
    {
      "resource_type": "ec2_instance",
      "category": "compute",
      "purpose": "web server",
      "confidence": 0.95,
      "mentioned_explicitly": true,
      "priority": 1
    }
  ]
}

Include:
1. Explicitly mentioned resources
2. Required dependencies (VPC, subnets, security groups, etc.)
3. Recommended resources for the use case

Resource Types: ec2_instance, rds_instance, vpc, subnet, security_group, s3_bucket, lambda_function, ecs_cluster, alb, etc."""
        },
        {
            "role": "user",
            "content": f"User request: {user_request}\n\nIdentify all required AWS resources."
        }
    ]
```

### Question Generation Prompt

```python
def build_question_prompt(
    resource_type: str,
    parameter_name: str,
    parameter_schema: ParameterSchema,
    context: Dict[str, Any]
) -> List[Dict[str, str]]:
    """Build messages for question generation"""
    
    context_summary = json.dumps({
        "resource_type": resource_type,
        "collected_parameters": {k: v for k, v in context.items() if v is not None},
        "infrastructure_purpose": context.get("purpose", "general infrastructure")
    }, indent=2)
    
    return [
        {
            "role": "system",
            "content": SYSTEM_PROMPT.format(
                stage="parameter_collection",
                collected_count=len(context),
                remaining_count=context.get("remaining", 0)
            )
        },
        {
            "role": "user",
            "content": f"""Generate a question to collect this parameter:

Resource: {resource_type}
Parameter: {parameter_name}
Type: {parameter_schema.type}
Required: {parameter_schema.required}
Description: {parameter_schema.description}
Explanation: {parameter_schema.explanation}

Current Context:
{context_summary}

Generate a clear question following the response format specified in your system prompt. Output JSON only."""
        }
    ]
```

## Function Calling

### Function Definitions

```python
FUNCTION_DEFINITIONS = [
    {
        "name": "parse_infrastructure_intent",
        "description": "Parse user's infrastructure request to identify required AWS resources",
        "parameters": {
            "type": "object",
            "properties": {
                "resources": {
                    "type": "array",
                    "description": "List of identified AWS resources",
                    "items": {
                        "type": "object",
                        "properties": {
                            "resource_type": {"type": "string"},
                            "category": {"type": "string"},
                            "purpose": {"type": "string"},
                            "confidence": {"type": "number"},
                            "mentioned_explicitly": {"type": "boolean"},
                            "priority": {"type": "integer"}
                        },
                        "required": ["resource_type", "category", "purpose"]
                    }
                }
            },
            "required": ["resources"]
        }
    },
    {
        "name": "generate_parameter_question",
        "description": "Generate a contextual question to collect a parameter",
        "parameters": {
            "type": "object",
            "properties": {
                "question": {"type": "string"},
                "explanation": {"type": "string"},
                "suggested_value": {"type": "string"},
                "suggestion_reasoning": {"type": "string"},
                "examples": {
                    "type": "array",
                    "items": {"type": "string"}
                }
            },
            "required": ["question", "explanation", "suggested_value"]
        }
    },
    {
        "name": "validate_parameter_value",
        "description": "Validate a user-provided parameter value",
        "parameters": {
            "type": "object",
            "properties": {
                "is_valid": {"type": "boolean"},
                "validation_message": {"type": "string"},
                "suggested_correction": {"type": "string"}
            },
            "required": ["is_valid"]
        }
    }
]
```

### Using Function Calling

```python
class GroqLLMClient:
    
    def call_function(
        self,
        messages: List[Dict[str, str]],
        functions: List[Dict],
        function_call: str = "auto"
    ) -> Dict[str, Any]:
        """Call LLM with function calling"""
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=[{"type": "function", "function": f} for f in functions],
            tool_choice=function_call,
            temperature=0.3  # Lower temperature for structured outputs
        )
        
        message = response.choices[0].message
        
        if message.tool_calls:
            tool_call = message.tool_calls[0]
            function_name = tool_call.function.name
            function_args = json.loads(tool_call.function.arguments)
            
            return {
                "function_name": function_name,
                "arguments": function_args
            }
        
        # Fallback to regular response
        return {
            "function_name": None,
            "content": message.content
        }
```

## Error Handling

### Retry Logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import time

class GroqAPIError(Exception):
    pass

class RateLimitError(GroqAPIError):
    pass

class GroqLLMClient:
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        retry=retry_if_exception_type(RateLimitError)
    )
    def generate_with_retry(self, messages: List[Dict[str, str]], **kwargs) -> str:
        """Generate with automatic retry on rate limits"""
        
        try:
            return self.generate(messages, **kwargs)
        
        except Exception as e:
            error_msg = str(e).lower()
            
            if "rate limit" in error_msg or "429" in error_msg:
                raise RateLimitError(f"Rate limit exceeded: {e}")
            elif "timeout" in error_msg:
                raise GroqAPIError(f"Request timeout: {e}")
            else:
                raise GroqAPIError(f"API error: {e}")
```

### Fallback Strategies

```python
class LLMService:
    """High-level LLM service with fallbacks"""
    
    def __init__(self):
        self.groq_client = GroqLLMClient()
        self.template_fallback = TemplateFallback()
    
    def generate_question(
        self,
        resource_type: str,
        parameter_name: str,
        context: dict
    ) -> Dict[str, str]:
        """Generate question with fallback to templates"""
        
        try:
            # Try LLM first
            messages = build_question_prompt(
                resource_type,
                parameter_name,
                get_parameter_schema(resource_type, parameter_name),
                context
            )
            
            response = self.groq_client.generate_with_retry(
                messages,
                json_mode=True
            )
            
            return json.loads(response)
        
        except Exception as e:
            logger.warning(f"LLM failed, using template fallback: {e}")
            
            # Fallback to pre-written templates
            return self.template_fallback.get_question(
                resource_type,
                parameter_name,
                context
            )
```

### Template Fallback

```python
class TemplateFallback:
    """Pre-written question templates as fallback"""
    
    QUESTION_TEMPLATES = {
        ("ec2_instance", "instance_type"): {
            "question": "What EC2 instance type would you like to use?",
            "explanation": "The instance type determines your server's CPU, memory, and network capacity.",
            "suggested_value": "t3.medium",
            "suggestion_reasoning": "Good balance of performance and cost for most web applications.",
            "examples": ["t3.micro", "t3.medium", "t3.large", "m5.large"]
        },
        ("vpc", "cidr_block"): {
            "question": "What CIDR block should I use for your VPC?",
            "explanation": "This defines the private IP address range for your entire network.",
            "suggested_value": "10.0.0.0/16",
            "suggestion_reasoning": "Provides 65,536 IP addresses, suitable for most deployments.",
            "examples": ["10.0.0.0/16", "172.16.0.0/16", "192.168.0.0/16"]
        },
        # ... more templates
    }
    
    def get_question(
        self,
        resource_type: str,
        parameter_name: str,
        context: dict
    ) -> Dict[str, str]:
        """Get question from template"""
        
        key = (resource_type, parameter_name)
        
        if key in self.QUESTION_TEMPLATES:
            return self.QUESTION_TEMPLATES[key]
        
        # Generic fallback
        schema = get_parameter_schema(resource_type, parameter_name)
        return {
            "question": f"Please provide the {parameter_name} for {resource_type}:",
            "explanation": schema.description,
            "suggested_value": schema.default_value or "",
            "suggestion_reasoning": "Default value from schema",
            "examples": schema.example_values
        }
```

## Response Parsing

### JSON Mode

```python
def parse_llm_json_response(response: str) -> Dict:
    """Parse LLM JSON response with error handling"""
    
    try:
        # Direct JSON parse
        return json.loads(response)
    
    except json.JSONDecodeError:
        # Try to extract JSON from markdown code blocks
        import re
        
        json_match = re.search(r'```json\s*(\{.*?\})\s*```', response, re.DOTALL)
        if json_match:
            return json.loads(json_match.group(1))
        
        # Try to find any JSON object
        json_match = re.search(r'\{.*\}', response, re.DOTALL)
        if json_match:
            return json.loads(json_match.group(0))
        
        raise ValueError(f"Could not parse JSON from response: {response[:100]}")
```

## Context Management

### Conversation History

```python
class ConversationContext:
    """Manage conversation context and history"""
    
    def __init__(self, max_history: int = 10):
        self.messages: List[Dict[str, str]] = []
        self.max_history = max_history
    
    def add_message(self, role: str, content: str):
        """Add message to history"""
        self.messages.append({"role": role, "content": content})
        
        # Keep only recent messages to stay within token limits
        if len(self.messages) > self.max_history:
            # Keep system message + recent messages
            self.messages = [self.messages[0]] + self.messages[-(self.max_history-1):]
    
    def get_messages(self) -> List[Dict[str, str]]:
        """Get messages for API call"""
        return self.messages.copy()
    
    def get_summary(self) -> str:
        """Get conversation summary"""
        return f"{len(self.messages)} messages in history"
```

### Token Counting

```python
def count_tokens(text: str) -> int:
    """Estimate token count (rough approximation)"""
    # Llama models use ~1.3 tokens per word on average
    words = len(text.split())
    return int(words * 1.3)

def truncate_context(
    messages: List[Dict[str, str]],
    max_tokens: int = 6000
) -> List[Dict[str, str]]:
    """Truncate messages to fit within token limit"""
    
    # Always keep system message
    system_msg = messages[0] if messages and messages[0]["role"] == "system" else None
    other_msgs = messages[1:] if system_msg else messages
    
    # Count tokens and remove oldest messages if needed
    total_tokens = sum(count_tokens(m["content"]) for m in messages)
    
    while total_tokens > max_tokens and len(other_msgs) > 1:
        removed = other_msgs.pop(0)
        total_tokens -= count_tokens(removed["content"])
    
    return ([system_msg] if system_msg else []) + other_msgs
```

## Monitoring and Logging

```python
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class GroqLLMClient:
    
    def generate(self, messages: List[Dict[str, str]], **kwargs) -> str:
        """Generate with logging"""
        
        start_time = datetime.now()
        
        logger.info(f"LLM request: {len(messages)} messages, {sum(count_tokens(m['content']) for m in messages)} tokens")
        
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                **kwargs
            )
            
            duration = (datetime.now() - start_time).total_seconds()
            
            logger.info(f"LLM response received in {duration:.2f}s")
            
            return response.choices[0].message.content
        
        except Exception as e:
            duration = (datetime.now() - start_time).total_seconds()
            logger.error(f"LLM error after {duration:.2f}s: {e}")
            raise
```

---