# Contributing to DevOps on GCP Training Materials

Thank you for considering contributing to this repository! Your improvements, corrections, and new content help the entire community learn GCP DevOps more effectively.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How to Contribute](#how-to-contribute)
- [Types of Contributions](#types-of-contributions)
- [Development Setup](#development-setup)
- [Content Guidelines](#content-guidelines)
- [Pull Request Process](#pull-request-process)
- [Reporting Issues](#reporting-issues)
- [Style Guide](#style-guide)

---

## Code of Conduct

This project is committed to providing a welcoming and inclusive environment for all contributors. By participating, you agree to:

- Be respectful and constructive in all interactions
- Welcome contributions from people of all backgrounds and experience levels
- Focus feedback on the content, not the contributor
- Assume good faith in all communications

Unacceptable behavior includes harassment, personal attacks, discriminatory language, or any conduct that would be inappropriate in a professional setting. Report violations to the repository maintainers.

---

## How to Contribute

### Quick Fixes (1–5 minutes)

For typos, broken commands, or outdated information:

1. Click the **Edit** (pencil) icon on any file in GitHub
2. Make your change directly in the browser
3. Scroll down, write a brief commit message
4. Click **Propose changes** → **Create pull request**

### Substantive Contributions (features, new sections, new files)

1. **Fork** this repository to your GitHub account
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/YOUR_USERNAME/devops-gcp-cloud-platform.git
   cd devops-gcp-cloud-platform
   ```
3. **Create a branch** for your change:
   ```bash
   git checkout -b fix/update-gke-commands
   # or
   git checkout -b feat/add-cloud-armor-section
   ```
4. **Make your changes** (see [Content Guidelines](#content-guidelines))
5. **Commit** with a descriptive message:
   ```bash
   git add .
   git commit -m "docs: add Cloud Armor examples to CONCEPTS.md"
   ```
6. **Push** to your fork:
   ```bash
   git push origin fix/update-gke-commands
   ```
7. **Open a Pull Request** against the `main` branch of this repository

---

## Types of Contributions

We welcome contributions in these categories:

### 🐛 Bug Fixes
- Incorrect or outdated `gcloud` commands
- Wrong flags, deprecated options, or changed API syntax
- Broken links or references
- Factual errors in concept explanations

### 📝 Content Improvements
- Clearer explanations of complex concepts
- Additional examples for existing sections
- Better formatting or structure
- Filling gaps in coverage

### ✨ New Content
- New GCP services not currently covered
- Additional use cases or real-world examples
- New troubleshooting scenarios
- Lab exercises or guided walkthroughs

### 🔧 Infrastructure
- Improvements to `.gitignore`
- Repository structure improvements
- Workflow automation

---

## Development Setup

No special tooling is required to contribute documentation. You need:

- A text editor (VS Code recommended)
- Git (installed locally)
- Optionally: `gcloud` SDK to test commands before submitting

### Verify Commands Before Submitting

All `gcloud` commands in this repository should be tested against a real GCP account. Before submitting new commands:

```bash
# Set up a test project
gcloud projects create test-contrib-$(date +%s)
gcloud config set project TEST_PROJECT_ID

# Test your commands
gcloud <your command here>

# Clean up
gcloud projects delete TEST_PROJECT_ID
```

---

## Content Guidelines

### Accuracy

- **Test all commands** against a real GCP environment before submitting
- Include the correct flags and required parameters
- Note when commands behave differently across versions or regions
- If a command may incur costs, add a ⚠️ cost warning

### Clarity

- Write for an audience with intermediate Linux/CLI skills but potentially new to GCP
- Define acronyms on first use in each document
- Explain *why* a command/approach is recommended, not just *how*
- Use concrete examples with realistic (but obviously fake) project IDs and names

### Completeness

- Each new concept should include: what it is, when to use it, examples, best practices, and cost notes
- Command examples should be self-contained (reader should be able to copy-paste and run them)
- Include expected output or describe what the user should see after running a command

### Format Conventions

Use the [Style Guide](#style-guide) below for formatting. Consistent formatting makes the material easier to scan and learn from.

---

## Pull Request Process

### Before Submitting

- [ ] Commands are tested against a real GCP environment
- [ ] Formatting follows the [Style Guide](#style-guide)
- [ ] No credentials, project IDs, or account emails from real accounts are included
- [ ] Placeholder values use clearly fake examples (`MY_PROJECT_ID`, `YOUR_ACCOUNT@example.com`)
- [ ] Any new sections are added to the relevant Table of Contents

### PR Title Format

Use conventional commit style:
```
docs: fix typo in CONCEPTS.md GKE section
docs: add Cloud Armor WAF examples to CONCEPTS.md
fix: correct gcloud run deploy flags in COMMANDS.md
feat: add Cloud Deploy section to TOOLS.md
```

### PR Description

Include:
- **What**: Brief description of the change
- **Why**: Why this change is needed (fixes issue, improves clarity, adds missing content)
- **Tested**: Confirmation that commands were tested
- **Issues**: Reference any related GitHub issues with `Fixes #123`

### Review Process

1. A maintainer will review your PR within 7 days
2. You may receive requests for changes — please respond to feedback within 14 days
3. Once approved, a maintainer will merge your PR

---

## Reporting Issues

Found a bug or have a suggestion? [Open an issue](../../issues/new) with:

### Bug Reports

Use this template:
```
**Problem**: Describe what is wrong (e.g., "gcloud command in COMMANDS.md line 47 fails with error X")

**Expected behavior**: What should happen

**Actual behavior**: What actually happens (include error message)

**Environment**:
- gcloud version: (run `gcloud version`)
- OS: (Linux/macOS/Windows)
- GCP region: (if relevant)
```

### Feature Requests

```
**Summary**: Brief description of the proposed addition

**Motivation**: Why this would be useful to learners

**Proposed content**: Sketch of what you'd like to see (concept, commands, etc.)
```

---

## Style Guide

### Markdown

- Use `##` for top-level section headers, `###` for subsections, `####` for sub-subsections
- Use fenced code blocks with language specifiers (` ```bash `, ` ```hcl `, ` ```yaml `)
- Use tables for comparisons, feature lists, and reference material
- Use `**bold**` for important terms and UI elements
- Use `_italics_` sparingly, only for emphasis
- Use `backticks` for all command names, file names, flags, and inline code

### Code Blocks

```bash
# Add comments to explain non-obvious commands
gcloud compute instances create my-vm \
  --zone=us-central1-a \       # Always specify zone explicitly
  --machine-type=e2-micro \    # Cheapest general-purpose type
  --image-family=debian-12 \
  --image-project=debian-cloud
```

- Use `\` for multi-line commands to improve readability
- Add `# comments` explaining why, not just what
- Use realistic but clearly fake values: `my-project`, `my-bucket`, `MY_PROJECT_ID`
- Do not include real project IDs, account emails, billing account IDs, or API keys

### Placeholder Values

Use these placeholder patterns consistently:

| Context | Placeholder |
|---------|------------|
| Project ID | `MY_PROJECT_ID` or `my-project` |
| Region | `us-central1` (actual region, not placeholder) |
| Account email | `YOUR_ACCOUNT@example.com` |
| Billing account | `BILLING_ACCOUNT_ID` |
| Cluster name | `my-cluster` or `CLUSTER_NAME` |
| Instance name | `my-vm` or `INSTANCE_NAME` |
| Bucket name | `my-bucket` or `BUCKET_NAME` |

### File Structure

Each major document follows this pattern:
1. Title + brief description
2. Table of Contents (for files > 3 sections)
3. Sections with clear headers
4. Cross-references to other files where relevant

---

Thank you for helping make this training repository better for everyone. Every contribution — no matter how small — is genuinely appreciated. 🙏
