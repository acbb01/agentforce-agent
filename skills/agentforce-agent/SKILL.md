---
name: agentforce-agent
description: >
  Build Salesforce Agentforce agents using Agent Script DSL. Full lifecycle:
  discover requirements, generate .agent files, validate, publish, and test.
  Triggers on: "build agentforce agent", "create agent script",
  "generate .agent file", "agentforce agent", "new agentforce agent".
  Do NOT trigger on generic "create agent" or non-Salesforce agent tasks.
argument-hint: "<mode> [--name <agent-name>] [--config <path>] [--org <alias>]"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - Agent
---

# Agentforce Agent Builder

Full lifecycle skill for building Salesforce Agentforce agents using Agent Script DSL.
Discover requirements, generate `.agent` files, validate, publish, and test — all from
the command line.

---

## Setup: Resolve Paths

Before executing any command, resolve the skill's directory paths. These paths are
required by every command to locate references, examples, and templates.

1. Use Glob to find this file:
   ```
   Glob pattern: **/agentforce-agent/SKILL.md
   ```
2. Set `SKILL_DIR` to the parent directory of the matched SKILL.md file.
3. Set `REFERENCES` to `SKILL_DIR/references`.
4. Set `EXAMPLES` to `SKILL_DIR/references/example-agents`.

All subsequent file reads use these resolved paths. Never hardcode absolute paths to
the skill directory.

---

## Parse Arguments and Dispatch

Read `$ARGUMENTS` and extract the command (first positional word) and any flags.

### Command Routing

| Input                  | Command           |
|------------------------|-------------------|
| (empty), `help`        | COMMAND: help     |
| `discover`             | COMMAND: discover |
| `generate`             | COMMAND: generate |
| `validate`             | COMMAND: validate |
| `publish`              | COMMAND: publish  |
| `guide`                | COMMAND: guide    |
| `test`                 | COMMAND: test     |

If the input does not match any command, display the help text and ask the user to
choose a valid command.

### Flag Parsing

Parse the following flags from `$ARGUMENTS`. All are optional unless noted otherwise
in a specific command.

| Flag                     | Description                                  | Default |
|--------------------------|----------------------------------------------|---------|
| `--name <agent-name>`   | Agent developer name (snake_case)            | none    |
| `--config <path>`       | Path to discovery config JSON                | none    |
| `--org <alias>`         | Salesforce org alias                         | none    |
| `--channel <type>`      | Deployment channel: web, whatsapp, voice     | web     |
| `--language <locale>`   | Conversational language locale               | es      |
| `--brand-voice <path>`  | Path to brand voice document                 | none    |

---

## COMMAND: help

Display usage information and exit. Do not prompt the user for anything.

```
Agentforce Agent Builder — Full Lifecycle Skill

Usage: /agentforce-agent <mode> [options]

Modes:
  discover    Gather requirements via conversation → saves JSON config
  generate    Generate .agent file from config + best practices
  validate    Validate syntax via SF CLI
  publish     Publish to Salesforce org
  guide       Post-publish GUI checklist
  test        Generate and run test scenarios

Options:
  --name <name>         Agent developer name (snake_case)
  --config <path>       Path to discovery config JSON
  --org <alias>         Salesforce org alias
  --channel <type>      Channel: web, whatsapp, voice (default: web)
  --language <locale>   Language: es, en, pt, etc. (default: es)
  --brand-voice <path>  Path to brand voice document

Typical workflow:
  1. /agentforce-agent discover --name my_agent
  2. /agentforce-agent generate --name my_agent
  3. /agentforce-agent validate --name my_agent --org MyOrg
  4. /agentforce-agent publish --name my_agent --org MyOrg
  5. /agentforce-agent guide --name my_agent
  6. /agentforce-agent test --name my_agent --org MyOrg
```

---

## COMMAND: discover

**Purpose**: Gather requirements through a structured conversation and save them as
a JSON config file that drives generation. Proactively queries the Salesforce org
to pre-fill configuration choices and reduce back-and-forth.

### Step 0: Org Probe

Before asking the user anything, gather org data via the SF CLI. Run these commands
**in parallel** (all are independent):

```bash
# 1. Check sf CLI is installed
sf --version

# 2. List connected orgs
sf org list --json
```

**If `sf` is not installed** (command not found):
- Warn: "Salesforce CLI (sf) not found. Org data will not be pre-filled. You can install it from https://developer.salesforce.com/tools/salesforcecli"
- Continue without org data — all subsequent steps work without it, just with less pre-filling.

**If `sf` is installed**, determine the target org:
- If `--org` was provided, use that alias.
- Otherwise, parse `sf org list --json` output. If a default org exists, use it.
- If no org is available, warn: "No connected Salesforce org found. Run `sf org login web --alias <alias>` to connect one. Continuing without org data."
- Continue without org data.

**If a target org is available**, run these 4 queries **in parallel**:

```bash
# 3a. Active users (for default_agent_user picker)
sf data query --query "SELECT Id, Name, Email, IsActive FROM User WHERE IsActive = true ORDER BY Name LIMIT 50" --target-org <alias> --json

# 3b. Existing agents (naming conflict check)
sf data query --query "SELECT DeveloperName, MasterLabel FROM BotDefinition ORDER BY DeveloperName" --target-org <alias> --json

# 3c. Knowledge availability
sf data query --query "SELECT Id, Name FROM KnowledgeArticleVersion WHERE PublishStatus = 'Online' LIMIT 1" --target-org <alias> --json

# 3d. Active Flows (for action topic suggestions)
sf data query --query "SELECT ApiName, Label, ProcessType FROM FlowDefinitionView WHERE IsActive = true AND ProcessType IN ('AutoLaunchedFlow', 'Flow') ORDER BY Label" --target-org <alias> --json
```

Each query may fail independently (permissions, object not enabled, etc.). If a query
fails, warn per query and continue with whatever succeeded. Store all results in
memory for use in Steps 2-4.

**Present org summary** before proceeding:

```
## Org Context

| Data Point | Value |
|-----------|-------|
| SF CLI | v2.X.X |
| Target org | <alias> (<username>) |
| Active users | <N> found |
| Existing agents | <list or "none"> |
| Knowledge articles | <N> online (or "not enabled") |
| Active Flows | <N> found |

This data will help pre-fill your agent configuration.
```

If no org data was gathered, show a simpler message:
```
## Org Context
No org data available. Configuration will require manual input for user email and org details.
```

### Step 1: Check for Existing Config

If `--config` was provided:
1. Read the file at the given path.
2. Parse the JSON and present a summary of the config to the user.
3. Ask: "This config already exists. Use it as-is, or modify it?"
4. If the user says use as-is, skip directly to Step 5 (save/confirm).
5. If modify, proceed to Step 2 but pre-fill answers from the existing config.

If `--config` was NOT provided but `--name` was provided:
1. Look for `<sfdx-project-root>/agent-config/<name>-discovery.json`.
2. If found, follow the same flow as above.
3. If not found, proceed to Step 2.

### Step 2: Brief Capture

If org data was gathered in Step 0, include it as context before the question:

> I've connected to your org **<alias>** and found **<N>** active users,
> **<N>** existing agents, and **<N>** knowledge articles. I'll use this
> to pre-fill your configuration.

If no org data was gathered, skip the context line.

Use AskUserQuestion to ask ONE main question. Do not split this into multiple
questions. The user should be able to provide a rich description in one pass.

> Describe the agent you want to build in 2-5 sentences. Include:
> - What business or program it serves
> - What it should help users with (FAQ questions, transactions, both)
> - What language it should respond in
> - Any specific tone or vocabulary requirements

Wait for the user's response.

If `--brand-voice` was provided, Read the brand voice document. Extract register
(tu/usted), tone descriptors, vocabulary terms, and emoji policy. These will feed
into the assumptions in Step 3.

### Step 3: Generate Assumptions

From the user's description (and brand voice document if provided), generate a
structured summary with EXPLICIT assumptions. Every assumption must be visible so
the user can correct it. **Pre-fill with org data where available.**

Present the following:

```
## Agent Summary

**Business**: [inferred business or program name]
**Agent Name**: [from --name flag, or inferred from description in snake_case]
**Type**: [FAQ-only / Transactional / Mixed]
**Language**: [inferred from description, or from --language flag]
**Channel**: [from --channel flag, or inferred from description]
```

**Agent name conflict check** (if org data available):
- Compare the agent name against existing BotDefinition DeveloperNames from Step 0.
- If no conflict: `No conflict with existing agents: [list of existing agent names]`
- If conflict: `CONFLICT: Agent "<name>" already exists in the org. Choose a different name or overwrite.`

```
### Topics Detected
1. [Topic name] — [one-line description] — Type: [FAQ / Action]
   - If Action: Flow target? Required inputs? Expected outputs?
2. [Topic name] — [one-line description] — Type: [FAQ / Action]
   ...
```

**Action topic suggestions** (if org data available and user described transactional
needs): Cross-reference the user's description against active Flows from Step 0.
If plausible matches exist, present them:

```
### Detected Flows (potential action targets)
| Flow API Name | Label | Type |
|---------------|-------|------|
| Get_Customer_Balance | Check Balance | AutoLaunchedFlow |
| Process_Return | Process Return | AutoLaunchedFlow |

Which of these should the agent invoke? (Select or skip)
```

Use AskUserQuestion with the matching Flows as options (plus "None of these").

```
### Brand Voice (Inferred)
- Register: [tu / usted / formal / informal]
- Tone: [warm-professional / friendly / corporate / other]
- Vocabulary: [domain-specific terms detected, if any]
- Emoji policy: [no emojis / minimal / expressive]

### Escalation Policy
- Trigger: [explicit user request only / frustration detection / both]
- Farewell message: [inferred warm handoff message in target language]

### Knowledge Base
- Enabled: [yes / no — yes if any FAQ topics detected]
- Citations: [false — default, rarely changed]
- Source: [will be determined in Step 4]
```

**Salesforce Configuration — default_agent_user**:

If org data includes active users, present a picker using AskUserQuestion:
```
### Salesforce Configuration
Select the default agent user (the Salesforce user the agent runs as):
  1. admin@company.com (Admin User)
  2. service@company.com (Service User)
  3. agentforce@company.com (Agentforce User)
  [Other — type email]
```

If no org data, ask the user to type the email directly:
```
### Salesforce Configuration
- default_agent_user: [Required — provide the Salesforce user email the agent will run as]
- Target org: [from --org flag, or "not specified — will be needed for validate/publish"]
```

Present this summary to the user. Then ask:
"Review the assumptions above. What needs to change? Reply 'looks good' to confirm,
or describe corrections."

Iterate: apply corrections, re-present, and ask again until the user confirms.

### Step 4: Knowledge Strategy

Determine the knowledge source for the agent. Follow this priority order:

**1. File-based Data Library (preferred)**

Ask: "Do you have FAQ or knowledge content as files (.csv, .md, .txt, .pdf) that
the agent should use as its knowledge base?"

If yes:
- Ask for the file path or directory.
- If in an SFDX project, check common locations: `force-app/main/default/datalibraries/`,
  `kb/`, `data/`, and the project root for `.csv`, `.md`, `.txt` files.
- If files are found, confirm them with the user.
- If no files exist but the user wants file-based KB, offer to generate a template
  FAQ CSV:
  ```csv
  Question,Answer,Category
  "How do I earn points?","You earn points by...","General"
  "What is the exchange rate?","The exchange rate is...","Rewards"
  ```
  Generate 3-5 example rows based on the topics detected in Step 3.
  Save to `<project-root>/kb/<agent-name>-faq.csv`.
- Set in config: `knowledge.source: "data-library-file"`, `knowledge.dataLibraryPath: "<path>"`

**2. Org Knowledge (fallback)**

If the user has no files but org data from Step 0 shows Knowledge articles exist:
- Report: "Your org has <N> published Knowledge articles. The agent can use org
  Knowledge as its source instead of file-based Data Library."
- Ask: "Use org Knowledge articles as the knowledge source?"
- If yes, set: `knowledge.source: "org-knowledge"`

**3. No knowledge**

If neither file-based nor org Knowledge is available or desired:
- Set: `knowledge.enabled: false`
- Warn: "Without a knowledge source, FAQ topics will produce generic answers with
  no grounding. Consider adding a Data Library before publishing."

**Important**: Regardless of the knowledge source chosen, the Data Library must still
be wired in the Agentforce Builder GUI after publishing (see COMMAND: guide, Step 1).
The `knowledge.source` field in the config records the *intended* source so the guide
command can provide specific instructions.

### Step 5: Save Config

1. Determine the SFDX project root by searching for `sfdx-project.json` in the
   current working directory and its parent directories.
2. If no SFDX project found, use the current working directory as the root.
3. Create directory `<root>/agent-config/` if it does not exist.
4. Write the confirmed config as JSON to `<root>/agent-config/<agent-name>-discovery.json`.

The JSON structure:

```json
{
  "agentName": "my_agent",
  "agentLabel": "My Agent",
  "business": "Acme Corp",
  "type": "mixed",
  "language": "es",
  "channel": "web",
  "brandVoice": {
    "register": "usted",
    "tone": "warm-professional",
    "vocabulary": ["puntos", "redimir"],
    "emojiPolicy": "no emojis"
  },
  "escalation": {
    "trigger": "explicit user request",
    "farewellMessage": "Lo conecto con un asesor. Un momento por favor."
  },
  "knowledge": {
    "enabled": true,
    "citationsEnabled": false,
    "source": "data-library-file",
    "dataLibraryPath": "kb/my_agent-faq.csv"
  },
  "salesforce": {
    "defaultAgentUser": "service@company.com",
    "targetOrg": "MyOrgAlias",
    "orgProbe": {
      "cliVersion": "2.113.6",
      "userCount": 47,
      "existingAgents": ["agent_1", "agent_2"],
      "knowledgeArticles": 125,
      "activeFlows": ["Get_Customer_Balance", "Process_Return"]
    }
  },
  "topics": [
    {
      "name": "FAQ_Topic",
      "label": "Frequently Asked Questions",
      "description": "Answer common questions about the program",
      "type": "faq",
      "sampleQuestions": ["How do I earn points?", "What is the exchange rate?"]
    },
    {
      "name": "Balance_Topic",
      "label": "Check Balance",
      "description": "Look up the customer's current points balance",
      "type": "action",
      "flowTarget": "flow://Get_Customer_Balance",
      "inputs": [{"name": "document_number", "type": "string", "description": "Customer ID number"}],
      "outputs": [{"name": "balance", "type": "number", "description": "Current points balance"}]
    }
  ]
}
```

The `salesforce.orgProbe` field is populated from Step 0 data. If no org was probed,
omit this field entirely. The `knowledge.source` and `knowledge.dataLibraryPath`
fields come from Step 4. If source is `"org-knowledge"`, omit `dataLibraryPath`.

Report to the user:

> Discovery complete. Config saved to `<absolute-path>`.
> Next: `/agentforce-agent generate --name <agent-name>`

---

## COMMAND: generate

**Purpose**: Generate a complete `.agent` file from the discovery config, reference
materials, and best practices.

### Step 0: Load Inputs

1. Load discovery config from `--config` (if provided) or auto-detect at
   `<sfdx-project-root>/agent-config/<name>-discovery.json`.
2. If neither `--config` nor `--name` is provided, or if the file is not found,
   report the error:
   > No config found. Run `/agentforce-agent discover --name <name>` first.
   Then stop.

### Step 1: Load References (Progressive)

Read these files from the REFERENCES directory in order. Each file provides
essential context for generation. If a file is missing, warn but continue.

1. `REFERENCES/agent-script-syntax.md` — DSL syntax rules, block structure, indentation
2. `REFERENCES/anti-patterns.md` — Common mistakes and their fixes
3. `REFERENCES/prompt-engineering.md` — System instruction patterns, brand voice encoding

### Step 2: Select and Load Template

Based on the `type` field in the discovery config, select the appropriate example
agent to use as a structural template:

| Agent Type    | Template File                         |
|---------------|---------------------------------------|
| faq-only      | `EXAMPLES/faq-only.agent`             |
| transactional | `EXAMPLES/transactional.agent`        |
| mixed         | `EXAMPLES/puntos-colombia.agent`      |

Read the selected template. This provides the structural skeleton — block ordering,
indentation patterns, and DSL conventions. The content will be replaced with values
from the discovery config.

If the template file does not exist, warn the user but proceed with generation
using the syntax reference alone.

### Step 3: Generate the .agent File

Generate the file section by section in this EXACT order. Do not reorder blocks.
Agent Script is order-sensitive.

#### 3a. config block

```
config:
   api_name: <agentName from config>
   label: "<agentLabel from config>"
   type: "customer"
   default_agent_user: "<salesforce.defaultAgentUser from config>"
```

#### 3b. variables block

Generate variables in this order:

1. **Linked messaging variables** (no defaults, no type declaration):
   ```
   variables:
      $Contact.FirstName:
      $Contact.LastName:
   ```

2. **Mutable variables for each action topic's data collection**:
   For each action topic, create a variable for each required input:
   ```
      customer_document_number:
         type: "string"
         description: "Customer identification number"
         default: ""
   ```

3. **Escalation control variable**:
   ```
      escalation_requested:
         type: "boolean"
         description: "Flag indicating the user has explicitly requested human agent transfer"
         default: False
   ```

#### 3c. system block

Encode the brand voice, language rules, and behavioral guardrails into a QUOTED
STRING. This is critical: `system.instructions` MUST be a quoted string, never
a template (`-> |`).

```
system:
   instructions: "You are [agent label], a virtual assistant for [business].

   LANGUAGE RULES:
   - Always respond in [language].
   - Use [register] register consistently.
   - Tone: [tone description].
   - [Vocabulary rules if any].
   - [Emoji policy].

   BEHAVIORAL RULES:
   - Answer only questions within your defined topics.
   - If you are not confident in your answer, say so honestly.
   - Never fabricate information. If unsure, offer to connect with a human agent.
   - Do not offer to escalate proactively. Only escalate when the user explicitly requests it.

   CONFIDENCE TIERS:
   - HIGH: Answer directly and conversationally.
   - MEDIUM: Answer with a softener ('Based on what I know...') and offer more help.
   - LOW: Acknowledge the question, state you are not sure, and offer escalation."

   welcome_message: "[Welcome message in target language]"
   error_message: "[Error message in target language]"
```

Generate the welcome_message and error_message in the target language from the config.

#### 3d. knowledge block

Only include if `knowledge.enabled` is true in the config.

```
knowledge:
   citations_enabled: False
```

If `knowledge.citationsEnabled` is true in the config, set to `True` instead.

#### 3e. language block

```
language: "<language from config>"
```

#### 3f. start_agent topic_selector

Generate a topic selector that routes to all topics defined in the config, plus
the escalation and off-topic topics.

```
start_agent:
   topic_selector:
      routing:
         - @topic.FAQ_Topic: "User asks a common question about the program"
         - @topic.Balance_Topic: "User wants to check their points balance"
         - @topic.Escalation: "User explicitly requests to speak with a human agent"
            available when @variables.escalation_requested == True
         - @topic.Off_Topic: "User asks something outside the agent's scope"

      after_reasoning:
         -> if @variables.escalation_requested == True:
            -> @utils.transition to @topic.Escalation
```

Generate one routing entry per topic from the config, plus Escalation and Off_Topic.
The Escalation route has an `available when` guard on the escalation_requested variable.
The `after_reasoning` block checks the flag and forces transition if set.

#### 3g. Topic Blocks

Generate one topic block per topic from the config, plus mandatory Escalation and
Off_Topic topics.

**FAQ Topics** (type: "faq"):

```
topic FAQ_Topic:
   description: "Answer common questions about [topic description]"
   scope:
      in_scope:
         - "Questions about [scope item 1]"
         - "Questions about [scope item 2]"
      out_of_scope:
         - "Questions unrelated to [business]"

   reasoning:
      instructions:
         -> |
         Use knowledge base to answer the user's question.
         If the answer is found, respond naturally and vary your closing:
         - "Was there anything else I can help with?"
         - "Do you have any other questions?"
         - "Is there anything else you need?"

         If the knowledge base does not contain the answer:
         1. Rephrase the question internally and search again.
         2. If still not found, say honestly that you do not have that information
            and offer to connect with a human agent.
         3. If the user then confirms they want a human agent, set
            escalation_requested to True.

      actions:
         - setVariables:
            escalation_requested: True

      after_reasoning:
         -> if @variables.escalation_requested == True:
            -> @utils.transition to @topic.Escalation
```

**Action Topics** (type: "action"):

```
topic Balance_Topic:
   description: "Look up the customer's current points balance"
   scope:
      in_scope:
         - "Requests to check points balance"
         - "Questions about current balance"
      out_of_scope:
         - "Requests to redeem or transfer points"

   actions:
      Get_Customer_Balance:
         description: "Retrieve the customer's current points balance"
         inputs:
            document_number:
               description: "Customer identification number"
               collect_from_user: True
         outputs:
            balance:
               description: "Current points balance"
         target: "flow://Get_Customer_Balance"

   reasoning:
      instructions:
         -> |
         Help the user check their points balance.

         STATE 1 — No input yet:
         Ask for their identification number politely.

         STATE 2 — Input collected:
         Run @actions.Get_Customer_Balance with the provided document number.

         STATE 3 — Output received:
         Present the balance naturally. Offer additional help.

         If the action fails or returns an error, apologize and offer escalation.
         If the user requests a human agent, set escalation_requested to True.

      actions:
         - @actions.Get_Customer_Balance
         - setVariables:
            escalation_requested: True

      after_reasoning:
         -> if @variables.escalation_requested == True:
            -> @utils.transition to @topic.Escalation
```

For each action topic, generate:
- An `actions` definition block with `description`, `inputs` (with `collect_from_user: True`
  for user-provided data), `outputs`, and `target` from the config.
- A `reasoning.instructions` template with 3-state conditional logic.
- `reasoning.actions` listing the defined action plus `setVariables` for escalation.
- `after_reasoning` with escalation transition check.

**Escalation Topic** (always generated):

```
topic Escalation:
   description: "Transfer the conversation to a human agent when explicitly requested"
   scope:
      in_scope:
         - "User explicitly asks for a human agent"
         - "User says they want to talk to a person"
      out_of_scope:
         - "General complaints without transfer request"

   reasoning:
      instructions:
         -> |
         The user has requested to speak with a human agent.
         Deliver a warm farewell message, then escalate.

         Say: "[escalation farewell message from config]"

         Then execute the escalation.

      actions:
         - @utils.escalate

      after_reasoning:
         -> set @variables.escalation_requested = False
```

**Off-Topic Topic** (always generated):

```
topic Off_Topic:
   description: "Handle questions outside the agent's defined scope"
   scope:
      in_scope:
         - "Any question not covered by other topics"
      out_of_scope: []

   reasoning:
      instructions:
         -> |
         The user asked something outside your scope.
         Politely let them know what you CAN help with.
         List your available topics briefly.
         Do not attempt to answer the off-topic question.
         If they insist or get frustrated, offer escalation.

      actions:
         - setVariables:
            escalation_requested: True

      after_reasoning:
         -> if @variables.escalation_requested == True:
            -> @utils.transition to @topic.Escalation
```

### Critical Generation Rules

These rules are non-negotiable. Violating any of them produces agents that fail
validation or behave incorrectly at runtime.

1. **system.instructions is a QUOTED STRING**. Never use `-> |` template syntax.
   Only `reasoning.instructions` uses the template syntax.

2. **Actions must be DEFINED before referenced**. The `actions:` block at topic level
   defines them. The `reasoning.actions:` block references them. Do not reference
   an action that has not been defined in the same topic.

3. **Use 3-space indentation consistently**. Agent Script is whitespace-sensitive.
   Mixing 2-space and 3-space indentation causes cryptic compile errors.

4. **No `connections:` block**. It is listed in the spec but is in `future_recipes/`
   and causes compiler errors. Omit entirely.

5. **No `else if`**. Agent Script does not support `else if`. Use nested `if`/`else`
   inside the else branch instead.

6. **No templates in system.instructions or static descriptions**. The `-> |` pipe
   syntax is only valid inside `reasoning.instructions`.

7. **No `run` nesting deeper than 1 level**. Flatten complex logic instead.

8. **Linked variables have NO defaults**. Variables like `$Contact.FirstName` must
   not have a `default:` field.

9. **`after_reasoning` has NO pipes or templates**. Only deterministic `->` arrow
   syntax and variable operations.

10. **All prompts and descriptions are in ENGLISH**. The agent responds in the target
    language because the system instructions tell it to. But topic descriptions,
    action descriptions, scope definitions, and reasoning instructions are all
    written in English. This is a Salesforce Agentforce convention.

### Step 4: Determine Output Path

1. Find the SFDX project root (directory containing `sfdx-project.json`).
2. Target directory: `<project>/force-app/main/default/aiAuthoringBundles/<agent-name>/`
3. Create the directory if it does not exist.

If `<agent-name>.agent` already exists at that path:
- Warn the user: "An agent file already exists at this path. Writing to
  `<agent-name>.v2.agent` to avoid overwriting. Review and rename manually."
- Write to `<agent-name>.v2.agent` instead.

### Step 5: Write Files

Write two files:

1. The `.agent` file to the target directory.
2. The bundle metadata file `<agent-name>.bundle-meta.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<AiAuthoringBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <developerName>{{AGENT_NAME}}</developerName>
    <masterLabel>{{AGENT_LABEL}}</masterLabel>
</AiAuthoringBundle>
```

Replace `{{AGENT_NAME}}` with the agent's developer name and `{{AGENT_LABEL}}` with
the agent's label from the config.

### Step 6: Present Summary

Show a section-by-section summary of what was generated:

```
## Generation Summary

**Config**
- Agent name: <name>
- Default user: <email>
- Type: <type>

**Variables**
- Linked: $Contact.FirstName, $Contact.LastName
- Mutable: <list of mutable variable names>
- Escalation flag: escalation_requested

**System Instructions**
- First 2 lines of instructions...
- Language: <language>
- Register: <register>

**Knowledge**
- Enabled: yes/no
- Citations: true/false

**Topics**
| # | Name | Type | Has Actions |
|---|------|------|-------------|
| 1 | FAQ_Topic | FAQ | No |
| 2 | Balance_Topic | Action | Get_Customer_Balance |
| 3 | Escalation | Escalation | @utils.escalate |
| 4 | Off_Topic | Redirect | No |

**Output**
- Agent file: <absolute path>
- Bundle XML: <absolute path>
```

Ask: "Review the generated agent above. Want to modify any section before validation?"

If the user requests changes, apply them to the generated file using Edit, then
re-present the summary.

Report:

> Agent generated at `<absolute-path>`.
> Next: `/agentforce-agent validate --name <name> --org <alias>`

---

## COMMAND: validate

**Purpose**: Validate agent syntax using the Salesforce CLI before publishing.

### Step 1: Pre-flight Check

Run:
```bash
sf --version
```

If the command fails or `sf` is not found:
- Report: "Salesforce CLI (sf) is not installed or not in PATH. Install it from
  https://developer.salesforce.com/tools/salesforcecli and try again."
- Stop.

Verify that `--org` was provided. If not, ask the user for the org alias.

### Step 2: Validate

Run:
```bash
sf agent validate authoring-bundle --api-name <agent-name> --target-org <org-alias>
```

Capture stdout and stderr.

### Step 3: Handle Result

**If 0 errors (exit code 0)**:

Report:
> Validation passed with 0 errors.
> Next: `/agentforce-agent publish --name <name> --org <alias>`

**If errors (exit code non-zero)**:

1. Read `REFERENCES/anti-patterns.md` to load the known error patterns and fixes.
2. Parse the error output. Extract each error message and line number if available.
3. For each error, match against known anti-patterns and suggest a specific fix.
4. Present the errors and suggestions:

```
## Validation Errors

### Error 1: [error message]
- Line: [line number if available]
- Likely cause: [matched anti-pattern]
- Suggested fix: [specific fix]

### Error 2: ...
```

5. Do NOT auto-fix the errors. Present suggestions and let the user decide which
   fixes to apply. Auto-fixing can introduce subtle bugs in a whitespace-sensitive
   DSL.

6. After the user applies fixes (or asks you to apply specific ones), suggest
   re-running validate:
   > Fixes applied. Re-run: `/agentforce-agent validate --name <name> --org <alias>`

---

## COMMAND: publish

**Purpose**: Publish the agent to a Salesforce org. This creates or updates the
agent draft in the target org.

### Step 1: Confirm

Ask the user for explicit confirmation before proceeding:

> Ready to publish `<agent-name>` to org `<org-alias>`? This will create or update
> the agent draft. Type "yes" to continue.

Wait for explicit confirmation. Do not proceed on ambiguous responses.

### Step 2: Publish

Run:
```bash
sf agent publish authoring-bundle --api-name <agent-name> --target-org <org-alias>
```

Capture stdout and stderr.

### Step 3: Handle Result

**If success (exit code 0)**:

Report:
> Agent `<agent-name>` published successfully to org `<org-alias>`.

Automatically proceed to COMMAND: guide to show the post-publish checklist.

**If error (exit code non-zero)**:

1. Read `REFERENCES/anti-patterns.md` to check for known publish errors.
2. Parse the error output and diagnose the root cause.
3. Present the error and suggested fix to the user.
4. Common publish errors:
   - "Internal Error, try again later" — often caused by invalid action targets
     (e.g., referencing managed actions with custom inputs/outputs). Check action
     `target` values.
   - Timeout — the publish job may still be running server-side. Wait 60 seconds
     and check the agent status in Setup.
   - Authentication error — run `sf org login web --alias <alias>` to re-authenticate.

---

## COMMAND: guide

**Purpose**: Present an active post-publish checklist with blocking gates. Some
steps cannot be done via CLI and require the Salesforce GUI.

Read the discovery config to determine if the agent has knowledge enabled.

Present the following checklist, adapting based on agent configuration:

```
## Post-Publish Checklist for <agent-name>

### Step 1: Wire Data Library [BLOCKING — knowledge-based agents only]
   Open: Setup > Agents > <agent-name> > Open in Agentforce Builder
   Navigate: Settings > Knowledge > Add Data Library
   Select your Data Library > Save

   WARNING: Without this step, the agent will NOT use the knowledge base.
   FAQ answers will be generic with no grounding. This is the number one
   silent failure in Agentforce deployments.

### Step 2: Commit Version
   In Agentforce Builder > Commit the draft version.
   This creates a restore point you can roll back to.

### Step 3: Activate
   Option A (CLI):
     sf agent activate --api-name <agent-name> --target-org <org-alias>

   Option B (GUI):
     Setup > Agents > <agent-name> > Activate

### Step 4: Verify
   Run: /agentforce-agent test --name <agent-name> --org <org-alias>
```

Conditional notes:
- If the agent has NO knowledge block (transactional-only), note that Step 1 can
  be skipped: "This agent is transactional-only with no knowledge block. Step 1
  (Data Library) can be skipped."
- If `knowledge.source` is `"data-library-file"` in the config, add to Step 1:
  "Your config specifies a file-based Data Library at `<dataLibraryPath>`. Make sure
  this file has been uploaded to the Data Library in Setup before wiring it here."
- If `knowledge.source` is `"org-knowledge"`, add to Step 1:
  "Your config specifies org Knowledge articles as the source. Select the appropriate
  Knowledge base when wiring the Data Library."
- If `--org` was not provided, omit the CLI command in Step 3 and note that the
  org alias is needed.

---

## COMMAND: test

**Purpose**: Generate test scenarios from the discovery config and optionally
execute them using the SF CLI agent preview.

### Step 1: Load Discovery Config

Load from `--config` or auto-detect at `<sfdx-project-root>/agent-config/<name>-discovery.json`.

If not found:
> No config found. Run `/agentforce-agent discover --name <name>` first.

### Step 2: Generate Scenarios

For each topic in the config, generate test scenarios. Always include these
baseline scenarios regardless of topics:

| # | Scenario       | Type       | Test Message                    | Expected Behavior                    |
|---|----------------|------------|---------------------------------|--------------------------------------|
| 1 | Greeting       | General    | "Hola"                          | Welcome message, offer help          |
| 2 | Off-topic      | Redirect   | "What's the weather?"           | Polite redirect to agent's scope     |
| 3 | Escalation     | Escalation | "Quiero hablar con un asesor"   | Warm farewell + transfer signal      |

For each FAQ topic, add:
| # | Scenario         | Type | Test Message                        | Expected Behavior                      |
|---|------------------|------|-------------------------------------|----------------------------------------|
| N | FAQ: <topic>     | FAQ  | [sample question from config]       | Knowledge-grounded answer              |
| N | FAQ: <topic> OOB | FAQ  | [question slightly outside scope]   | Honest "I don't know" + offer help     |

For each action topic, add:
| # | Scenario            | Type   | Test Message                          | Expected Behavior                      |
|---|---------------------|--------|---------------------------------------|----------------------------------------|
| N | Action: <topic>     | Action | [request matching topic description]  | Ask for required input                 |
| N | Action: <topic> E2E | Action | [request with input already provided] | Run action, present result             |

Generate ALL test messages in the agent's target language (from config).

Present the scenario table to the user. Ask:
"Review these test scenarios. Want to add, remove, or modify any before running?"

Apply any changes the user requests.

### Step 3: Run Tests (Optional)

Ask: "Want to run these test scenarios via `sf agent preview`? This requires the
agent to be published and activated. (yes/no)"

If the user says no, skip to Step 4 with scenarios only (no results).

If the user says yes:

1. Start a preview session:
   ```bash
   sf agent preview start --api-name <agent-name> --target-org <org-alias>
   ```
   Capture the session ID from the output.

2. For each scenario, send the test message:
   ```bash
   sf agent preview send --session-id <session-id> --message "<test message>"
   ```
   Capture the agent's response.

3. After all scenarios, end the session:
   ```bash
   sf agent preview end --session-id <session-id>
   ```

4. For each scenario, compare the agent's response to the expected behavior:
   - Does the response match the expected type (answer, redirect, escalation)?
   - Is the response in the correct language?
   - Does it follow the brand voice guidelines?

   Mark each scenario as PASS or FAIL with a brief note.

### Step 4: Report

Present the results table:

```
## Test Results for <agent-name>

| # | Scenario       | Input              | Response (first 80 chars)     | Pass? |
|---|----------------|--------------------|-------------------------------|-------|
| 1 | Greeting       | "Hola"             | "Hola! Soy tu asistente..."   | PASS  |
| 2 | Off-topic      | "What's the wea.." | "No puedo ayudarte con..."    | PASS  |
| 3 | Escalation     | "Quiero hablar..." | "Lo conecto con un asesor..." | PASS  |
| 4 | FAQ: rewards   | "Como gano punt.." | "Puedes ganar puntos..."      | PASS  |
| 5 | Action: balance| "Cual es mi sal.." | "Necesito tu numero de..."    | PASS  |
```

If tests were not run (user declined), show the scenarios table without the Response
and Pass columns.

Report:

> Testing complete. [X/Y] scenarios passed.

If any scenarios failed, list the failures with details and suggest investigation steps.

---

## Important Guidelines

These guidelines apply across ALL commands. They encode hard-won lessons from
production Agentforce deployments.

### 1. All Agent Prompts and Descriptions MUST Be in English

This is a Salesforce Agentforce convention. Topic descriptions, action descriptions,
scope definitions, and reasoning instructions are written in English. The agent
responds in whatever language the system instructions specify. Do not write
descriptions or reasoning prompts in Spanish, Portuguese, or any other language,
even if the agent's conversational language is not English.

### 2. The Skill Is Self-Contained

All required knowledge is in the `references/` directory. Do not depend on external
files like MEMORY.md, PROGRESS.md, or AGENT_SCRIPT_MANUAL. The references directory
contains the canonical syntax documentation, anti-patterns, and prompt engineering
guide that drive generation and validation.

### 3. Never Auto-Fix Validation Errors

When validation fails, suggest fixes and let the user decide. Auto-fixing can
introduce subtle bugs in a whitespace-sensitive DSL where a single space difference
changes semantics. Present the error, the likely cause from anti-patterns.md, and
the suggested fix. Let the user apply it or ask you to apply a specific fix.

### 4. Indentation Is Critical

Agent Script uses 3-space indentation. This is not configurable. Mixing 2-space
and 3-space indentation, or using tabs, causes cryptic compile errors that do not
point to the indentation issue. The generate command must enforce 3-space indentation
throughout. When editing existing files, preserve the indentation of surrounding code.

### 5. Data Library Wiring Is the Number One Silent Failure

An agent can publish and activate successfully but return generic, ungrounded answers
because the Data Library was never connected in the Agentforce Builder GUI. There is
no CLI path for this connection as of Spring '26. The `guide` command makes this
explicit and marks it as BLOCKING for knowledge-based agents. Do not skip this step.

### 6. Agent Script Is Beta (Spring '26)

Syntax may change between Salesforce releases. If the SF CLI rejects a construct
that is documented in the references, do not assume the references are wrong. Check:
1. The SF CLI version (`sf --version`) — minimum 2.113.6 required.
2. The Salesforce release notes for any Agent Script changes.
3. The official Agent Script documentation for the current spec.

When in doubt, validate early and often. The validate command is cheap and fast.

### 7. Escalation Must Be Explicit Only

Never generate agents that proactively offer escalation. The agent should only
escalate when the user explicitly requests to speak with a human agent. Proactive
escalation offers (like "Would you like me to connect you with an agent?") cause
false positive escalation detection in downstream systems, breaking conversation
flows silently.

### 8. Config Is the Single Source of Truth

Once discovery is complete, the config JSON drives everything. The generate command
reads the config. The test command reads the config. If the user wants to change
the agent's behavior, they should update the config (re-run discover or edit the
JSON) and re-generate. Do not make ad-hoc changes to the .agent file that are not
reflected in the config.

### 9. One Agent, One Config, One Bundle

Each agent has exactly one discovery config, one .agent file, and one bundle-meta.xml.
Do not generate multiple .agent files for a single agent. If the user wants to
iterate, overwrite (or version with .v2) the existing file. The SFDX project
structure enforces this: one directory per bundle under `aiAuthoringBundles/`.

### 10. Progressive Disclosure

Do not dump all information at once. Each command shows only what is relevant to
that stage. The help command shows the full menu. The discover command focuses on
requirements. The generate command shows a summary. The guide command shows only
post-publish steps. The test command shows only test results. Keep output focused
and actionable.
