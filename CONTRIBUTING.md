# Contributing Guidelines

To ensure smooth collaboration for the System Design Project, all team members should adhere to the following branching and commit guidelines.

## Branching Strategy

We follow a structured feature-branch workflow. All work should be branched off `develop` and merged back via Pull Requests.

- `main`: Stable, production-ready code and finalized, submitted documentation.
- `develop`: The active integration branch. All features and ongoing document updates are merged here first.
- `feature/<issue-number>-<short-desc>`: For new system features, front-end components, or prototype additions (e.g., `feature/12-bed-allocation`).
- `doc/<issue-number>-<short-desc>`: For drafting or updating LaTeX reports and project documents (e.g., `doc/4-srs-draft`).
- `fix/<issue-number>-<short-desc>`: For bug fixes in the codebase or typos in documents.

## Commit Message Format

We use a standardized format to keep the project history clean and easy to navigate.

**Format:**
`<type>(<scope>): <description>`

**Allowed Types:**
- `feat`: A new feature or major document section.
- `fix`: A bug fix or typo correction.
- `docs`: Documentation only changes (e.g., updating the README).
- `style`: Changes that do not affect the meaning of the code (white-space, formatting, etc).
- `refactor`: A code change that neither fixes a bug nor adds a feature.
- `chore`: Routine tasks, updating dependencies, or modifying `.gitignore`.

**Examples:**
- `docs(readme): add repository structure and project description`
- `feat(ward): implement real-time patient tracking interface`
- `chore(gitignore): add rules for node_modules and OS files`