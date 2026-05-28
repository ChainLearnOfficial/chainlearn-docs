# Contributing Guide

Welcome to ChainLearn! This guide covers how to contribute to the platform, including our PR process, code style, and development workflow.

## Getting Started

1. Read the [Architecture Overview](../architecture/overview.md) to understand the system
2. Set up your [local development environment](local-development.md)
3. Pick an issue from the [GitHub Issues](https://github.com/chainlearn) board
4. Create a branch and start coding

## Repository Structure

Each repository follows a consistent structure:

```
repo/
├── .github/
│   └── workflows/       # CI/CD pipelines
├── src/                 # Source code
├── tests/               # Test files
├── docs/                # Repo-specific documentation
├── .env.example         # Environment variable template
├── .eslintrc.js         # Linting config (JS/TS repos)
├── package.json         # Dependencies (JS repos)
└── README.md            # Repo-specific setup instructions
```

## Branch Naming

Use descriptive branch names with prefixes:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feat/` | New feature | `feat/add-course-categories` |
| `fix/` | Bug fix | `fix/quiz-scoring-rounding` |
| `docs/` | Documentation | `docs/update-api-spec` |
| `refactor/` | Code refactoring | `refactor/extract-auth-middleware` |
| `test/` | Adding tests | `test/quiz-submission-integration` |
| `chore/` | Maintenance | `chore/update-dependencies` |

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**
- `feat` -- New feature
- `fix` -- Bug fix
- `docs` -- Documentation only
- `style` -- Formatting, no code change
- `refactor` -- Code change that neither fixes a bug nor adds a feature
- `test` -- Adding or updating tests
- `chore` -- Maintenance tasks

**Examples:**
```
feat(api): add course enrollment endpoint
fix(contracts): correct token transfer amount calculation
docs(api): update OpenAPI spec for quiz endpoints
test(ai): add integration tests for quiz generation
```

## Pull Request Process

### 1. Create Your Branch

```bash
git checkout -b feat/my-feature
```

### 2. Make Changes

- Write code following the style guide below
- Add or update tests for your changes
- Update documentation if needed

### 3. Test Locally

```bash
# Run the relevant test suite
npm test          # JS/TS repos
pytest            # Python repos
cargo test        # Rust repos

# Run linting
npm run lint      # JS/TS repos
ruff check .      # Python repos
cargo clippy      # Rust repos
```

### 4. Commit and Push

```bash
git add .
git commit -m "feat(api): add course enrollment endpoint"
git push origin feat/my-feature
```

### 5. Open a Pull Request

Fill out the PR template:

```markdown
## What

Brief description of the change.

## Why

Why this change is needed.

## How

Implementation details.

## Testing

How this was tested.

## Checklist
- [ ] Tests pass locally
- [ ] Linting passes
- [ ] Documentation updated (if needed)
- [ ] No breaking changes (or documented)
```

### 6. Code Review

- All PRs require at least 1 approval
- Address all review comments
- CI must pass (tests, linting, build)

### 7. Merge

We use **squash and merge** for clean git history. The PR title becomes the commit message, so make it descriptive.

## Code Style

### TypeScript / JavaScript (API, Frontend, Indexer)

- **Formatter:** Prettier (configured in `.prettierrc`)
- **Linter:** ESLint with the project config
- **Naming:** camelCase for variables/functions, PascalCase for types/classes, SCREAMING_SNAKE_CASE for constants
- **Imports:** Absolute imports from `@/` prefix, sorted by type

```typescript
// Good
import { Request, Response } from 'express';

import { prisma } from '@/lib/prisma';
import { validateAuth } from '@/middleware/auth';
import { CourseService } from '@/services/course';

export async function getCourse(req: Request, res: Response): Promise<void> {
  const courseId = req.params.id;
  const course = await CourseService.findById(courseId);
  res.json(course);
}
```

### Python (AI Service)

- **Formatter:** Black
- **Linter:** Ruff
- **Type hints:** Required for all function signatures
- **Naming:** snake_case for variables/functions, PascalCase for classes

```python
from fastapi import APIRouter, HTTPException

from app.services.cohere_client import CohereClient
from app.models.quiz import QuizRequest, QuizResponse

router = APIRouter()

@router.post("/generate/quiz", response_model=QuizResponse)
async def generate_quiz(request: QuizRequest) -> QuizResponse:
    """Generate quiz questions based on course content."""
    client = CohereClient()
    questions = await client.generate_quiz(
        content=request.course_content,
        difficulty=request.difficulty,
        count=request.question_count,
    )
    return QuizResponse(questions=questions)
```

### Rust (Smart Contracts)

- **Formatter:** rustfmt (default config)
- **Linter:** Clippy with `--deny warnings`
- **Naming:** snake_case for functions/variables, PascalCase for types, SCREAMING_SNAKE_CASE for constants

```rust
use soroban_sdk::{contract, contractimpl, Address, Env, String};

#[contract]
pub struct LearnToken;

#[contractimpl]
impl LearnToken {
    pub fn transfer(env: Env, from: Address, to: Address, amount: i128) {
        from.require_auth();
        assert!(amount > 0, "Amount must be positive");
        // Implementation
    }
}
```

### YAML / OpenAPI

- 2-space indentation
- Alphabetical keys where practical
- Descriptions on all endpoints and fields

## Testing Standards

### Coverage Requirements

| Repo | Minimum Coverage | Test Types |
|------|-----------------|------------|
| chainlearn-api | 80% | Unit, integration |
| chainlearn-ai | 75% | Unit, integration |
| chainlearn-contracts | 90% | Unit, integration |
| chainlearn-frontend | 70% | Unit, component |
| chainlearn-indexer | 80% | Unit, integration |

### Test Naming

```typescript
// describe/it pattern
describe('CourseService', () => {
  describe('enroll', () => {
    it('should create an enrollment record', async () => { ... });
    it('should throw if already enrolled', async () => { ... });
    it('should call ProgressTracker contract', async () => { ... });
  });
});
```

```python
# pytest pattern
class TestQuizGeneration:
    def test_generates_correct_number_of_questions(self): ...
    def test_respects_difficulty_setting(self): ...
    def test_handles_empty_content_gracefully(self): ...
```

### Contract Tests

Contract tests must cover:
- Happy path for all public functions
- Authorization checks (unauthorized calls should fail)
- Edge cases (zero amounts, duplicate enrollments, etc.)
- Event emission verification

## Documentation Standards

- All public functions must have doc comments
- API endpoints must be documented in the OpenAPI spec
- Architecture decisions are recorded as ADRs in `chainlearn-docs/decisions/`
- README files in each repo must be kept up to date

## Issue Labels

| Label | Meaning |
|-------|---------|
| `bug` | Something is broken |
| `enhancement` | New feature or improvement |
| `documentation` | Docs need updating |
| `good first issue` | Good for new contributors |
| `help wanted` | Extra attention needed |
| `priority: high` | Urgent fix needed |
| `priority: low` | Nice to have |

## Getting Help

- **Discord:** `#chainlearn-dev` channel
- **GitHub Discussions:** For architecture and design questions
- **GitHub Issues:** For bugs and feature requests

## License

By contributing to ChainLearn, you agree that your contributions will be licensed under the MIT License.
