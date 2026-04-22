# Qodo Merge (PR Agent) Documentation

## Overview
Qodo Merge (formerly PR Agent) is an AI-powered tool designed to automate and enhance the Pull Request (PR) review process. It acts as a virtual senior engineer, providing code reviews, actionable suggestions, auto-generated descriptions, and documentation.

---

## Infrastructure & Setup Files

### 1. `docker-compose.yml`
This file is the backbone of your local Qodo Merge deployment. Its primary purposes are:
* **Container Orchestration:** It pulls the official `codiumai/pr-agent` image and runs it as a background service (`pr-agent-service`).
* **Environment Variables:** It injects crucial API keys (like `GITHUB_TOKEN` and `GROQ_API_KEY`) and specifies the AI model (e.g., `llama-3.3-70b-versatile`) so the tool can communicate with GitHub and the LLM provider.
* **Volume Mounting:** It mounts the `configuration.toml` file into the container at `/app/pr_agent/settings/.secrets.toml` so that your custom settings safely override the tool's defaults without breaking the core configuration.
* **Continuous Execution:** Using the `tail -f /dev/null` entrypoint, it keeps the container alive asynchronously, allowing you to run `docker exec` commands on-demand.


### 2. `configuration.toml`
This file acts as the AI's brain and behavior controller. Its primary purposes are:
* **Rule Customization:** It allows you to disable noisy default outputs (like effort estimations or default security labels) by setting flags to `false`.
* **Prompt Engineering:** Through the `extra_instructions` field, you can give the AI specific rules (e.g., "Format response in bullet points", "Focus heavily on security", "Ignore missing comments").
* **Output Tuning:** You can dictate how many code suggestions the AI should generate (`num_code_suggestions`) and limit the maximum number of findings (`num_max_findings`) to keep reviews concise and relevant.

---

## The 7 Core Commands

You can run these commands against any Pull Request by replacing the `<command>` and the `pr_url` in the following base command:
```powershell
docker exec -it pr-agent-service pr-agent --pr_url "https://github.com/USERNAME/REPO/pull/NUMBER" <command>
```

### 1. `review` (General PR Quality Check)
Performs a comprehensive sweep of your PR. It provides a summary, estimates the effort needed to review, and points out major architectural or logical flaws.
* **Best used when:** You just opened a PR and want a high-level sanity check, or you are reviewing someone else's PR and need a quick summary of what changed.

### 2. `improve` (Actionable Code Suggestions)
Unlike `review` which gives general feedback, `improve` targets specific lines of code. It scans for bugs, performance issues, and best practices, and leaves **commit-ready code snippets** as inline comments on GitHub. You can click "Commit suggestion" directly in the GitHub UI.
* **Best used when:** You want the AI to act like a senior engineer pair-programming with you and actively fixing your code before human reviewers see it.

### 3. `describe` (Auto-write PR Descriptions)
Reads the diff of your code and automatically writes a highly detailed PR Title and PR Description. It summarizes the exact files changed and the business logic applied.
* **Best used when:** You made a lot of changes and don't want to spend 15 minutes manually writing a summary for your coworkers.

### 4. `ask` (Chat with your PR)
Turns your PR into a ChatGPT-like session with context. You can ask specific questions about the code changes, and the AI will answer based purely on the diff.
* **Best used when:** You are reviewing a massive PR and don't understand a specific design choice, or want the AI to check for a specific edge case.
* **Example:** `... ask "Does this PR introduce any thread-safety issues?"`

### 5. `add_docs` (Auto-generate Documentation)
Scans your PR for newly added functions, classes, or methods that lack documentation, and generates proper docstrings (like Python Docstrings, JSDoc, etc.) as inline code suggestions.
* **Best used when:** You finished coding the logic but haven't written the documentation yet.

### 6. `reflect` (AI questions the Author)
Instead of the AI giving answers, the AI plays the role of a strict reviewer. It reads the PR and asks *you* (the author) questions about potential edge cases, missing test coverage, or unhandled exceptions.
* **Best used when:** You want to prepare for a tough code review and ensure you haven't forgotten any edge cases before involving human reviewers.

### 7. `update_changelog` (Maintain Release Notes)
If your repository uses a `CHANGELOG.md` file, this command will read the PR and automatically suggest the exact markdown to add to the changelog.
* **Best used when:** You are finalizing a PR and need to quickly generate accurate release notes based on the actual code diff.

---

## Additional Advanced Commands

In addition to the 7 core commands, Qodo Merge supports extra capabilities for advanced workflows:

### 8. `generate_labels` (Auto-labeling)
Automatically analyzes the PR diff and creates GitHub labels (like `bug`, `enhancement`, `security`, `dependencies`) and assigns them to the PR.
* **Best used when:** You want to maintain an organized repository without manually tagging PRs.

### 9. `similar_issue` (Catch Recurring Bugs)
If your PR is linked to an issue, it searches your repository for past similar issues.
* **Best used when:** You want to prevent merging fixes for problems that have already been supposedly fixed previously.

### 10. `ask_line` (Line-specific Questions)
Allows you to ask the AI a question about a specific line of code within the PR diff, rather than the whole PR.

---

## Completely Customizing the Review Output

If the default AI review format doesn't match your team's style, you can **100% customize** how the review looks. Because Qodo Merge uses Large Language Models (like Llama-3.3 or GPT-4), you can enforce strict, template-based formatting via prompt engineering.

### How to use Custom Templates
To change the review structure, you must update your `configuration.toml` file.

1. **Disable default sections** (to remove noisy tables and UI elements you don't want).
2. **Provide your own Markdown template** inside the `extra_instructions` block.

#### Example `configuration.toml` configuration:
```toml
[pr_reviewer]
# 1. Turn off default tables and summaries:
require_tests_review = false
require_estimate_effort_to_review = false
require_security_review = false
require_ticket_analysis_review = false
enable_intro_text = false
enable_review_labels_security = false
enable_review_labels_effort = false

# 2. Provide the template you want the AI to strictly follow:
extra_instructions = \"\"\"
You are a senior code reviewer. You MUST format your entire review EXACTLY matching the markdown template below. Do not add any extra sections.

# 🚀 Quick Summary
[Write 2 sentences summarizing the PR here]

# 🔬 Code Quality & Security
* [Bullet point 1 about quality/security]
* [Bullet point 2 about quality/security]

# 💡 Suggested Changes
[List exactly 3 actionable changes the developer should make. If the code is perfect, just write "Looks good to me!"]

# 📊 Final Verdict
[Write either "Mergeable", "Needs Changes", or "Critical Fixes Required"]
\"\"\"
```

Since the `configuration.toml` file is dynamically mounted using your `docker-compose.yml`, saving changes to this file applies them **instantly** to the next `review` command!

---

## Pro-Tip: Running via GitHub Webhooks
Currently, execution is done manually in the terminal. However, the `docker-compose.yml` mounts port `3000`, meaning Qodo Merge is running a web server! If you configure a **GitHub Webhook** to point to your machine's port 3000 (using a tool like Ngrok for local testing), you can trigger these commands directly from GitHub. 

Simply type a comment on your PR saying `/improve` or `/describe`, and the AI will automatically fetch the diff, analyze it, and reply!


