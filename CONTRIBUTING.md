# Contributing to DevOps Zero to Hero

Thank you for taking the time to contribute. This project is for beginners, so keeping things clear and practical is the most important guideline.

---

## What You Can Contribute

- Fix typos, grammar, or unclear explanations
- Improve or add exercises
- Add missing cheatsheets or resources
- Update outdated commands or tool versions
- Translate content to other languages

## What Does Not Belong Here

- Interview question lists (out of scope — this project is hands-on only)
- Theory-heavy content without practical exercises
- Proprietary tool references (stick to open-source or free-tier tools)
- Content that requires paid accounts beyond AWS Free Tier

---

## How to Contribute

1. Fork the repository
2. Create a branch: `git checkout -b fix/typo-in-day-03` or `feat/add-nginx-exercise`
3. Make your changes
4. Open a pull request against `main`

### Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| Bug fix / typo | `fix/<short-description>` | `fix/wrong-chmod-command` |
| New content | `feat/<short-description>` | `feat/add-helm-day` |
| Cheatsheet | `cheatsheet/<tool>` | `cheatsheet/kubectl` |

### Commit Message Format

Keep it short and describe what changed:

```
fix: correct chmod example in day-02
feat: add nginx reverse proxy exercise to week-2
docs: update terraform backend instructions
```

---

## Style Guide

- **Write for a beginner.** Assume the reader has never done this before.
- **Show, don't tell.** Every concept should have a working command or code block.
- **Explain why, not just how.** One line of context is worth more than three lines of what.
- **Use placeholder values consistently.** Domain names use `<your-domain>`, usernames use `<your-username>`, and bucket names use `YOURNAME` as a suffix.
- **Keep commands copy-pasteable.** Avoid line wrapping that breaks commands.
- **Test your examples.** If you add a command, run it and confirm it works.

---

## Placeholder Conventions

| What | Placeholder |
|------|-------------|
| Domain name | `<your-domain>` (e.g., example.com) |
| Docker Hub username | `<your-dockerhub-username>` |
| AWS account ID | `123456789012` |
| S3 bucket name | `my-bucket-YOURNAME` |
| EC2 key pair | `my-ec2-key` |
| IP address | `203.0.113.5` (TEST-NET from RFC 5737) |

---

## Questions

Open a GitHub Issue with the `question` label.

---

Licensed under [MIT](./LICENSE).
