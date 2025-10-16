# LLM Code Deployment API

An automated application that receives task briefs, generates code using LLMs, deploys to GitHub Pages, and manages multi-round revisions—all through a FastAPI-based REST endpoint.

## Overview

This project implements an end-to-end automated code deployment system that:
- **Receives** task requests via JSON POST to an API endpoint
- **Verifies** user authentication using a secret key
- **Generates** complete web applications using OpenAI's LLM
- **Creates** GitHub repositories with proper structure and licensing
- **Deploys** applications to GitHub Pages automatically
- **Handles** multi-round revisions to iteratively improve generated code
- **Notifies** evaluation servers with deployment details

## Project Structure

```
app/
├── __init__.py           # Package initialization
├── main.py               # FastAPI application and request handling
├── llm_generator.py      # OpenAI integration for code generation
├── github_utils.py       # GitHub API operations (repos, files, Pages)
├── notify.py             # Evaluation server notification with retry logic
└── signature.py          # Authentication utilities
requirements.txt          # Python dependencies
.env                      # Environment variables (not tracked in git)
```

## Features

### Round 1: Build & Deploy
- Accept JSON POST requests with task briefs and evaluation criteria
- Validate user secret for authentication
- Parse attachments (images, data files) from base64-encoded data URIs
- Generate complete application code using OpenAI GPT models
- Create public GitHub repository with unique name based on task ID
- Add MIT license automatically
- Enable GitHub Pages for instant deployment
- Notify evaluation server with repository and deployment details

### Round 2: Revise & Update
- Accept modification requests for existing repositories
- Load previous README and code for context-aware updates
- Update application code based on new requirements
- Maintain git history without exposing secrets
- Re-deploy updated application to GitHub Pages
- Send updated metadata to evaluation server

## Setup Instructions

### Prerequisites
- Python 3.11 or higher
- GitHub account with Personal Access Token
- OpenAI API key

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd <project-directory>
```

### 2. Create Virtual Environment

```bash
python -m venv venv
```

Activate the virtual environment:

**Windows (PowerShell):**
```powershell
.\venv\Scripts\Activate.ps1
```

**Windows (Command Prompt):**
```cmd
venv\Scripts\activate.bat
```

**macOS/Linux:**
```bash
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables

Create a `.env` file in the project root:

```env
# GitHub Configuration
GITHUB_TOKEN=ghp_your_github_personal_access_token
GITHUB_USERNAME=your_github_username

# OpenAI Configuration
OPENAI_API_KEY=sk-your_openai_api_key

# Optional: Use custom OpenAI-compatible endpoint
# OPENAI_BASE_URL=https://api.openai.com/v1

# Authentication Secret (shared via submission form)
USER_SECRET=your_secret_phrase
```

**Important:** Never commit the `.env` file to version control. Add it to `.gitignore`.

### 5. Run the Application

```bash
uvicorn app.main:app --reload
```

The API will be available at `http://localhost:8000`

For custom port:
```bash
uvicorn app.main:app --reload --port 5000
```

## Usage

### API Endpoint

**POST** `/api-endpoint`

Accepts JSON requests with the following structure:

```json
{
  "email": "student@example.com",
  "secret": "your_secret_phrase",
  "task": "calculator-app-001",
  "round": 1,
  "nonce": "unique-nonce-value",
  "brief": "Create a simple calculator web app with +, -, *, / operations",
  "checks": [
    "Repo has MIT license",
    "README.md is professional",
    "Calculator performs basic arithmetic",
    "UI is clean and responsive"
  ],
  "evaluation_url": "https://example.com/evaluate",
  "attachments": [
    {
      "name": "logo.png",
      "url": "data:image/png;base64,iVBORw0KG..."
    }
  ]
}
```

### Response

On success, returns HTTP 200 with:
```json
{
  "status": "success",
  "message": "Request accepted and processing in background"
}
```

The system then:
1. Generates application code using OpenAI
2. Creates/updates GitHub repository
3. Deploys to GitHub Pages
4. Notifies evaluation server with:
   - Repository URL
   - Commit SHA
   - GitHub Pages URL

### Testing the API

**PowerShell Example:**

```powershell
$json = @'
{
  "email": "test@example.com",
  "secret": "your_secret",
  "task": "test-app-001",
  "round": 1,
  "nonce": "test-nonce-123",
  "brief": "Create a simple hello world webpage",
  "checks": ["Has MIT license", "README is present"],
  "evaluation_url": "https://httpbin.org/post",
  "attachments": []
}
'@

Invoke-RestMethod -Uri 'http://localhost:8000/api-endpoint' `
  -Method Post -ContentType 'application/json' -Body $json
```

**cURL Example:**

```bash
curl -X POST http://localhost:8000/api-endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "secret": "your_secret",
    "task": "test-app-001",
    "round": 1,
    "nonce": "test-nonce-123",
    "brief": "Create a simple hello world webpage",
    "checks": ["Has MIT license"],
    "evaluation_url": "https://httpbin.org/post",
    "attachments": []
  }'
```

## Code Explanation

### `app/main.py`
The FastAPI application entry point that:
- Loads environment variables using `python-dotenv`
- Defines the `/api-endpoint` POST route
- Validates incoming requests against `USER_SECRET`
- Launches background tasks for asynchronous processing
- Handles both Round 1 (new repo) and Round 2 (revision) workflows

### `app/llm_generator.py`
Manages OpenAI integration:
- Initializes OpenAI client with API key and optional custom base URL
- Decodes base64 attachments from data URIs
- Generates complete application code (HTML, CSS, JavaScript, README)
- Creates context-aware prompts for LLM with task requirements
- Handles text and binary file previews for improved generation

### `app/github_utils.py`
Provides GitHub operations via PyGithub:
- Creates public repositories with unique names
- Manages file creation and updates (text and binary)
- Generates MIT license text
- Enables GitHub Pages deployment via REST API
- Handles existing repositories gracefully

### `app/notify.py`
Handles evaluation server communication:
- Sends POST requests to evaluation URLs with retry logic
- Implements exponential backoff (1s, 2s, 4s, 8s, 16s)
- Returns success/failure status for monitoring

### `app/signature.py`
Reserved for authentication utilities (currently unused).

## Dependencies

Key packages from `requirements.txt`:

| Package | Version | Purpose |
|---------|---------|---------|
| `fastapi` | 0.118.0 | Web framework for API endpoints |
| `uvicorn` | 0.37.0 | ASGI server for running FastAPI |
| `openai` | 1.109.1 | OpenAI API client for code generation |
| `PyGithub` | 2.8.1 | GitHub API v3 wrapper |
| `httpx` | 0.28.1 | Modern HTTP client for notifications |
| `python-dotenv` | 1.1.1 | Environment variable management |
| `pydantic` | 2.11.9 | Data validation |
| `requests` | 2.32.5 | HTTP library for API calls |

## Security Considerations

- **Never commit** `.env` file or expose API keys
- **Validate** user secrets before processing requests
- **Use** environment variables for all sensitive data
- **Enable** GitHub's secret scanning on repositories
- **Verify** no secrets appear in git history (use `trufflehog`, `gitleaks`)

## License

This project is licensed under the MIT License.

## Troubleshooting

### "OPENAI_API_KEY is not set" Error
Ensure your `.env` file contains a valid OpenAI API key and is in the project root directory.

### "Invalid credentials" from GitHub
Verify your `GITHUB_TOKEN` has the following scopes:
- `repo` (full control of private repositories)
- `workflow` (if using GitHub Actions)

### GitHub Pages Not Enabling
- Ensure repository is public
- Check that the repository has at least one commit
- Wait 1-2 minutes for GitHub Pages to propagate

### Request Timeout
The system processes requests asynchronously. Check:
- Server logs for error messages
- GitHub repository creation status
- OpenAI API rate limits

## Contact & Support

For issues or questions about this project, please open an issue on the repository or contact the development team.

---

**Last Updated:** October 2025


