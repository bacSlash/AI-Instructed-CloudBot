# Streamlit Implementation Guide

## CRITICAL: Understanding Streamlit Execution Model

### The Core Problem
Streamlit reruns the **entire script from top to bottom** on every user interaction. This means:
- Every button click reruns everything
- Every text input reruns everything  
- Variables are reset unless stored in `st.session_state`
- API calls will be made multiple times unless cached

### The Solution: Session State + Callbacks

**ALWAYS use `st.session_state` for ANY data that needs to persist between reruns.**

---

## Session State Initialization Pattern

**CRITICAL: Initialize session state FIRST, before any other code.**

```python
import streamlit as st
from typing import List, Dict, Any
from datetime import datetime

def initialize_session_state():
    """Initialize all session state variables - CALL THIS FIRST"""
    
    # Conversation history
    if 'messages' not in st.session_state:
        st.session_state.messages = []
    
    # Current conversation stage
    if 'conversation_stage' not in st.session_state:
        st.session_state.conversation_stage = 'init'
    
    # Identified resources
    if 'identified_resources' not in st.session_state:
        st.session_state.identified_resources = []
    
    # Collected parameters
    if 'collected_parameters' not in st.session_state:
        st.session_state.collected_parameters = {}
    
    # Current resource being configured
    if 'current_resource_index' not in st.session_state:
        st.session_state.current_resource_index = 0
    
    # Current parameter being collected
    if 'current_parameter_index' not in st.session_state:
        st.session_state.current_parameter_index = 0
    
    # Parameters for current resource
    if 'current_resource_parameters' not in st.session_state:
        st.session_state.current_resource_parameters = []
    
    # User's initial request
    if 'initial_request' not in st.session_state:
        st.session_state.initial_request = None
    
    # Generated terraform code
    if 'terraform_code' not in st.session_state:
        st.session_state.terraform_code = {}
    
    # Processing flag to prevent duplicate API calls
    if 'processing' not in st.session_state:
        st.session_state.processing = False
    
    # Error state
    if 'error_message' not in st.session_state:
        st.session_state.error_message = None

# MUST BE FIRST LINE IN app.py after imports
initialize_session_state()
```

---

## Chat Interface Implementation

### CORRECT Pattern: Using Callbacks

```python
def handle_user_input():
    """Callback function when user submits input - PREVENTS RERUN ISSUES"""
    
    user_input = st.session_state.user_input_widget
    
    # Don't process empty input
    if not user_input or not user_input.strip():
        return
    
    # Don't process if already processing
    if st.session_state.processing:
        return
    
    # Set processing flag
    st.session_state.processing = True
    
    # Add user message to history
    st.session_state.messages.append({
        'role': 'user',
        'content': user_input,
        'timestamp': datetime.now()
    })
    
    # Process based on conversation stage
    try:
        if st.session_state.conversation_stage == 'init':
            # This is the initial request
            st.session_state.initial_request = user_input
            process_initial_request(user_input)
            
        elif st.session_state.conversation_stage == 'parameter_collection':
            # This is a parameter value
            process_parameter_response(user_input)
        
        else:
            # Handle other stages
            process_generic_input(user_input)
    
    except Exception as e:
        st.session_state.error_message = str(e)
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"❌ Error: {str(e)}",
            'timestamp': datetime.now()
        })
    
    finally:
        # Clear processing flag
        st.session_state.processing = False
        
        # Clear the input widget
        st.session_state.user_input_widget = ""


def render_chat_interface():
    """Render the chat interface"""
    
    # Display chat messages
    for message in st.session_state.messages:
        with st.chat_message(message['role']):
            st.markdown(message['content'])
    
    # Chat input with callback - THE KEY PART
    st.chat_input(
        "Your message...",
        key="user_input_widget",
        on_submit=handle_user_input  # This prevents rerun issues
    )
```

### Why This Works:
1. **Callback runs BEFORE rerun** - `handle_user_input()` executes when user presses Enter
2. **Processing flag prevents duplicate calls** - Won't process same input twice
3. **Input is cleared automatically** - Widget state is managed properly
4. **Messages persist in session_state** - Conversation history maintained

---

## Processing Functions Pattern

### CRITICAL: Never call LLM directly in main script flow

```python
def process_initial_request(user_input: str):
    """Process the initial infrastructure request"""
    
    # Add processing message
    st.session_state.messages.append({
        'role': 'assistant',
        'content': '🔍 Analyzing your infrastructure request...',
        'timestamp': datetime.now()
    })
    
    # Call LLM to parse intent - ONLY ONCE
    from src.llm.groq_client import GroqLLMClient
    from src.orchestrator.intent_parser import parse_infrastructure_intent
    
    llm_client = GroqLLMClient()
    
    try:
        # Parse intent
        resources = parse_infrastructure_intent(llm_client, user_input)
        
        # Store in session state
        st.session_state.identified_resources = resources
        
        # Transition to dependency analysis
        st.session_state.conversation_stage = 'dependency_analysis'
        
        # Analyze dependencies
        from src.orchestrator.dependency_resolver import resolve_dependencies
        parameter_order = resolve_dependencies(resources)
        st.session_state.current_resource_parameters = parameter_order
        
        # Transition to parameter collection
        st.session_state.conversation_stage = 'parameter_collection'
        st.session_state.current_parameter_index = 0
        
        # Generate first question
        generate_next_question()
        
    except Exception as e:
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"❌ Error analyzing request: {str(e)}",
            'timestamp': datetime.now()
        })
        st.session_state.conversation_stage = 'error'


def generate_next_question():
    """Generate the next question to ask user"""
    
    # Check if we're done
    if st.session_state.current_parameter_index >= len(st.session_state.current_resource_parameters):
        # All parameters collected, generate code
        transition_to_generation()
        return
    
    # Get current parameter to collect
    param_info = st.session_state.current_resource_parameters[st.session_state.current_parameter_index]
    
    # Generate question using LLM
    from src.llm.groq_client import GroqLLMClient
    from src.orchestrator.question_generator import generate_parameter_question
    
    llm_client = GroqLLMClient()
    
    try:
        question_data = generate_parameter_question(
            llm_client,
            param_info['resource_type'],
            param_info['parameter_name'],
            st.session_state.collected_parameters
        )
        
        # Format question message
        question_message = f"""**{question_data['question']}**

ℹ️ {question_data['explanation']}

💡 **Suggested:** {question_data['suggested_value']}
   {question_data['suggestion_reasoning']}

**Examples:** {', '.join(question_data['examples'])}"""
        
        # Add to messages
        st.session_state.messages.append({
            'role': 'assistant',
            'content': question_message,
            'timestamp': datetime.now()
        })
        
    except Exception as e:
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"❌ Error generating question: {str(e)}",
            'timestamp': datetime.now()
        })


def process_parameter_response(user_input: str):
    """Process user's response to parameter question"""
    
    # Get current parameter info
    param_info = st.session_state.current_resource_parameters[st.session_state.current_parameter_index]
    
    # Validate the input
    from src.validation.schema_validator import validate_parameter
    
    validation_result = validate_parameter(
        param_info['parameter_name'],
        user_input,
        param_info['resource_type']
    )
    
    if validation_result['is_valid']:
        # Store the parameter
        param_key = f"{param_info['resource_type']}_{param_info['parameter_name']}"
        st.session_state.collected_parameters[param_key] = user_input
        
        # Confirmation message
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"✅ {param_info['parameter_name']} set to: {user_input}",
            'timestamp': datetime.now()
        })
        
        # Move to next parameter
        st.session_state.current_parameter_index += 1
        
        # Generate next question
        generate_next_question()
        
    else:
        # Show validation error
        error_msg = f"❌ Invalid value: {validation_result['error']}\n\n"
        error_msg += "Please try again with a valid value."
        
        st.session_state.messages.append({
            'role': 'assistant',
            'content': error_msg,
            'timestamp': datetime.now()
        })
        # Don't increment index - ask same question again


def transition_to_generation():
    """Transition to code generation stage"""
    
    st.session_state.conversation_stage = 'generation'
    
    st.session_state.messages.append({
        'role': 'assistant',
        'content': '✅ All parameters collected! Generating your Terraform code...',
        'timestamp': datetime.now()
    })
    
    # Generate Terraform code
    from src.terraform.generator import TerraformGenerator
    
    try:
        generator = TerraformGenerator()
        terraform_files = generator.generate_code(
            st.session_state.identified_resources,
            st.session_state.collected_parameters
        )
        
        st.session_state.terraform_code = terraform_files
        st.session_state.conversation_stage = 'complete'
        
        # Show completion message
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"""🎉 **Terraform code generated successfully!**

Generated files:
- main.tf ({len(terraform_files.get('main.tf', ''))} lines)
- variables.tf ({len(terraform_files.get('variables.tf', ''))} lines)
- outputs.tf ({len(terraform_files.get('outputs.tf', ''))} lines)

You can download the files using the button in the sidebar.""",
            'timestamp': datetime.now()
        })
        
    except Exception as e:
        st.session_state.messages.append({
            'role': 'assistant',
            'content': f"❌ Error generating code: {str(e)}",
            'timestamp': datetime.now()
        })
        st.session_state.conversation_stage = 'error'
```

---

## Main App Structure

### COMPLETE app.py Structure

```python
import streamlit as st
import os
from dotenv import load_dotenv

# Load environment variables FIRST
load_dotenv()

# Page config MUST BE FIRST STREAMLIT COMMAND
st.set_page_config(
    page_title="TerraConverse",
    page_icon="🚀",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Import custom modules
from src.ui.session_init import initialize_session_state
from src.ui.chat_interface import render_chat_interface
from src.ui.sidebar import render_sidebar

# Initialize session state - MUST BE EARLY
initialize_session_state()

# Main app
def main():
    """Main application entry point"""
    
    # Title
    st.title("🚀 TerraConverse")
    st.caption("AI-Powered Terraform Generator")
    
    # Create layout
    sidebar_col = st.sidebar
    main_col = st.container()
    
    # Render sidebar
    with sidebar_col:
        render_sidebar()
    
    # Render main chat interface
    with main_col:
        # Show error if any
        if st.session_state.error_message:
            st.error(st.session_state.error_message)
            if st.button("Clear Error"):
                st.session_state.error_message = None
                st.rerun()
        
        # Show welcome message if no messages yet
        if len(st.session_state.messages) == 0:
            show_welcome_message()
        
        # Render chat interface
        render_chat_interface()

def show_welcome_message():
    """Show welcome message on first load"""
    st.markdown("""
    ### Welcome to TerraConverse! 👋
    
    I'm your AI assistant for generating Terraform infrastructure code. 
    Just describe what you want to build, and I'll guide you through the configuration.
    
    **Example requests:**
    - "I need a web server with a PostgreSQL database"
    - "Set up a VPC with public and private subnets"
    - "Create an S3 bucket for static website hosting"
    
    **Start by describing your infrastructure needs below** 👇
    """)

if __name__ == "__main__":
    main()
```

---

## Critical Patterns to Follow

### 1. Never Call LLM in Main Script Flow

**WRONG ❌:**
```python
# This will call LLM on EVERY rerun
if st.button("Generate"):
    response = llm_client.generate(prompt)  # BAD!
    st.write(response)
```

**CORRECT ✅:**
```python
def handle_generate():
    """Callback function"""
    if 'response' not in st.session_state:
        st.session_state.response = llm_client.generate(prompt)

st.button("Generate", on_click=handle_generate)

if 'response' in st.session_state:
    st.write(st.session_state.response)
```

### 2. Use Keys for All Widgets

```python
# Always use unique keys
st.text_input("Enter value", key="param_value_widget")
st.button("Submit", key="submit_button")
st.selectbox("Choose", options, key="selector_widget")
```

### 3. Use st.rerun() Sparingly

```python
# Only use st.rerun() when you NEED to force a rerun
# Usually callbacks handle this automatically

def reset_conversation():
    """Reset button callback"""
    st.session_state.messages = []
    st.session_state.conversation_stage = 'init'
    st.session_state.collected_parameters = {}
    # Rerun needed here to show fresh state
    st.rerun()

st.button("Start Over", on_click=reset_conversation)
```

### 4. Prevent Duplicate Processing

```python
# Always use a processing flag
def process_data():
    if st.session_state.get('processing', False):
        return  # Already processing, skip
    
    st.session_state.processing = True
    
    try:
        # Do expensive operation
        result = expensive_operation()
        st.session_state.result = result
    finally:
        st.session_state.processing = False
```

---

## File Structure for Streamlit App

```
terraconverse/
├── app.py                          # Main entry point (uses patterns above)
├── .env                            # Environment variables
├── requirements.txt
│
└── src/
    ├── ui/
    │   ├── __init__.py
    │   ├── session_init.py         # initialize_session_state()
    │   ├── chat_interface.py       # render_chat_interface()
    │   ├── sidebar.py              # render_sidebar()
    │   └── callbacks.py            # All callback functions
    │
    ├── orchestrator/
    │   ├── __init__.py
    │   ├── intent_parser.py        # parse_infrastructure_intent()
    │   ├── dependency_resolver.py  # resolve_dependencies()
    │   └── question_generator.py   # generate_parameter_question()
    │
    ├── llm/
    │   ├── __init__.py
    │   └── groq_client.py          # GroqLLMClient class
    │
    ├── validation/
    │   ├── __init__.py
    │   └── schema_validator.py     # validate_parameter()
    │
    └── terraform/
        ├── __init__.py
        └── generator.py            # TerraformGenerator class
```

---

## Debugging Streamlit Issues

### Common Issues and Solutions

**Issue: App reruns infinitely**
- **Cause:** Widget value changes in main flow trigger reruns
- **Solution:** Use callbacks, don't set widget values in main flow

**Issue: LLM called multiple times**
- **Cause:** LLM call in main script flow
- **Solution:** Only call LLM in callbacks, cache with session state

**Issue: Messages duplicated**
- **Cause:** Appending to messages on every rerun
- **Solution:** Only append in callbacks, check if message already exists

**Issue: Input cleared but not processed**
- **Cause:** Processing happens after rerun
- **Solution:** Use on_submit callback for chat_input

**Issue: State lost between interactions**
- **Cause:** Not using st.session_state
- **Solution:** Store everything in session_state

### Debug Mode

```python
# Add debug sidebar
with st.sidebar.expander("🐛 Debug Info", expanded=False):
    st.write("**Session State:**")
    st.json({
        'stage': st.session_state.conversation_stage,
        'messages_count': len(st.session_state.messages),
        'parameters_collected': len(st.session_state.collected_parameters),
        'processing': st.session_state.processing,
        'current_param_index': st.session_state.current_parameter_index
    })
```

---

## LLM Integration Best Practices

### Singleton Pattern for LLM Client

```python
# src/llm/groq_client.py

import os
from groq import Groq

_llm_client_instance = None

def get_llm_client():
    """Get singleton LLM client instance"""
    global _llm_client_instance
    
    if _llm_client_instance is None:
        api_key = os.getenv('GROQ_API_KEY')
        if not api_key:
            raise ValueError("GROQ_API_KEY not set in environment")
        
        _llm_client_instance = Groq(api_key=api_key)
    
    return _llm_client_instance


def call_llm(messages, temperature=0.7, max_tokens=2048):
    """Call LLM with error handling"""
    client = get_llm_client()
    
    try:
        response = client.chat.completions.create(
            model="llama-3.1-70b-versatile",
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens
        )
        return response.choices[0].message.content
    
    except Exception as e:
        raise Exception(f"LLM API Error: {str(e)}")
```

### Caching LLM Responses

```python
import streamlit as st
import hashlib
import json

def get_cached_llm_response(prompt_key, messages):
    """Get cached LLM response or call API"""
    
    # Create cache key
    cache_key = f"llm_cache_{prompt_key}"
    
    # Check if in session state
    if cache_key in st.session_state:
        return st.session_state[cache_key]
    
    # Call LLM
    response = call_llm(messages)
    
    # Cache in session state
    st.session_state[cache_key] = response
    
    return response
```

---

## Testing Strategy

### Manual Testing Checklist

1. **Fresh Start**
   - [ ] App loads without errors
   - [ ] Welcome message displays
   - [ ] Input field is ready

2. **First Message**
   - [ ] Can type and submit initial request
   - [ ] LLM processes request (loading indicator?)
   - [ ] First question appears
   - [ ] No duplicate processing

3. **Second Message**
   - [ ] Can respond to first question
   - [ ] Validation works
   - [ ] Second question appears
   - [ ] No app crash
   - [ ] No infinite reruns

4. **Full Flow**
   - [ ] Complete all questions
   - [ ] Code generation works
   - [ ] Download available
   - [ ] Can start over

5. **Error Handling**
   - [ ] Invalid input shows error
   - [ ] Can retry after error
   - [ ] API errors handled gracefully

---

## Performance Optimization

### Lazy Loading

```python
# Don't import expensive modules at top level
# Import only when needed

def process_initial_request(user_input):
    # Import here, not at top of file
    from src.orchestrator.intent_parser import parse_infrastructure_intent
    from src.llm.groq_client import get_llm_client
    
    # Rest of function...
```

### Progress Indicators

```python
def long_running_operation():
    """Show progress for long operations"""
    
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    status_text.text("Step 1: Parsing intent...")
    # Do work
    progress_bar.progress(33)
    
    status_text.text("Step 2: Resolving dependencies...")
    # Do work
    progress_bar.progress(66)
    
    status_text.text("Step 3: Generating questions...")
    # Do work
    progress_bar.progress(100)
    
    status_text.empty()
    progress_bar.empty()
```

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Status**: CRITICAL FOR IMPLEMENTATION
