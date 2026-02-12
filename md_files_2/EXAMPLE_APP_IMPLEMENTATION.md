# Complete Working Example Implementation

## Full Working app.py

This is a COMPLETE, WORKING example that demonstrates the correct Streamlit patterns. GitHub Copilot should use this as the primary reference for the main application structure.

```python
"""
TerraConverse - AI-Powered Terraform Generator
Main application entry point

CRITICAL: This file demonstrates the CORRECT way to structure a Streamlit app
with LLM integration. Pay special attention to:
1. Session state initialization
2. Callback pattern for user input
3. Preventing duplicate LLM calls
4. State management between reruns
"""

import streamlit as st
import os
from dotenv import load_dotenv
from datetime import datetime
import json

# Load environment variables FIRST
load_dotenv()

# Page config MUST BE FIRST STREAMLIT COMMAND
st.set_page_config(
    page_title="TerraConverse",
    page_icon="🚀",
    layout="wide",
    initial_sidebar_state="expanded"
)

# ============================================================================
# SESSION STATE INITIALIZATION - CRITICAL SECTION
# ============================================================================

def initialize_session_state():
    """
    Initialize all session state variables.
    MUST be called early to prevent KeyError issues.
    """
    # Conversation history
    if 'messages' not in st.session_state:
        st.session_state.messages = []
    
    # Conversation stage
    if 'conversation_stage' not in st.session_state:
        st.session_state.conversation_stage = 'init'
    
    # User's initial infrastructure request
    if 'initial_request' not in st.session_state:
        st.session_state.initial_request = None
    
    # Identified AWS resources
    if 'identified_resources' not in st.session_state:
        st.session_state.identified_resources = []
    
    # Parameters to collect (ordered list)
    if 'parameters_to_collect' not in st.session_state:
        st.session_state.parameters_to_collect = []
    
    # Current parameter index
    if 'current_parameter_index' not in st.session_state:
        st.session_state.current_parameter_index = 0
    
    # Collected parameter values
    if 'collected_parameters' not in st.session_state:
        st.session_state.collected_parameters = {}
    
    # Generated Terraform code
    if 'terraform_code' not in st.session_state:
        st.session_state.terraform_code = {}
    
    # Processing flag to prevent duplicate API calls
    if 'processing' not in st.session_state:
        st.session_state.processing = False
    
    # Error handling
    if 'error_message' not in st.session_state:
        st.session_state.error_message = None
    
    # Debug mode
    if 'debug_mode' not in st.session_state:
        st.session_state.debug_mode = False

# Initialize session state IMMEDIATELY
initialize_session_state()

# ============================================================================
# HELPER FUNCTIONS
# ============================================================================

def add_message(role: str, content: str):
    """Add a message to the conversation history"""
    st.session_state.messages.append({
        'role': role,
        'content': content,
        'timestamp': datetime.now().isoformat()
    })

def get_llm_client():
    """Get Groq LLM client (singleton pattern)"""
    try:
        from groq import Groq
        api_key = os.getenv('GROQ_API_KEY')
        if not api_key:
            raise ValueError("GROQ_API_KEY environment variable not set")
        return Groq(api_key=api_key)
    except ImportError:
        raise ImportError("groq library not installed. Run: pip install groq")

def call_llm(messages, temperature=0.7, max_tokens=2048, json_mode=False):
    """Call LLM with error handling"""
    client = get_llm_client()
    
    params = {
        "model": "llama-3.1-70b-versatile",
        "messages": messages,
        "temperature": temperature,
        "max_tokens": max_tokens
    }
    
    if json_mode:
        params["response_format"] = {"type": "json_object"}
    
    try:
        response = client.chat.completions.create(**params)
        return response.choices[0].message.content
    except Exception as e:
        raise Exception(f"LLM API Error: {str(e)}")

# ============================================================================
# PROCESSING FUNCTIONS - Called from callbacks
# ============================================================================

def process_initial_request(user_input: str):
    """
    Process the user's initial infrastructure request.
    This function is called from the callback, so it runs BEFORE rerun.
    """
    try:
        add_message('assistant', '🔍 Analyzing your infrastructure request...')
        
        # Build LLM prompt for intent parsing
        system_prompt = """You are an AWS infrastructure expert. Parse the user's request and identify required AWS resources.

Output ONLY valid JSON with this structure:
{
  "resources": [
    {
      "resource_type": "ec2_instance",
      "category": "compute",
      "purpose": "web server"
    }
  ],
  "parameters": [
    {
      "resource_type": "vpc",
      "parameter_name": "cidr_block",
      "display_name": "VPC CIDR Block",
      "default_value": "10.0.0.0/16"
    }
  ]
}

Include:
1. Explicitly mentioned resources
2. Required dependencies (VPC, subnets, security groups)
3. Parameters needed for each resource in logical order"""

        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"User request: {user_input}"}
        ]
        
        # Call LLM to parse intent
        response = call_llm(messages, temperature=0.3, json_mode=True)
        
        # Parse JSON response
        intent_data = json.loads(response)
        
        # Store identified resources
        st.session_state.identified_resources = intent_data.get('resources', [])
        st.session_state.parameters_to_collect = intent_data.get('parameters', [])
        
        # Create summary message
        resource_list = [r['resource_type'] for r in st.session_state.identified_resources]
        summary = f"""✅ **Analysis complete!**

I'll help you configure the following AWS resources:
{chr(10).join([f'- {r}' for r in resource_list])}

Total parameters to collect: {len(st.session_state.parameters_to_collect)}

Let's start with the configuration questions."""
        
        add_message('assistant', summary)
        
        # Transition to parameter collection
        st.session_state.conversation_stage = 'parameter_collection'
        st.session_state.current_parameter_index = 0
        
        # Generate first question
        generate_next_question()
        
    except json.JSONDecodeError as e:
        error_msg = f"❌ Error parsing LLM response: {str(e)}"
        add_message('assistant', error_msg)
        st.session_state.error_message = error_msg
    except Exception as e:
        error_msg = f"❌ Error processing request: {str(e)}"
        add_message('assistant', error_msg)
        st.session_state.error_message = error_msg

def generate_next_question():
    """Generate the next parameter question"""
    
    # Check if all parameters collected
    if st.session_state.current_parameter_index >= len(st.session_state.parameters_to_collect):
        transition_to_generation()
        return
    
    # Get current parameter to collect
    param = st.session_state.parameters_to_collect[st.session_state.current_parameter_index]
    
    try:
        # Build prompt for question generation
        system_prompt = """Generate a clear question to collect this parameter. Output ONLY valid JSON:
{
  "question": "What ... ?",
  "explanation": "This determines ...",
  "suggested_value": "...",
  "suggestion_reasoning": "Recommended because ...",
  "examples": ["ex1", "ex2"]
}"""

        context = {
            "resource_type": param['resource_type'],
            "parameter_name": param['parameter_name'],
            "collected_so_far": st.session_state.collected_parameters
        }
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"""Generate question for:
Parameter: {param['parameter_name']} ({param['display_name']})
Resource: {param['resource_type']}
Context: {json.dumps(context)}"""}
        ]
        
        response = call_llm(messages, temperature=0.5, json_mode=True)
        question_data = json.loads(response)
        
        # Format question message
        question_msg = f"""**{question_data['question']}**

ℹ️  {question_data['explanation']}

💡 **Suggested:** `{question_data['suggested_value']}`  
{question_data['suggestion_reasoning']}

**Examples:** {', '.join(f'`{ex}`' for ex in question_data.get('examples', []))}"""
        
        add_message('assistant', question_msg)
        
    except Exception as e:
        # Fallback to simple question
        fallback_msg = f"""**Please provide: {param['display_name']}**

ℹ️  This value is needed for configuring {param['resource_type']}.

💡 **Suggested:** `{param.get('default_value', 'N/A')}`

Type your value below."""
        
        add_message('assistant', fallback_msg)

def process_parameter_response(user_input: str):
    """Process user's response to a parameter question"""
    
    param = st.session_state.parameters_to_collect[st.session_state.current_parameter_index]
    
    # Simple validation (could be enhanced)
    if not user_input or not user_input.strip():
        add_message('assistant', '❌ Please provide a valid value.')
        return
    
    # Store the parameter
    param_key = f"{param['resource_type']}_{param['parameter_name']}"
    st.session_state.collected_parameters[param_key] = user_input
    
    # Confirmation
    add_message('assistant', f"✅ **{param['display_name']}** set to: `{user_input}`")
    
    # Move to next parameter
    st.session_state.current_parameter_index += 1
    
    # Generate next question or finish
    generate_next_question()

def transition_to_generation():
    """Move to code generation stage"""
    
    st.session_state.conversation_stage = 'generation'
    
    add_message('assistant', '✅ All parameters collected! Generating your Terraform code...')
    
    try:
        # Simple code generation (would use templates in real implementation)
        terraform_code = generate_terraform_code_simple()
        
        st.session_state.terraform_code = terraform_code
        st.session_state.conversation_stage = 'complete'
        
        # Success message
        files_summary = '\n'.join([f"- {filename} ({len(content)} characters)" 
                                   for filename, content in terraform_code.items()])
        
        add_message('assistant', f"""🎉 **Terraform code generated successfully!**

**Generated files:**
{files_summary}

You can download the files using the buttons in the sidebar.

Would you like to start a new configuration?""")
        
    except Exception as e:
        error_msg = f"❌ Error generating code: {str(e)}"
        add_message('assistant', error_msg)
        st.session_state.error_message = error_msg

def generate_terraform_code_simple():
    """
    Simple Terraform code generation.
    In real implementation, this would use Jinja2 templates.
    """
    # Generate basic main.tf
    main_tf = """terraform {
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
}

# Resources will be generated based on collected parameters
"""
    
    # Add resource blocks based on identified resources
    for resource in st.session_state.identified_resources:
        main_tf += f"\n# {resource['resource_type']}\n"
        main_tf += "# Configuration here...\n\n"
    
    # Generate variables.tf
    variables_tf = """variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

"""
    
    for param in st.session_state.parameters_to_collect:
        param_name = param['parameter_name']
        variables_tf += f"""variable "{param_name}" {{
  description = "{param['display_name']}"
  type        = string
  default     = "{param.get('default_value', '')}"
}}

"""
    
    # Generate outputs.tf
    outputs_tf = """# Outputs will be defined based on created resources
"""
    
    return {
        'main.tf': main_tf,
        'variables.tf': variables_tf,
        'outputs.tf': outputs_tf
    }

# ============================================================================
# CALLBACK FUNCTIONS - Handle user interactions
# ============================================================================

def handle_user_input():
    """
    Callback function for chat input.
    This is called BEFORE the rerun, so we can update session state safely.
    """
    # Get user input from widget
    user_input = st.session_state.user_input_widget
    
    # Ignore empty input
    if not user_input or not user_input.strip():
        return
    
    # Prevent duplicate processing
    if st.session_state.processing:
        return
    
    # Set processing flag
    st.session_state.processing = True
    
    # Add user message
    add_message('user', user_input)
    
    try:
        # Route based on conversation stage
        if st.session_state.conversation_stage == 'init':
            # First message - process as infrastructure request
            st.session_state.initial_request = user_input
            process_initial_request(user_input)
            
        elif st.session_state.conversation_stage == 'parameter_collection':
            # Collecting parameters
            process_parameter_response(user_input)
            
        elif st.session_state.conversation_stage == 'complete':
            # Conversation complete, might be asking to start over
            if 'new' in user_input.lower() or 'start' in user_input.lower():
                reset_conversation()
                add_message('assistant', '🔄 Starting fresh! What infrastructure would you like to build?')
            else:
                add_message('assistant', 'The configuration is complete. Use the download buttons in the sidebar, or type "start new" to begin again.')
        
        else:
            add_message('assistant', 'Processing...')
    
    except Exception as e:
        error_msg = f"❌ Unexpected error: {str(e)}"
        add_message('assistant', error_msg)
        st.session_state.error_message = error_msg
    
    finally:
        # Clear processing flag
        st.session_state.processing = False

def reset_conversation():
    """Reset conversation to initial state"""
    st.session_state.messages = []
    st.session_state.conversation_stage = 'init'
    st.session_state.initial_request = None
    st.session_state.identified_resources = []
    st.session_state.parameters_to_collect = []
    st.session_state.current_parameter_index = 0
    st.session_state.collected_parameters = {}
    st.session_state.terraform_code = {}
    st.session_state.error_message = None

# ============================================================================
# UI RENDERING
# ============================================================================

def render_sidebar():
    """Render the sidebar with progress and controls"""
    
    st.sidebar.title("📊 Session Info")
    
    # Status
    status_emoji = {
        'init': '⚪',
        'parameter_collection': '🔵',
        'generation': '🟡',
        'complete': '🟢',
        'error': '🔴'
    }
    
    st.sidebar.write(f"**Status:** {status_emoji.get(st.session_state.conversation_stage, '⚪')} {st.session_state.conversation_stage}")
    
    # Progress
    if st.session_state.parameters_to_collect:
        total = len(st.session_state.parameters_to_collect)
        current = st.session_state.current_parameter_index
        progress = min(current / total, 1.0) if total > 0 else 0
        
        st.sidebar.progress(progress)
        st.sidebar.write(f"Parameters: {current}/{total}")
    
    # Collected parameters
    if st.session_state.collected_parameters:
        with st.sidebar.expander("📋 Collected Parameters", expanded=False):
            for key, value in st.session_state.collected_parameters.items():
                st.write(f"**{key}:** {value}")
    
    # Download buttons
    if st.session_state.terraform_code:
        st.sidebar.divider()
        st.sidebar.subheader("📥 Download Files")
        
        for filename, content in st.session_state.terraform_code.items():
            st.sidebar.download_button(
                label=f"Download {filename}",
                data=content,
                file_name=filename,
                mime="text/plain",
                key=f"download_{filename}"
            )
    
    # Reset button
    st.sidebar.divider()
    if st.sidebar.button("🔄 Start New Configuration", use_container_width=True):
        reset_conversation()
        st.rerun()
    
    # Debug mode
    st.sidebar.divider()
    st.session_state.debug_mode = st.sidebar.checkbox("🐛 Debug Mode", value=st.session_state.debug_mode)
    
    if st.session_state.debug_mode:
        with st.sidebar.expander("Debug Info", expanded=True):
            st.json({
                'stage': st.session_state.conversation_stage,
                'messages': len(st.session_state.messages),
                'resources': len(st.session_state.identified_resources),
                'params_total': len(st.session_state.parameters_to_collect),
                'params_collected': len(st.session_state.collected_parameters),
                'processing': st.session_state.processing
            })

def render_welcome():
    """Render welcome message"""
    st.markdown("""
    ### Welcome to TerraConverse! 👋
    
    I'm your AI assistant for generating production-ready Terraform infrastructure code.
    
    **How it works:**
    1. Describe your infrastructure needs in plain English
    2. I'll ask you configuration questions
    3. Download your complete Terraform code
    
    **Example requests:**
    - "I need a web server with a PostgreSQL database"
    - "Set up a VPC with public and private subnets"  
    - "Create an S3 bucket for static website hosting"
    
    **Start by typing your infrastructure request below** 👇
    """)

def render_chat():
    """Render the chat interface"""
    
    # Display all messages
    for message in st.session_state.messages:
        with st.chat_message(message['role']):
            st.markdown(message['content'])
    
    # Chat input with callback
    st.chat_input(
        "Describe your infrastructure or answer the question...",
        key="user_input_widget",
        on_submit=handle_user_input
    )

# ============================================================================
# MAIN APPLICATION
# ============================================================================

def main():
    """Main application entry point"""
    
    # Title
    st.title("🚀 TerraConverse")
    st.caption("AI-Powered Terraform Infrastructure Generator")
    
    # Check for API key
    if not os.getenv('GROQ_API_KEY'):
        st.error("⚠️ GROQ_API_KEY environment variable not set. Please add it to your .env file.")
        st.stop()
    
    # Render sidebar
    render_sidebar()
    
    # Show error if any
    if st.session_state.error_message:
        st.error(st.session_state.error_message)
        if st.button("Clear Error"):
            st.session_state.error_message = None
            st.rerun()
    
    # Show welcome or chat
    if len(st.session_state.messages) == 0:
        render_welcome()
    
    # Always render chat interface
    render_chat()

if __name__ == "__main__":
    main()
```

## Key Takeaways for GitHub Copilot

When generating code based on the MD files, follow these critical patterns:

1. **Session State First**: Initialize ALL session state at the very beginning
2. **Callbacks Only**: Never process user input in main flow, always use callbacks
3. **Processing Flag**: Use a flag to prevent duplicate API calls
4. **Error Handling**: Wrap all LLM calls in try-except
5. **Simple First**: Start with working simple version, then enhance
6. **Test After Each Stage**: Make sure conversation works through multiple messages

## Testing This Implementation

1. Create `.env` file with `GROQ_API_KEY=your_key`
2. Run `streamlit run app.py`
3. Type: "I need a web server"
4. Verify you can send multiple messages without crashes
5. Check that LLM is only called once per user input
6. Confirm session state persists between interactions

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Status**: REFERENCE IMPLEMENTATION - USE THIS AS PRIMARY EXAMPLE
