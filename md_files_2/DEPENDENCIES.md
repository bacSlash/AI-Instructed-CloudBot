# Dependencies and Setup Instructions

## Required Python Packages

```txt
# requirements.txt

# Web Framework
streamlit==1.31.0

# LLM Integration
groq==0.4.1

# AWS SDK
boto3==1.34.0
botocore==1.34.0

# Data Validation
pydantic==2.5.3

# Template Engine
jinja2==3.1.3

# Retry Logic
tenacity==8.2.3

# Environment Variables
python-dotenv==1.0.0

# Utilities
typing-extensions==4.9.0
```

## System Requirements

- Python 3.10 or higher
- Terraform CLI >= 1.0 (for code validation)
- AWS CLI (optional, for credential management)

## Installation

```bash
# Clone repository
git clone <repository-url>
cd terraconverse

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Copy environment template
cp .env.example .env

# Edit .env and add your Groq API key
nano .env
```

## Environment Configuration

```bash
# .env
GROQ_API_KEY=your_groq_api_key_here
AWS_REGION=us-east-1
LOG_LEVEL=INFO
```

## Running the Application

```bash
streamlit run app.py
```

---