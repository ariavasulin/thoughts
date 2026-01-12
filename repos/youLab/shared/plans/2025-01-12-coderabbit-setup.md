# CodeRabbit Setup Implementation Plan

## Overview

Set up CodeRabbit AI code review for the YouLab project to provide automated PR reviews with security scanning, anti-pattern detection, and documentation suggestions.

## Current State Analysis

**Existing Tooling:**
- Ruff for linting (comprehensive `select = ["ALL"]` config)
- basedpyright for strict type checking
- Pre-commit hooks running lint + typecheck + tests
- CLAUDE.md exists (CodeRabbit auto-reads this)

**Key Constraints:**
- YouLab is public (eligible for free CodeRabbit tier)
- OpenWebUI is a nested git repo (separate `.git`) - requires separate CodeRabbit setup
- Warnings only, no blocking merges
- All suggestions welcome (including docstrings)

### Key Discoveries:
- CodeRabbit is PR-based only - no "scan entire repo" feature
- Semgrep community rules not bundled due to licensing - must provide custom rules
- CodeRabbit reads CLAUDE.md automatically for coding guidelines

## Desired End State

1. CodeRabbit automatically reviews all PRs to YouLab
2. Security vulnerabilities detected via Semgrep integration
3. Anti-patterns and code smells flagged
4. Documentation improvements suggested
5. One-time audit completed on existing codebase

**Verification:**
- Create a test PR and confirm CodeRabbit posts review comments
- Verify security rules catch test vulnerabilities
- Confirm warnings appear but don't block merges

## What We're NOT Doing

- Not setting up CodeRabbit on OpenWebUI fork (separate task if desired)
- Not enabling merge blocking (warnings only)
- Not configuring CI/CD integration beyond CodeRabbit's native GitHub integration
- Not replacing existing Ruff/basedpyright setup (CodeRabbit complements, doesn't replace)

## Implementation Approach

Install CodeRabbit via GitHub Marketplace, configure via `.coderabbit.yaml`, add Semgrep rules for security, then trigger a one-time audit PR.

---

## Phase 1: Install CodeRabbit via GitHub Marketplace

### Overview
Enable CodeRabbit on the YouLab repository through GitHub's marketplace.

### Steps:

1. **Visit GitHub Marketplace**
   - Go to: https://github.com/marketplace/coderabbitai
   - Click "Set up a plan" or "Install it for free"

2. **Authorize CodeRabbit**
   - Grant requested permissions (read-only access to code, PR comments)
   - Select your GitHub account/organization

3. **Configure Repository Access**
   - Choose "Only select repositories"
   - Select `ariavasulin/YouLab`
   - (Later: add OpenWebUI fork if desired)

4. **Activate in CodeRabbit Dashboard**
   - Visit https://app.coderabbit.ai/settings/repositories
   - Confirm YouLab appears and is active

### Success Criteria:

#### Automated Verification:
- [ ] CodeRabbit appears in repository's installed GitHub Apps
- [ ] Repository visible in CodeRabbit dashboard at app.coderabbit.ai

#### Manual Verification:
- [ ] CodeRabbit dashboard shows YouLab as active
- [ ] No permission errors in GitHub App settings

**Implementation Note**: This phase is manual GitHub UI work. After completing, pause for confirmation before proceeding.

---

## Phase 2: Create `.coderabbit.yaml` Configuration

### Overview
Add CodeRabbit configuration file tailored to YouLab's Python codebase.

### Changes Required:

#### 1. Create Configuration File
**File**: `.coderabbit.yaml` (repository root)

```yaml
# CodeRabbit Configuration for YouLab
# Docs: https://docs.coderabbit.ai/reference/configuration

language: "en-US"
tone_instructions: "Be concise. Focus on bugs, security issues, and Python best practices. This is a learning project so constructive suggestions are welcome."

reviews:
  # Review thoroughness - assertive catches more issues
  profile: "assertive"

  # Don't block merges - warnings only
  request_changes_workflow: false

  # Generate PR summaries
  high_level_summary: true
  poem: false
  review_status: true
  collapse_walkthrough: false

  # Auto-review settings
  auto_review:
    enabled: true
    drafts: false  # Skip draft PRs

  # Exclude generated/vendor code
  path_filters:
    - "!**/__pycache__/**"
    - "!**/.venv/**"
    - "!**/node_modules/**"
    - "!**/*.egg-info/**"
    - "!**/htmlcov/**"
    - "!**/.pytest_cache/**"
    - "!**/.ruff_cache/**"
    # Exclude nested OpenWebUI repo (has its own git)
    - "!OpenWebUI/**"

  # Path-specific review instructions
  path_instructions:
    - path: "src/youlab_server/**/*.py"
      instructions: |
        Focus on:
        - Type hint completeness on public functions
        - Proper async/await patterns
        - Pydantic model validation
        - Exception handling with context
        - Potential security issues (injection, secrets)

        Note: This codebase uses structlog for logging, not print().
        The project uses Letta for AI agents and Honcho for message persistence.

    - path: "src/youlab_server/server/**/*.py"
      instructions: |
        FastAPI-specific checks:
        - Proper dependency injection patterns
        - Correct HTTP status codes
        - Request/response model validation
        - Authentication/authorization patterns
        - Rate limiting considerations

    - path: "src/youlab_server/agents/**/*.py"
      instructions: |
        Letta agent patterns:
        - Proper agent lifecycle management
        - Memory block handling
        - Tool definitions completeness
        - Error recovery in agent operations

    - path: "src/youlab_server/memory/**/*.py"
      instructions: |
        Memory management:
        - Thread safety for concurrent access
        - Proper cleanup/disposal patterns
        - Memory leak prevention

    - path: "tests/**/*.py"
      instructions: |
        Test quality:
        - Adequate edge case coverage
        - Proper pytest fixture usage
        - Async test markers where needed
        - Descriptive test names
        - Avoid testing implementation details

    - path: "config/**/*.toml"
      instructions: |
        TOML configuration:
        - Valid syntax
        - Required fields present
        - Consistent naming conventions

    - path: "docs/**/*.md"
      instructions: |
        Documentation:
        - Check if content matches current code behavior
        - Flag outdated API references
        - Verify code examples are syntactically valid

  # Pre-merge checks (warnings only)
  pre_merge_checks:
    docstrings:
      mode: "warning"
      threshold: 50  # Start low, can increase later
    title:
      mode: "warning"
    description:
      mode: "warning"
    custom_checks:
      - name: "No Hardcoded Secrets"
        mode: "warning"
        instructions: |
          Check for hardcoded API keys, passwords, tokens, or secrets.
          Secrets should be loaded from environment variables or .env files.
      - name: "Deprecated Module Usage"
        mode: "warning"
        instructions: |
          Flag imports from deprecated modules (per CLAUDE.md):
          - agents/templates.py (use config/courses/*.toml)
          - agents/default.py (use AgentManager.create_agent())
          - agents/base.py (use Letta agents via AgentManager)
          - memory/manager.py (use agent-driven memory)
          - memory/blocks.py PersonaBlock/HumanBlock (use dynamic TOML blocks)

  # Integrated tools
  tools:
    # Python linting - Ruff already runs locally, but CodeRabbit can catch PR-only issues
    ruff:
      enabled: true

    # Security scanning
    semgrep:
      enabled: true
      config_file: ".semgrep.yaml"
    gitleaks:
      enabled: true

    # Shell scripts
    shellcheck:
      enabled: true

    # Markdown
    markdownlint:
      enabled: true

# Knowledge base settings
knowledge_base:
  opt_out: false
  code_guidelines: true  # Reads CLAUDE.md automatically
  learnings:
    scope: "local"  # Learn patterns from this repo only

# Docstring generation settings
code_generation:
  docstrings:
    path_instructions:
      - path: "src/**/*.py"
        instructions: |
          Use Google-style docstrings:
          - Brief one-line summary
          - Args section with types and descriptions
          - Returns section with type and description
          - Raises section for exceptions

          Example:
          def process_message(content: str, user_id: str) -> Message:
              \"\"\"Process and store a user message.

              Args:
                  content: The message content to process.
                  user_id: The ID of the user sending the message.

              Returns:
                  The processed Message object with metadata.

              Raises:
                  ValidationError: If content is empty or exceeds limits.
              \"\"\"
```

### Success Criteria:

#### Automated Verification:
- [ ] YAML syntax valid: `python -c "import yaml; yaml.safe_load(open('.coderabbit.yaml'))"`
- [ ] File committed to repository

#### Manual Verification:
- [ ] Comment `@coderabbitai configuration` on any PR to verify config is loaded
- [ ] Verify path filters exclude expected directories

**Implementation Note**: After creating this file, we need to commit it before CodeRabbit can use it.

---

## Phase 3: Create Semgrep Security Rules

### Overview
Add custom Semgrep rules for OWASP vulnerability detection since CodeRabbit doesn't bundle community rules.

### Changes Required:

#### 1. Create Semgrep Configuration
**File**: `.semgrep.yaml` (repository root)

```yaml
# Semgrep Security Rules for YouLab
# These rules detect common Python security vulnerabilities
# Docs: https://semgrep.dev/docs/writing-rules/rule-syntax

rules:
  # === INJECTION VULNERABILITIES ===

  - id: python-sql-injection
    patterns:
      - pattern-either:
          - pattern: $CURSOR.execute($QUERY + ...)
          - pattern: $CURSOR.execute(f"...")
          - pattern: $CURSOR.execute("..." % ...)
          - pattern: $CURSOR.executemany($QUERY + ...)
    message: "Potential SQL injection. Use parameterized queries instead."
    severity: ERROR
    languages: [python]
    metadata:
      owasp: "A03:2021 - Injection"
      cwe: "CWE-89"

  - id: python-command-injection
    patterns:
      - pattern-either:
          - pattern: os.system(...)
          - pattern: os.popen(...)
          - pattern: subprocess.call($CMD, shell=True, ...)
          - pattern: subprocess.run($CMD, shell=True, ...)
          - pattern: subprocess.Popen($CMD, shell=True, ...)
    message: "Potential command injection. Avoid shell=True and use list arguments."
    severity: ERROR
    languages: [python]
    metadata:
      owasp: "A03:2021 - Injection"
      cwe: "CWE-78"

  - id: python-code-injection
    patterns:
      - pattern-either:
          - pattern: eval(...)
          - pattern: exec(...)
          - pattern: compile(..., ..., "exec")
    message: "Dangerous use of eval/exec. Avoid executing dynamic code."
    severity: ERROR
    languages: [python]
    metadata:
      owasp: "A03:2021 - Injection"
      cwe: "CWE-94"

  # === DESERIALIZATION ===

  - id: python-unsafe-deserialization
    patterns:
      - pattern-either:
          - pattern: pickle.loads(...)
          - pattern: pickle.load(...)
          - pattern: yaml.load(..., Loader=yaml.Loader)
          - pattern: yaml.load(..., Loader=yaml.UnsafeLoader)
          - pattern: yaml.unsafe_load(...)
    message: "Unsafe deserialization. Use yaml.safe_load() or avoid pickle for untrusted data."
    severity: ERROR
    languages: [python]
    metadata:
      owasp: "A08:2021 - Software and Data Integrity Failures"
      cwe: "CWE-502"

  # === HARDCODED SECRETS ===

  - id: python-hardcoded-secret
    patterns:
      - pattern-either:
          - pattern: $VAR = "..."
          - pattern: $VAR = '...'
    pattern-regex: '(?i)(password|secret|api_key|apikey|token|auth|credential).*["\'][a-zA-Z0-9+/=]{8,}["\']'
    message: "Potential hardcoded secret. Use environment variables instead."
    severity: WARNING
    languages: [python]
    metadata:
      owasp: "A07:2021 - Identification and Authentication Failures"
      cwe: "CWE-798"

  - id: python-hardcoded-password-assignment
    pattern: |
      password = "..."
    message: "Hardcoded password detected. Load from environment variable."
    severity: ERROR
    languages: [python]
    metadata:
      cwe: "CWE-259"

  # === CRYPTOGRAPHIC ISSUES ===

  - id: python-weak-hash
    patterns:
      - pattern-either:
          - pattern: hashlib.md5(...)
          - pattern: hashlib.sha1(...)
    message: "Weak hash algorithm. Use SHA-256 or stronger for security purposes."
    severity: WARNING
    languages: [python]
    metadata:
      owasp: "A02:2021 - Cryptographic Failures"
      cwe: "CWE-328"

  - id: python-insecure-random
    patterns:
      - pattern-either:
          - pattern: random.random()
          - pattern: random.randint(...)
          - pattern: random.choice(...)
    message: "Insecure random for security purposes. Use secrets module instead."
    severity: WARNING
    languages: [python]
    metadata:
      owasp: "A02:2021 - Cryptographic Failures"
      cwe: "CWE-330"

  # === NETWORK SECURITY ===

  - id: python-ssl-verification-disabled
    patterns:
      - pattern-either:
          - pattern: requests.get(..., verify=False, ...)
          - pattern: requests.post(..., verify=False, ...)
          - pattern: httpx.get(..., verify=False, ...)
          - pattern: httpx.post(..., verify=False, ...)
          - pattern: httpx.Client(..., verify=False, ...)
          - pattern: httpx.AsyncClient(..., verify=False, ...)
    message: "SSL verification disabled. This allows man-in-the-middle attacks."
    severity: ERROR
    languages: [python]
    metadata:
      owasp: "A07:2021 - Identification and Authentication Failures"
      cwe: "CWE-295"

  # === PATH TRAVERSAL ===

  - id: python-path-traversal
    patterns:
      - pattern-either:
          - pattern: open($PATH + ..., ...)
          - pattern: open(f"...", ...)
          - pattern: pathlib.Path($PATH + ...)
    message: "Potential path traversal. Validate and sanitize file paths."
    severity: WARNING
    languages: [python]
    metadata:
      owasp: "A01:2021 - Broken Access Control"
      cwe: "CWE-22"

  # === LOGGING ISSUES ===

  - id: python-sensitive-data-logging
    patterns:
      - pattern-either:
          - pattern: logging.$METHOD(..., password=..., ...)
          - pattern: logging.$METHOD(..., token=..., ...)
          - pattern: logging.$METHOD(..., secret=..., ...)
          - pattern: logger.$METHOD(..., password=..., ...)
          - pattern: logger.$METHOD(..., token=..., ...)
          - pattern: structlog.$METHOD(..., password=..., ...)
    message: "Potential sensitive data in logs. Mask or remove secrets before logging."
    severity: WARNING
    languages: [python]
    metadata:
      owasp: "A09:2021 - Security Logging and Monitoring Failures"
      cwe: "CWE-532"

  # === FASTAPI SPECIFIC ===

  - id: fastapi-missing-rate-limit
    pattern: |
      @app.post(...)
      async def $FUNC(...):
          ...
    message: "Consider adding rate limiting to this endpoint."
    severity: INFO
    languages: [python]
    metadata:
      owasp: "A04:2021 - Insecure Design"

  # === EXCEPTION HANDLING ===

  - id: python-bare-except
    pattern: |
      try:
          ...
      except:
          ...
    message: "Bare except catches all exceptions including KeyboardInterrupt. Catch specific exceptions."
    severity: WARNING
    languages: [python]
    metadata:
      cwe: "CWE-396"

  - id: python-exception-pass
    pattern: |
      try:
          ...
      except $E:
          pass
    message: "Silent exception handling. At minimum, log the error."
    severity: WARNING
    languages: [python]
    metadata:
      cwe: "CWE-390"
```

### Success Criteria:

#### Automated Verification:
- [ ] YAML syntax valid: `python -c "import yaml; yaml.safe_load(open('.semgrep.yaml'))"`
- [ ] Semgrep parses rules: `semgrep --config .semgrep.yaml --validate` (if semgrep installed locally)

#### Manual Verification:
- [ ] CodeRabbit uses rules when reviewing PRs (visible in review comments)

**Implementation Note**: The Semgrep rules focus on Python-specific vulnerabilities relevant to this web service codebase.

---

## Phase 4: Trigger One-Time Full Repository Audit

### Overview
Create a PR that triggers CodeRabbit to review the entire codebase, providing a baseline security and quality audit.

### Steps:

1. **Create an audit branch**
   ```bash
   git checkout -b audit/coderabbit-initial-review
   ```

2. **Add a trigger file** (will be removed after audit)
   ```bash
   echo "# CodeRabbit Audit - $(date -I)" > .audit-trigger
   git add .audit-trigger
   git commit -m "chore: trigger CodeRabbit initial audit"
   ```

3. **Push and create PR**
   ```bash
   git push -u origin audit/coderabbit-initial-review
   gh pr create --title "chore: CodeRabbit Initial Codebase Audit" --body "$(cat <<'EOF'
   ## Purpose

   Trigger a full CodeRabbit review of the existing codebase to identify:
   - Security vulnerabilities
   - Anti-patterns and code smells
   - Documentation gaps
   - Potential improvements

   ## Instructions for CodeRabbit

   @coderabbitai full review

   Please perform a comprehensive review of:
   1. All Python code in `src/youlab_server/`
   2. All test files in `tests/`
   3. Configuration files in `config/`
   4. Documentation in `docs/`

   Focus areas:
   - Security vulnerabilities (OWASP Top 10)
   - Type safety and error handling
   - Async/await patterns
   - Memory management
   - API design

   ## After Review

   This PR will be closed without merging. The audit trigger file is temporary.
   EOF
   )"
   ```

4. **Wait for CodeRabbit review** (typically 5-10 minutes for full review)

5. **Review findings and create issues**
   - Go through CodeRabbit's comments
   - Create GitHub issues for actionable findings
   - Prioritize security issues

6. **Close the audit PR**
   ```bash
   gh pr close audit/coderabbit-initial-review
   git checkout main
   git branch -D audit/coderabbit-initial-review
   ```

### Success Criteria:

#### Automated Verification:
- [ ] PR created successfully
- [ ] CodeRabbit posts review comments

#### Manual Verification:
- [ ] Review comments cover security, patterns, and documentation
- [ ] Actionable findings documented as GitHub issues
- [ ] No critical security vulnerabilities left unaddressed

**Implementation Note**: This is a one-time audit. After completing, close the PR without merging. Future PRs will be automatically reviewed.

---

## Testing Strategy

### Integration Tests:
- Create a test PR with intentional issues to verify CodeRabbit catches them:
  - Add `eval()` call → should trigger security warning
  - Add function without type hints → should suggest adding them
  - Add bare `except:` → should warn about exception handling

### Manual Testing Steps:
1. After Phase 1: Verify CodeRabbit appears in GitHub App installations
2. After Phase 2: Comment `@coderabbitai configuration` on any PR to verify config loaded
3. After Phase 3: Create PR with security issue to test Semgrep rules
4. After Phase 4: Review audit findings for completeness

## Future Considerations

- **OpenWebUI Fork**: If desired, repeat Phase 1 for the OpenWebUI fork repo
- **Docstring Threshold**: Start at 50%, increase to 75% once baseline established
- **Custom Rules**: Add more Semgrep rules based on audit findings
- **CI Integration**: Consider adding `semgrep` to pre-commit hooks locally

## References

- CodeRabbit Docs: https://docs.coderabbit.ai/
- Configuration Reference: https://docs.coderabbit.ai/reference/configuration
- Semgrep Rule Syntax: https://semgrep.dev/docs/writing-rules/rule-syntax
- OWASP Top 10: https://owasp.org/Top10/
