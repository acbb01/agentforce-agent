# Agent Script DSL — Quick Reference

Valid for: Agent Script DSL (Spring '26 Beta)

---

## File Format

- Extension: `.agent`, UTF-8 encoding
- Indentation: **3 spaces** (whitespace-sensitive, like Python/YAML — tabs will cause parse errors)
- Bundle structure: `.agent` file + `.bundle-meta.xml` in `aiAuthoringBundles/<name>/`
- `sf agent publish` generates Bot + BotVersion + GenAiPlannerBundle + GenAiPlugin(s) + GenAiFunction(s)

## Top-Level Blocks (in order)

Blocks appear at root indentation. Order matters.

### config

```
config:
   developer_name: "MyAgent"
   default_agent_user: "user@orgid.ext"
   agent_label: "My Agent"
   description: "Agent description"
```

- `default_agent_user`: **MUST be a real Salesforce user email** (not a placeholder). Failing this causes activation errors.

### variables

```
variables:
   mutable:
      customerName:
         type: string
         default: ""
      retryCount:
         type: number
         default: 0
      isVerified:
         type: boolean
         default: false
   linked:
      accountId:
         type: string
         source: "@MessagingSession.AccountId"
```

- **Mutable**: `type` (string | number | boolean) + `default` (required).
- **Linked**: `type` + `source` (required). **NO `default` allowed** — value comes from the source binding.

### system

```
system:
   instructions: "You are a helpful assistant. Always respond in Spanish."
   messages:
      welcome: "Hola, bienvenido."
      error: "Lo siento, ocurrio un error."
```

- `instructions`: **QUOTED STRING only**. Do NOT use `-> |` template syntax here. This is a common generation error.

### knowledge

```
knowledge:
   citations_enabled: False
```

- `citations_enabled` (bool): **Required** when block is present.
- `rag_feature_config_id` (string): Optional. Created after UI Data Library connection.
- `citations_url` (string): Optional.
- This block alone does NOT wire the Data Library — you must also connect it via Agentforce Builder UI (one-time, persists across republishes).

### language

```
language:
   default_locale: "es"
```

Valid locales: `ar`, `bg`, `ca`, `cs`, `da`, `de`, `el`, `en_AU`, `en_GB`,
`en_US`, `es`, `es_MX`, `et`, `fi`, `fr`, `fr_CA`, `he`, `hi`, `hr`, `hu`,
`id`, `in`, `it`, `iw`, `ja`, `ko`, `ms`, `nl_NL`, `no`, `pl`, `pt_BR`,
`pt_PT`, `ro`, `sv`, `th`, `tl`, `tr`, `vi`, `zh_CN`, `zh_TW`.

CAUTION: bare `en` and `pt` are NOT valid — use `en_US` and `pt_BR`.
Only include `additional_locales` / `all_additional_locales` if multi-language is needed.

### start_agent \<name\>

```
start_agent greeting:
   label: "Greeting"
   description: "Initial routing"
   reasoning:
      ...
   after_reasoning:
      ...
```

### topic \<name\>

```
topic faq:
   label: "FAQ"
   description: "Answers frequently asked questions"
   actions:
      ...
   reasoning:
      ...
   before_reasoning:
      ...
   after_reasoning:
      ...
```

## Actions Block

Defined at topic level. Must be defined **before** being referenced in `reasoning.actions`.

```
topic balance:
   actions:
      get_balance:
         description: "Retrieves customer point balance"
         inputs:
            customerId:
               type: string
               description: "The customer ID"
         outputs:
            balance:
               type: number
               description: "Current balance"
         target: "flow://Get_Customer_Balance"
```

### Target formats

| Format | Example |
|---|---|
| Flow | `target: "flow://Get_Customer_Balance"` |
| Apex | `target: "apex://BalanceController"` |
| Prompt Template | `target: "prompt://MyTemplate"` |

Do NOT use `standardInvocableAction://` targets for managed actions — they cause "Internal Error" on publish.

## Reasoning Block

### Arrow syntax (`->`) — Procedural logic

```
reasoning:
   instructions:
      -> if @variables.isVerified == true:
         -> run @actions.get_balance
         -> set @variables.lastChecked = "now"
      -> else:
         -> | Please verify your identity first.
         -> @utils.transition to @topic.verification
```

### Pipe syntax (`|`) — LLM prompt text

```
reasoning:
   instructions:
      -> |
         You are answering a question about points.
         The customer name is {!@variables.customerName}.
         Provide a helpful response.
```

- `->` (arrow): conditionals (`if`/`else`), `run @actions`, `set @variables`, `@utils.transition`
- `|` (pipe): LLM prompt text with template expressions `{!@variables.x}`
- Template expressions `{!...}` are ONLY valid inside `|` pipes.

### reasoning.actions — Tools exposed to LLM

Separate from topic-level action definitions. This list tells the LLM which tools it can invoke.

```
reasoning:
   actions:
      - @actions.get_balance
      - @utils.escalate
      - @utils.transition to @topic.faq
      - @utils.setVariables:
           with customerId = @variables.customerId
```

## Reasoning Actions (LLM tools)

| Action | Purpose |
|---|---|
| `@actions.<name>` | Invoke a defined action |
| `@utils.transition to @topic.<name>` | Route conversation to another topic |
| `@utils.setVariables` | LLM slot-filling (see below) |
| `@utils.escalate` | Structural human transfer (no text body) |

### @utils.setVariables

```
- @utils.setVariables:
     with customerId = field "Ask the customer for their ID"
     with channel = "whatsapp"
```

- `field = ...` (unquoted after `field`): LLM extracts value from conversation
- `field = "fixed"` (quoted string): assigns a fixed value

### Conditional availability

```
- @actions.get_balance:
     available when @variables.isVerified == true
```

Hides the tool from the LLM unless the condition is met.

## After Reasoning Block

Runs **after** reasoning exits. Deterministic only.

```
after_reasoning:
   -> if @variables.needsEscalation == true:
      -> @utils.escalate
   -> else:
      -> @utils.transition to @topic.faq
```

**Supports**: `if`/`else`, `set @variables`, `@utils.transition`, `run @actions`

**Does NOT support**: pipes (`|`), template expressions (`{!...}`), LLM prompts. This is a common generation error.

## Template Expressions

Syntax: `{!expression}` — **ONLY valid inside `|` pipes**.

```
-> |
   Balance: {!@variables.balance}
   Total: {!@variables.balance + @variables.bonus}
   Status: {!if(@variables.isActive, "Active", "Inactive")}
```

- Supports: variable interpolation, arithmetic (`+`, `-`), comparison, logical (`and`, `or`, `not`)
- **NOT supported in**: `system.instructions`, static `description` fields, `after_reasoning` blocks

## Resource References

| Reference | Resolves to |
|---|---|
| `@variables.x` | Declared variable |
| `@actions.x` | Declared action |
| `@topic.x` | Declared topic |
| `@outputs.x` | Action output field |
| `@utils.escalate` | Human transfer |
| `@utils.transition` | Topic routing |
| `@utils.setVariables` | Slot-filling |
| `@MessagingSession.*` | Session context fields |
| `@MessagingEndUser.*` | End-user context fields |

## Operators

| Type | Operators |
|---|---|
| Comparison | `==`, `!=`, `<`, `<=`, `>`, `>=`, `is`, `is not` |
| Logical | `and`, `or`, `not` |

## Common Generation Errors

1. **Using `-> |` template in `system.instructions`** — must be a quoted string.
2. **Putting `{!...}` in `after_reasoning`** — only deterministic logic allowed.
3. **Referencing an action before defining it** — define in `actions:` block first.
4. **Using tabs instead of 3-space indentation** — causes silent parse failures.
5. **Setting `default` on a `linked` variable** — linked variables get values from `source` only.
6. **Using `connections:` block** — listed in spec but NOT supported (compiler error). Remove entirely.
7. **Targeting managed actions with `standardInvocableAction://`** — causes "Internal Error" on publish.
8. **Using a placeholder email for `default_agent_user`** — must be a real org user.
