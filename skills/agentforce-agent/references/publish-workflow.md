# Agent Script Publish Workflow

> Valid for: Agent Script DSL (Spring '26 Beta)

## Prerequisites

- SF CLI 2.113.6+ installed (`sf --version`)
- Einstein enabled in org (Setup > Einstein > Enable)
- Agentforce enabled in org
- Authenticated to target org (`sf org login web --alias <alias>`)
- SFDX project with `sfdx-project.json`

## CLI Commands

### Validate Syntax
```bash
sf agent validate authoring-bundle --api-name <agent-name> --target-org <org-alias>
```
Checks DSL syntax. Returns 0 errors on success. Does NOT deploy anything.

### Publish (Creates Draft)
```bash
sf agent publish authoring-bundle --api-name <agent-name> --target-org <org-alias>
```
Generates and deploys: Bot + BotVersion + GenAiPlannerBundle + GenAiPlugin(s) + GenAiFunction(s).
Creates a **draft version** — does NOT auto-activate.

### Activate
```bash
sf agent activate --api-name <agent-name> --target-org <org-alias>
```
Activates the latest committed version. Requires `default_agent_user` set to real org email.

### Preview / Test
```bash
# Start interactive preview session
sf agent preview start --api-name <agent-name> --target-org <org-alias>

# With real Flow/Apex execution (not mocked)
sf agent preview start --api-name <agent-name> --target-org <org-alias> --use-live-actions

# Send test message
sf agent preview send --session-id <session-id> --message "test message"

# End session
sf agent preview end --session-id <session-id>
```
Session traces saved to `.sfdx/agents/<agent-name>/sessions/<session-id>/`.

## Required GUI Steps (Cannot Be Automated)

### 1. Wire Data Library (one-time per agent, persists across republishes)
1. Open Salesforce Setup
2. Navigate to: Setup > Agents > [Agent Name] > Open in Agentforce Builder
3. Go to: Settings > Knowledge > Add Data Library
4. Select your Data Library > Save

**BLOCKING**: Without this step, the agent will NOT use knowledge base. FAQ answers will be generic with no grounding.

### 2. Commit Version
1. In Agentforce Builder, find the new draft version
2. Click "Commit" to finalize the draft
3. This creates a restore point

### 3. Activate Agent
- Via CLI: `sf agent activate --api-name <agent-name>`
- Via GUI: Setup > Agents > [Agent Name] > Activate

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Internal Error, try again later" | Managed action target (EmployeeCopilot__*) | Use `knowledge:` block instead |
| "Expected 3, but got 4" | Inconsistent indentation | Fix to 3-space indent throughout |
| "Invalid syntax after conditional" | `connections:` block present | Remove `connections:` block entirely |
| Publish succeeds but agent has no knowledge | Data Library not wired in GUI | Complete GUI Step 1 above |
| Activation fails | `default_agent_user` is placeholder | Set to real org user email |

## Typical Workflow

```
1. Edit .agent file locally
2. sf agent validate authoring-bundle --api-name <name>     # Check syntax
3. sf agent publish authoring-bundle --api-name <name>      # Deploy draft
4. GUI: Link Data Library (if first time or knowledge-based agent)
5. GUI: Commit version
6. sf agent activate --api-name <name>                      # Or GUI activate
7. sf agent preview start --api-name <name>                 # Test
```
