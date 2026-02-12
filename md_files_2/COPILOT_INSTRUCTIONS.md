# Instructions for GitHub Copilot

## READ THIS FIRST

You have been provided with 16 comprehensive documentation files for building TerraConverse, a conversational AI that generates Terraform infrastructure code. This document explains how to use them effectively.

---

## 🚨 CRITICAL FILES - READ THESE FIRST

Before generating ANY code, you MUST read these files in this exact order:

### 1. EXAMPLE_APP_IMPLEMENTATION.md ⭐⭐⭐
**THIS IS THE MOST IMPORTANT FILE**

Contains a **complete, working app.py** that demonstrates:
- ✅ Correct Streamlit session state pattern
- ✅ Proper callback usage to prevent app crashes
- ✅ How to prevent duplicate LLM calls
- ✅ Full conversation flow implementation
- ✅ Error handling

**Use this file as your PRIMARY reference for app.py structure.**

### 2. STREAMLIT_IMPLEMENTATION.md ⭐⭐⭐
**SECOND MOST IMPORTANT**

Explains WHY the patterns in EXAMPLE_APP_IMPLEMENTATION work:
- Why Streamlit reruns cause issues
- Session state initialization patterns
- Callback vs direct processing
- Common mistakes and how to avoid them
- Debugging strategies

**Read this to UNDERSTAND the Streamlit-specific challenges.**

### 3. PROJECT_OVERVIEW.md ⭐⭐
High-level understanding of what you're building:
- Project scope and goals
- Technology stack
- Success criteria
- What's in scope vs out of scope

---

## 📚 Reference Files - Read as Needed

### Architecture & Design
- **ARCHITECTURE.md**: System components, data flow, deployment
- **WIREFRAMES.md**: UI layouts and user flows
- **FEATURES.md**: Detailed feature specifications

### Data & Models
- **DATA_MODELS.md**: Pydantic schemas, session state structure
- **AWS_RESOURCE_KNOWLEDGE_BASE.md**: AWS service definitions and dependencies

### Implementation Details
- **CONVERSATION_FLOW.md**: Dialog management, question sequencing, LLM prompts
- **TERRAFORM_GENERATION.md**: Jinja2 templates, HCL generation
- **LLM_INTEGRATION.md**: Groq API integration, prompt engineering
- **VALIDATION_SECURITY.md**: Validation rules, security checks

### Standards & Setup
- **CODING_STANDARDS.md**: Python style guide, project structure
- **API_SPECIFICATIONS.md**: Function signatures
- **DEPENDENCIES.md**: Requirements and installation
- **IMPLEMENTATION_GUIDE.md**: Development roadmap

---

## 🎯 Code Generation Strategy

### Step 1: Start with app.py
**Copy the complete app.py from EXAMPLE_APP_IMPLEMENTATION.md**

This gives you a working foundation. DO NOT modify the:
- Session state initialization pattern
- Callback structure for `handle_user_input()`
- Processing flag logic

### Step 2: Create Supporting Modules (in order)
Follow this exact order to avoid dependency issues:

1. **src/ui/session_init.py**
   - Extract `initialize_session_state()` from app.py
   
2. **src/llm/groq_client.py**
   - Use patterns from LLM_INTEGRATION.md
   - Implement singleton `get_llm_client()`
   - Implement `call_llm()` with error handling

3. **src/models/** (all Pydantic models)
   - Use DATA_MODELS.md as reference
   - Create: message.py, resource.py, session.py, validation.py

4. **src/knowledge/resource_schemas.py**
   - Use AWS_RESOURCE_KNOWLEDGE_BASE.md
   - Define schemas for EC2, VPC, RDS, etc.

5. **src/knowledge/dependency_graph.py**
   - Use AWS_RESOURCE_KNOWLEDGE_BASE.md
   - Implement dependency relationships

6. **src/orchestrator/** (in this order)
   - intent_parser.py: Parse user requests → AWS resources
   - dependency_resolver.py: Build parameter collection order
   - question_generator.py: Generate contextual questions

7. **src/validation/schema_validator.py**
   - Use VALIDATION_SECURITY.md
   - Implement parameter validation

8. **src/terraform/generator.py**
   - Use TERRAFORM_GENERATION.md
   - Implement HCL generation with Jinja2

9. **src/terraform/templates/**
   - Copy templates from TERRAFORM_GENERATION.md
   - Start with: vpc.tf.j2, ec2_instance.tf.j2, rds_instance.tf.j2

10. **src/ui/** (enhance the basics)
    - chat_interface.py: Extract from app.py
    - sidebar.py: Extract from app.py
    - callbacks.py: Extract callback functions

### Step 3: Integrate and Test
After each module:
1. Import it into app.py
2. Test that the app still runs
3. Test conversation flow
4. Fix any issues before moving to next module

---

## ⚠️ CRITICAL PATTERNS TO FOLLOW

### 1. Session State Pattern (from STREAMLIT_IMPLEMENTATION.md)

```python
# ✅ ALWAYS DO THIS
def initialize_session_state():
    if 'messages' not in st.session_state:
        st.session_state.messages = []
    # ... all other state variables

initialize_session_state()  # Call immediately after imports
```

### 2. Callback Pattern (from EXAMPLE_APP_IMPLEMENTATION.md)

```python
# ✅ CORRECT - Use callbacks
def handle_user_input():
    user_input = st.session_state.user_input_widget
    # Process input here
    st.session_state.messages.append({'role': 'user', 'content': user_input})

st.chat_input("Message", key="user_input_widget", on_submit=handle_user_input)

# ❌ WRONG - This causes crashes
user_input = st.chat_input("Message")
if user_input:  # This reruns on every interaction!
    # Process input
```

### 3. Processing Flag Pattern

```python
# ✅ ALWAYS check processing flag
def handle_user_input():
    if st.session_state.processing:
        return  # Prevent duplicate calls
    
    st.session_state.processing = True
    try:
        # Call LLM and process
        pass
    finally:
        st.session_state.processing = False
```

### 4. LLM Call Pattern

```python
# ✅ CORRECT - Call LLM only in callbacks
def process_initial_request(user_input):
    response = call_llm(messages)  # Only called once
    st.session_state.result = response

# ❌ WRONG - Don't call LLM in main flow
if some_condition:
    response = call_llm(messages)  # Called on every rerun!
```

---

## 🐛 Common Issues and Solutions

### Issue: "App stops working after 2nd message"
**Cause**: LLM being called in main flow instead of callback  
**Solution**: Move ALL LLM calls to callback functions  
**Reference**: STREAMLIT_IMPLEMENTATION.md, section "Never Call LLM in Main Script Flow"

### Issue: "Messages duplicated"
**Cause**: Appending messages on every rerun  
**Solution**: Only append in callbacks, check if message exists  
**Reference**: EXAMPLE_APP_IMPLEMENTATION.md, `handle_user_input()` function

### Issue: "Session state KeyError"
**Cause**: Accessing session state before initialization  
**Solution**: Call `initialize_session_state()` FIRST  
**Reference**: STREAMLIT_IMPLEMENTATION.md, "Session State Initialization Pattern"

### Issue: "LLM called multiple times"
**Cause**: No processing flag or caching  
**Solution**: Use processing flag and session state caching  
**Reference**: STREAMLIT_IMPLEMENTATION.md, "Prevent Duplicate Processing"

### Issue: "Input cleared but not processed"
**Cause**: Not using on_submit callback  
**Solution**: Use `st.chat_input(on_submit=callback)`  
**Reference**: EXAMPLE_APP_IMPLEMENTATION.md, `render_chat()` function

---

## 📋 Checklist Before Generating Code

- [ ] I've read EXAMPLE_APP_IMPLEMENTATION.md completely
- [ ] I understand the Streamlit rerun model from STREAMLIT_IMPLEMENTATION.md
- [ ] I know which files contain the information I need
- [ ] I will use callbacks for ALL user input processing
- [ ] I will initialize session state FIRST
- [ ] I will use a processing flag to prevent duplicate LLM calls
- [ ] I will test after each module is added
- [ ] I will start simple (basic flow) then enhance

---

## 🎯 Minimum Viable Product (MVP)

For the first working version, implement ONLY:

1. ✅ Basic app.py (from EXAMPLE_APP_IMPLEMENTATION.md)
2. ✅ Simple intent parsing (identify EC2 + VPC only)
3. ✅ Basic parameter collection (5 parameters max)
4. ✅ Simple validation (format checking only)
5. ✅ Basic Terraform generation (one template)

**Don't implement initially:**
- ❌ All AWS services (start with EC2 + VPC)
- ❌ Complex validation (AWS API calls)
- ❌ Security scanning
- ❌ Advanced error recovery
- ❌ Fancy UI features

**Get the conversation working first, then enhance.**

---

## 🚀 Quick Start Commands

```bash
# 1. Create project structure
mkdir -p terraconverse/src/{ui,orchestrator,llm,knowledge,validation,terraform/templates,models}
cd terraconverse

# 2. Create files
touch app.py
touch src/__init__.py
touch src/ui/__init__.py
# ... etc for all directories

# 3. Copy EXAMPLE_APP_IMPLEMENTATION.md content to app.py

# 4. Create .env
echo "GROQ_API_KEY=your_key_here" > .env

# 5. Install dependencies (from DEPENDENCIES.md)
pip install streamlit groq pydantic jinja2 tenacity python-dotenv

# 6. Run
streamlit run app.py
```

---

## 📞 When You Need Help

If something doesn't work:

1. **Check EXAMPLE_APP_IMPLEMENTATION.md** - Does your code follow the same pattern?
2. **Check STREAMLIT_IMPLEMENTATION.md** - Are you violating any Streamlit rules?
3. **Add debug mode** - Use the debug sidebar from EXAMPLE_APP_IMPLEMENTATION.md
4. **Simplify** - Remove features until it works, then add back one at a time
5. **Check session state** - Print `st.session_state` to see what's stored

---

## ✨ Final Tips

1. **Copy, don't recreate**: The EXAMPLE_APP_IMPLEMENTATION.md is battle-tested. Use it.
2. **Test frequently**: Test after each function, each module, each feature.
3. **Start simple**: Get basic conversation working before adding features.
4. **Follow the patterns**: The patterns exist because they solve real problems.
5. **Read the comments**: The example code has explanatory comments - read them!

**Good luck! You have everything you need to build a working TerraConverse application.** 🚀

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Status**: CRITICAL - READ FIRST
