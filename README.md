# AI Agent Skills

A collection of custom skills for AI coding assistants (like Claude Code, GitHub Copilot) to enhance their capabilities. 

## Available Skills

| Skill | Description | Triggers |
|-------|-------------|----------|
| **[cv-resume](skills/cv-resume/SKILL.md)** | Generate professional, ATS-optimized CVs and resumes as HTML. | create, write, build CV/resume |

*More skills will be added in the future!*

## Installation

### Manual Installation
Copy the skills you want to your agent's skills directory:

```bash
# Claude Code
cp -r skills/* ~/.claude/skills/

# GitHub Copilot
cp -r skills/* ~/.copilot/skills/

# Project-level (Claude Code)
cp -r skills/* .claude/skills/
```

## How to use

Once installed, your AI assistant will inherently understand how to execute the given skill based on the context and triggers outlined in its `SKILL.md`. 
For example, just tell your agent to "help me make a resume" and it will invoke the `cv-resume` skill.
