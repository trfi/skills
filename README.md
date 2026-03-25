# AI Agent Skills

A collection of custom skills for AI coding assistants (like Claude Code, Cursor, Antigravity, etc.) to enhance their capabilities. 

## Available Skills

| Skill | Description | Triggers |
|-------|-------------|----------|
| **[cv-resume](skills/cv-resume/SKILL.md)** | Generate professional, ATS-optimized CVs and resumes as HTML. | create, write, build CV/resume |
| **[dokploy-docker-compose](skills/dokploy-docker-compose/SKILL.md)** | Deploy and troubleshoot Docker Compose stacks on Dokploy (self-hosted VPS platform). Covers volume strategies, networking, env vars, init containers, healthchecks, and common errors. | Dokploy, deploy docker-compose to VPS, self-hosted deployment |

## Example Use Cases

### dokploy-docker-compose

- *"I want to self-host Plausible Analytics on Dokploy. Here's the docker-compose.yml from their GitHub - adapt it for Dokploy."*
- *"How do I expose my n8n instance at `n8n.mydomain.com` with SSL on Dokploy?"*
- *"I need to run two separate WordPress sites on the same Dokploy server."*

### cv-resume

- *"Help me write a resume for a senior backend engineer role. I've been at Stripe for 4 years working on payments infrastructure."*
- *"Here's my LinkedIn profile - turn it into a polished CV."*
- *"My resume looks boring. Can you redesign it and make it stand out while keeping it ATS-friendly?"*
- *"I'm switching from teaching to UX design - help me write a resume that positions my experience for the change."*

## Installation

### Option 1: Install All Skills
```bash
npx skills add trfi/skills
```

### Option 2: Install Specific Skills
```bash
# CV/Resume skill
npx skills add trfi/skills@cv-resume

# Dokploy Docker Compose skill
npx skills add trfi/skills@dokploy-docker-compose
bunx skills add trfi/skills@dokploy-docker-compose
```

### Manual Installation
Copy the skills you want to your agent's skills directory:

```bash
# Claude Code
cp -r skills/* ~/.claude/skills/

# Cursor, Antigravity
cp -r skills/* ~/.agents/skills/

# Project-level (Claude Code)
cp -r skills/* .claude/skills/
```

## How to use

Once installed, your AI assistant will inherently understand how to execute the given skill based on the context and triggers outlined in its `SKILL.md`. 
For example, just tell your agent to "help me make a resume" and it will invoke the `cv-resume` skill.
