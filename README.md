# ketangpai-auto-answer

Codex skill for automating quiz answering on `ketangpai.com` from an existing logged-in Chrome session.

This repository is structured as a single installable skill:

- `SKILL.md`: core skill instructions and workflow
- `agents/openai.yaml`: Codex UI metadata

## Requirements

- Codex desktop with the bundled Chrome control plugin available
- An existing Chrome session already logged into `ketangpai.com`

## Install

Install from GitHub with Codex's bundled installer script:

```powershell
python C:\Users\lzy\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py --repo 3217602760lzy/ketangpai-auto-answer --path . --name ketangpai-auto-answer
```

Manual install also works:

1. Clone or download this repository.
2. Copy the repository folder to `~/.codex/skills/ketangpai-auto-answer`.
3. Restart Codex.

## Use

Invoke the skill explicitly in Codex:

```text
Use $ketangpai-auto-answer to answer this ketangpai quiz.
```

Typical workflow inside the skill:

1. Open the quiz in the user's logged-in Chrome session.
2. Traverse all questions and capture question text, type, and options.
3. Derive an answer key.
4. Revisit each question and compare current selections with target answers.
5. Click only mismatched options.
6. Wait for the save-success toast after each answer.
7. Run a final verification pass before submission.

## Repository URL

[https://github.com/3217602760lzy/ketangpai-auto-answer](https://github.com/3217602760lzy/ketangpai-auto-answer)
