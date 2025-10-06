# Contributing to Kubernetes for Students

Thanks for considering contributing! This repo is built by students, for students.

## How to Contribute

### 1. Report Issues
If you find problems:
- Bugs in code or YAML
- Typos or grammar errors
- Experiments that don't work
- Confusing explanations

Create an issue with appropriate labels.

### 2. Suggest Improvements
Have ideas for:
- New chapters
- More relatable analogies
- More interesting challenges
- Useful tools or resources

Open an issue for discussion before creating large PRs.

### 3. Submit Pull Requests

**Good PR:**
- Focused on one thing
- Has context for why this change is needed
- Tested (if it's code or YAML)
- Follows existing style

**PRs that will be rejected:**
- Changes writing style to be overly formal
- Adds emojis everywhere (we keep it clean)
- Copy-paste from official docs without adaptation
- Breaking changes without prior discussion

## Writing Style Guidelines

### Voice & Tone
- **Casual but professional** - Imagine explaining to a smart college friend, not to a professor or a kid
- **No condescending** - Don't be patronizing, assume readers have basic intelligence
- **No emojis** - We avoid emoticons, use words instead
- **English only** - All content should be in English for broader accessibility

### Structure
Each chapter must have:
1. Quick intro (why this matters)
2. Analogy or story
3. Technical explanation
4. Hands-on experiment
5. Challenge exercise

### Code & YAML
- Always include comments
- Use realistic examples (not foo/bar everywhere)
- Tested on minikube minimum
- Include expected output

## Development Setup

```bash
# Fork the repo
# Clone your fork
git clone https://github.com/YOUR_USERNAME/kubernetes-for-students.git

# Create branch
git checkout -b feature/your-feature-name

# Make changes
# Test changes

# Commit
git commit -m "Add: short description"

# Push
git push origin feature/your-feature-name

# Create PR
```

## Testing Your Changes

Before submitting PR:
- [ ] Test all commands and YAML in edited chapters
- [ ] Check for typos and grammar
- [ ] Ensure markdown formatting is consistent
- [ ] Screenshots (if needed) are clear and helpful

## Questions?

Open an issue with "question" label or reach out to maintainers.

Let's make Kubernetes learning less painful for everyone!
