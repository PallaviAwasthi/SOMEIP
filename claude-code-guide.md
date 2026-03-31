# Claude Code: Commands & Software Development Guide

## What is Claude Code?

Claude Code is Anthropic's AI-powered CLI tool that assists with software engineering tasks directly in your terminal. It understands your codebase, writes code, runs commands, and helps debug — all through natural language.

---

## Installation & Setup

```bash
npm install -g @anthropic-ai/claude-code
claude                        # start Claude Code
claude "your prompt here"     # one-shot mode
```

---

## Slash Commands (Built-in)

| Command | Description |
|---|---|
| `/help` | Show available commands and usage |
| `/clear` | Clear conversation history |
| `/compact` | Compress conversation to save context |
| `/model` | View or switch the AI model |
| `/fast` | Toggle fast mode (faster responses) |
| `/cost` | Show token usage and cost estimate |
| `/status` | Show current session status |
| `/memory` | View saved memory entries |
| `/review` | Review recent changes |
| `/quit` or `/exit` | Exit Claude Code |

---

## Core Capabilities

### 1. Understand Code
Ask Claude to explain any part of your codebase:
```
> Explain how authentication works in this project
> What does the UserService class do?
> Walk me through the data flow from API request to database
```

### 2. Read & Search Files
Claude can find and read files without you specifying paths:
```
> Find all API endpoints in this project
> Where is the database connection configured?
> Show me all files that import the Logger module
```

### 3. Write & Edit Code
```
> Add input validation to the registration form
> Create a utility function that formats dates as ISO strings
> Refactor the payment module to use async/await
```

### 4. Debug Issues
```
> This test is failing — help me fix it
> Why is the login endpoint returning a 401?
> Find the memory leak in the image processing service
```

### 5. Run Terminal Commands
Claude can execute shell commands on your behalf:
```
> Run the test suite
> Install the missing dependencies
> Show me the git log for the last week
```

---

## Common Software Development Workflows

### Starting a New Feature
```
> I need to add a user profile page. What files should I create or modify?
> Scaffold a REST API endpoint for /api/products with CRUD operations
> Create a React component for a data table with sorting and pagination
```

### Code Review & Refactoring
```
> Review the changes I just made and suggest improvements
> This function is too long — break it into smaller pieces
> Find duplicate logic in the controllers and extract it to a shared utility
```

### Testing
```
> Write unit tests for the OrderService class
> Add integration tests for the /api/users endpoint
> What edge cases am I missing in these tests?
```

### Git Workflow
```
> What changed since the last commit?
> Write a commit message for these changes
> Create a pull request summary for this branch
```

### Debugging & Error Handling
```
> I'm getting "Cannot read property of undefined" — find the cause
> Add proper error handling to all async functions in this file
> Why is this query slow? Suggest optimizations
```

### Documentation
```
> Generate API documentation for all public endpoints
> Add JSDoc comments to the utility functions
> Explain this algorithm in plain English
```

---

## Working with Specific Tech Stacks

### Node.js / Express
```
> Add rate limiting middleware to the Express app
> Set up Prisma ORM with PostgreSQL
> Create a JWT authentication middleware
```

### React / Frontend
```
> Create a custom hook for fetching and caching API data
> Convert this class component to a functional component with hooks
> Add form validation using React Hook Form
```

### Python / Django / Flask
```
> Add a Django REST Framework serializer for the Product model
> Write a Flask route that handles file uploads
> Create a Celery task for sending emails asynchronously
```

### Databases
```
> Write a migration to add an index on users.email
> Optimize this SQL query — it's doing a full table scan
> Convert this raw SQL to a Django ORM query
```

### DevOps / CI/CD
```
> Write a GitHub Actions workflow for running tests on pull requests
> Create a Dockerfile for this Node.js application
> Add a health check endpoint for the Kubernetes liveness probe
```

---

## Tips for Effective Use

### Be Specific
| Less Effective | More Effective |
|---|---|
| "Fix the bug" | "The login endpoint returns 500 when email contains uppercase letters" |
| "Add a feature" | "Add pagination to the GET /api/posts endpoint, 20 items per page" |
| "Improve this code" | "Refactor this function to reduce nesting and improve readability" |

### Provide Context
```
> We're using PostgreSQL 15, Node.js 20, and Prisma. Add full-text search to the products table.
> This is a multi-tenant SaaS app. Make sure the new endpoint filters by tenant ID.
```

### Iterative Refinement
```
> [After initial output] Make the error messages more user-friendly
> [After a fix] Now add tests for this change
> [After review] The naming doesn't match our convention — rename to camelCase
```

### Use `/compact` for Long Sessions
When working on large tasks, run `/compact` to compress history and free up context without losing the thread.

---

## Memory & Persistence

Claude Code can remember preferences across sessions:

```
> Remember that we use tabs not spaces in this project
> Always use async/await instead of .then() chains
> Our API responses always follow the { data, error, meta } format
```

View saved memories with `/memory`.

---

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl+C` | Cancel current operation |
| `Ctrl+L` | Clear screen |
| `Up Arrow` | Navigate prompt history |
| `Esc` | Interrupt multi-line input |

---

## Permissions & Safety

Claude Code will ask for confirmation before:
- Deleting files or directories
- Force-pushing to git remotes
- Running destructive database operations
- Modifying CI/CD pipelines

You can approve or deny each action individually.

---

## Example: Full Feature Development Session

```
You: Add a password reset flow to the app

Claude: [reads auth files, identifies email service, checks existing routes]
        I'll add the following:
        1. POST /api/auth/forgot-password — sends reset email
        2. POST /api/auth/reset-password — validates token, updates password
        3. PasswordReset model migration
        Shall I proceed?

You: Yes, go ahead

Claude: [creates migration, model, routes, email template, tests]

You: The token should expire in 15 minutes, not 1 hour

Claude: [updates token expiry logic in the reset service]

You: Write a commit message for all of this

Claude: feat(auth): add password reset flow with email verification

You: Create a PR summary

Claude: [generates pull request description with changes, testing notes, and screenshots guidance]
```

---

## Getting Help

- In-session: `/help`
- Issues & feedback: https://github.com/anthropics/claude-code/issues
- Documentation: Run `claude --help` in your terminal
