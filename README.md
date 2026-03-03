# Agentforce Agent Builder

A Claude Code Skill for building Salesforce Agentforce agents using Agent Script DSL.

## What it does

Full lifecycle automation for Agentforce agent creation:

1. **Discover** — Gather requirements via conversation, save as structured config
2. **Generate** — Produce a complete `.agent` file following best practices
3. **Validate** — Check syntax via SF CLI with intelligent error diagnosis
4. **Publish** — Deploy to Salesforce org
5. **Guide** — Post-publish GUI checklist (Data Library wiring, activation)
6. **Test** — Generate and run test scenarios

## Prerequisites

- Claude Code CLI
- Salesforce CLI (`sf`) v2.113.6+
- Authenticated Salesforce org with Einstein and Agentforce enabled
- SFDX project (`sfdx-project.json`)

## Installation

Add this plugin to your Claude Code environment:

```bash
# From the plugin directory
claude plugins add ./agentforce-agent
```

## Usage

```bash
# Step 1: Describe your agent
/agentforce-agent discover --name my_faq_agent

# Step 2: Generate the .agent file
/agentforce-agent generate --name my_faq_agent

# Step 3: Validate syntax
/agentforce-agent validate --name my_faq_agent --org MyOrg

# Step 4: Publish to org
/agentforce-agent publish --name my_faq_agent --org MyOrg

# Step 5: Follow GUI checklist
/agentforce-agent guide --name my_faq_agent

# Step 6: Test
/agentforce-agent test --name my_faq_agent --org MyOrg
```

## Agent Types Supported

| Type | Description | Template |
|------|-------------|----------|
| FAQ-only | Knowledge base Q&A, no external data | `faq-only.agent` |
| Transactional | Data lookups via Flows/Apex, no KB | `transactional.agent` |
| Mixed | Both FAQ and transactional topics | `puntos-colombia.agent` |

## Key Features

- **Self-contained knowledge**: All Agent Script DSL syntax, anti-patterns, and prompt engineering patterns are bundled in the skill's reference documents
- **Battle-tested patterns**: Encodes learnings from 28 sessions of production agent development
- **Anti-pattern detection**: Suggests fixes for 20 known Agent Script pitfalls
- **Intelligent generation**: Applies tiered confidence, rotating closings, anti-escalation guardrails automatically
- **Idempotent**: Won't overwrite existing agent files without warning

## Compatibility

- Agent Script DSL: Spring '26 Beta
- Salesforce CLI: 2.113.6+
- Claude Code: Latest

## License

MIT
