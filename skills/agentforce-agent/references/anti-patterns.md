# Agent Script Anti-Patterns

> Valid for: Agent Script DSL (Spring '26 Beta)
>
> 20 anti-patterns discovered across 28 sessions of Agent Script development.

---

## Won't Compile

These prevent `sf agent publish` or `sf agent validate authoring-bundle` from succeeding.

### 1. `connections:` block

Listed in the spec and present in `future_recipes/`, but NOT supported in Spring '26 Beta. Produces compiler error: "Invalid syntax after conditional statement".

```yaml
# WRONG
connections:
   messaging:
      escalation_message: "Transferring you now."
      outbound_route_type: "queue"
      outbound_route_name: "Support_Queue"
```

```yaml
# RIGHT
# Remove the connections block entirely.
# Configure escalation routing in Salesforce Omni-Channel setup, not Agent Script.
```

### 2. `else if` not supported

The DSL does not have an `else if` construct. Use nested `if`/`else` instead.

```yaml
# WRONG
-> if @variables.category == "balance":
   | Checking your balance.
-> else if @variables.category == "faq":
   | Let me look that up.
```

```yaml
# RIGHT
-> if @variables.category == "balance":
   | Checking your balance.
-> else:
   -> if @variables.category == "faq":
      | Let me look that up.
```

### 3. Templates in `system.instructions`

`system.instructions` accepts a quoted string only. The `->` arrow or `|` pipe syntax is not valid here.

```yaml
# WRONG
system:
   instructions:
      -> |
      You are a helpful agent.
```

```yaml
# RIGHT
system:
   instructions: "You are a helpful agent."
```

### 4. Actions referenced before defined

Actions must be defined in the `actions:` block of a topic BEFORE they are referenced in `reasoning.actions`. The compiler reads top-to-bottom.

```yaml
# WRONG
topic: Balance_Topic
   reasoning:
      actions:
         - @actions.get_balance
   actions:
      get_balance:
         description: "Retrieve customer balance"
         target: "flow://Get_Customer_Balance"
```

```yaml
# RIGHT
topic: Balance_Topic
   actions:
      get_balance:
         description: "Retrieve customer balance"
         target: "flow://Get_Customer_Balance"
   reasoning:
      actions:
         - @actions.get_balance
```

### 5. Linked variables with defaults

Linked variables pull their value from a source. Assigning a default with `=` is not allowed.

```yaml
# WRONG
variables:
   customer_name: linked string = "John"
```

```yaml
# RIGHT
variables:
   customer_name: linked string
```

### 6. `run` nesting

Only one level of `run` is permitted. You cannot nest a `run` inside another `run`.

```yaml
# WRONG
-> run @actions.validate_account
   -> run @actions.get_balance
```

```yaml
# RIGHT
-> run @actions.validate_account
-> run @actions.get_balance
```

---

## Runtime Failure

These compile and pass validation but fail at publish time or produce broken behavior at runtime.

### 7. Redefining managed actions

`standardInvocableAction://EmployeeCopilot__AnswerQuestionsWithKnowledge` compiles but causes "Internal Error, try again later" on publish. Managed actions cannot be redefined with custom inputs/outputs.

```yaml
# WRONG
actions:
   answer_with_knowledge:
      description: "Answer from knowledge base"
      target: "standardInvocableAction://EmployeeCopilot__AnswerQuestionsWithKnowledge"
      inputs:
         query: string
      outputs:
         answer: string
```

```yaml
# RIGHT
# Do not redefine managed actions. Use the knowledge: block instead.
knowledge:
   citations_enabled: False
```

### 8. `knowledge:` block without UI Data Library connection

The `knowledge:` block in Agent Script enables the feature, but the Data Library must ALSO be connected via the Agentforce Builder GUI (one-time step). Without it, FAQ answers will be generic hallucinations instead of grounded responses.

```yaml
# WRONG — assumes this is sufficient
knowledge:
   citations_enabled: False
# (no UI wiring done)
```

```yaml
# RIGHT
knowledge:
   citations_enabled: False
# THEN: Agentforce Builder > Agent > Data Library > connect PuntosFAQFile
# This persists across republishes.
```

### 9. Placeholder `default_agent_user`

Placeholder or fake emails cause publish failure. Must be an actual Salesforce user in the org.

```yaml
# WRONG
config:
   default_agent_user: "agent@example.com"
```

```yaml
# RIGHT
config:
   default_agent_user: "puntos_colombia_agent@00dho00000zthl0.ext"
```

### 10. Flow wrapper for knowledge retrieval

Wrapping knowledge retrieval in a Flow action loses the RAG grounding context. The agent returns generic answers instead of grounded KB content.

```yaml
# WRONG
actions:
   search_kb:
      description: "Search knowledge via flow"
      target: "flow://Knowledge_Search_Flow"
```

```yaml
# RIGHT
# Use the knowledge: block at root level + UI Data Library connection.
knowledge:
   citations_enabled: False
```

### 11. `streamKnowledgeSearch` as action target

Compiles but still requires UI Data Library connection. Offers no advantage over the `knowledge:` block approach.

```yaml
# WRONG — no benefit over knowledge: block
actions:
   search_kb:
      target: "standardInvocableAction://streamKnowledgeSearch"
```

```yaml
# RIGHT
knowledge:
   citations_enabled: False
# + one-time UI Data Library connection
```

---

## Subtle Bugs

These compile, publish, and run -- but produce incorrect, degraded, or confusing agent behavior.

### 12. Pipes in `after_reasoning`

The `|` pipe syntax is not allowed in `after_reasoning`. Use `reasoning.instructions` with `->` for LLM-directed text.

```yaml
# WRONG
after_reasoning:
   | Thank you for your question.
```

```yaml
# RIGHT
after_reasoning:
   -> @variables.closing_message = "Thank you for your question."
```

### 13. Templates in static descriptions

Variable interpolation like `{!@variables.name}` does NOT work in `description` fields. Templates only work inside `|` pipes within `instructions:->`.

```yaml
# WRONG
topic: Greeting_Topic
   description: "Greets {!@variables.customer_name}"
```

```yaml
# RIGHT
topic: Greeting_Topic
   description: "Greets the customer by name"
   reasoning:
      instructions:
         -> |
         Greet the customer: {!@variables.customer_name}
```

### 14. Single-keyword escalation triggers

Using a single word like "asesor" as an escalation signal triggers on the agent's OWN conversational offers ("puedo conectarte con un asesor"), creating false positive escalations.

```yaml
# WRONG — in Lambda escalation detection
ESCALATION_KEYWORDS = ["asesor", "agente", "humano"]
```

```yaml
# RIGHT — require multi-word phrases indicating ACTIVE transfer
ESCALATION_PHRASES = ["te estoy transfiriendo", "te voy a conectar con"]
ESCALATION_SOFT_KEYWORDS = ["asesor", "agente", "humano"]
# Require 2+ soft keywords co-occurring, or 1 exact phrase
```

### 15. Over-restrictive knowledge instructions

Telling the agent "only use verified KB data, if not found say you don't have info and offer an advisor" causes the agent to give up immediately on any question not found verbatim in the knowledge base.

```yaml
# WRONG
reasoning:
   instructions:
      -> |
      Only answer from verified knowledge base data.
      If the answer is not found, say you cannot help and offer an advisor.
```

```yaml
# RIGHT
reasoning:
   instructions:
      -> |
      Use the knowledge base to answer customer questions.
      If the topic is outside your scope, let the customer know
      and suggest related topics you can help with.
```

### 16. Robotic closing patterns

Always ending with the same formulaic question sounds mechanical and damages brand voice.

```yaml
# WRONG — every response ends identically
reasoning:
   instructions:
      -> |
      Always end your response with: "Would you like help with anything else?"
```

```yaml
# RIGHT
reasoning:
   instructions:
      -> |
      End conversations naturally. Vary your closings between
      offering further help, summarizing what was resolved,
      or simply confirming the answer was useful.
```

### 17. Redundant action calls without guards

Without guard conditions, expensive actions (API calls, Flow invocations) run on every conversational turn, even when the data has already been retrieved.

```yaml
# WRONG — runs every turn
reasoning:
   actions:
      - @actions.get_balance
```

```yaml
# RIGHT — guard with variable check
-> if @variables.balance == "":
   -> run @actions.get_balance
```

### 18. Assuming topic transitions return

After `@utils.transition to @topic.B`, control returns to `start_agent` when topic B completes -- NOT back to the originating topic A. Design topics as self-contained units.

```yaml
# WRONG — expecting return to topic A
topic: Topic_A
   reasoning:
      instructions:
         -> |
         Collect info, then transition to Topic_B.
         After Topic_B, continue processing here.
         -> @utils.transition to @topic.Topic_B
```

```yaml
# RIGHT — Topic_B handles its own completion
topic: Topic_A
   reasoning:
      instructions:
         -> |
         Collect info, then transition to Topic_B for processing.
         -> @utils.transition to @topic.Topic_B

topic: Topic_B
   reasoning:
      instructions:
         -> |
         Process the request and provide the final answer to the customer.
```

### 19. Editing GenAiPlannerBundle directly

`sf agent publish` generates the GenAiPlannerBundle from the `.agent` file. Manual edits to the bundle are overwritten on the next publish.

```yaml
# WRONG
# Manually editing force-app/.../aiAuthoringBundles/MyAgent.bundle-meta.xml
# to add topic references or change planner behavior
```

```yaml
# RIGHT
# Edit the .agent file, then run:
# sf agent publish --api-name MyAgent
# The bundle is regenerated automatically.
```

### 20. Stale sessions after agent activation

After activating a new agent version, existing sessions may continue using a cached older version. Users must start a new conversation to pick up changes.

```
# WRONG — assuming existing sessions update automatically
# "I published and activated v5, all users should see changes now"
```

```
# RIGHT — existing sessions are pinned to their creation-time version
# New conversations will use the latest active version.
# For testing, always start a fresh session after publishing.
# In production, consider session TTLs or notifying users to restart.
```
