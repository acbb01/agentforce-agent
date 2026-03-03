# Prompt Engineering Patterns for Agentforce Agents

> Valid for: Agent Script DSL (Spring '26 Beta)
>
> Battle-tested patterns from 28 sessions of Puntos Colombia agent development.
> These are NOT theoretical recommendations -- every pattern here solved a real problem.

---

## System Instructions Design

System instructions in Agent Script must be a **quoted string** (not a `-> |` template).
Keep under 500 words for reliability. Longer instructions cause the LLM to ignore later directives.

### Structure Order (proven reliable)

Role definition -> Scope boundaries -> Language rules -> Vocabulary table -> Tone directives -> Guardrails

Always specify what the agent IS and what it IS NOT. Without negative boundaries, the agent will
attempt to answer anything.

### Example Structure

```
"You are the virtual assistant of [Company], [one-line description]. Your role is to help
[audience] with [scope].

LANGUAGE & TONE:
- ALWAYS respond in [language] using [register: tu/usted].
- Tone: [warm-professional / friendly / corporate / casual].
- Keep messages short ([channel-specific length]).

BRAND VOCABULARY (mandatory):
- [correct term] (never [wrong alternatives])
- [correct term] (never [wrong alternatives])

KNOWLEDGE & ESCALATION:
- Prioritize knowledge base for answers.
- If KB doesn't have exact answer, use general program knowledge.
- Suggest self-service channels before escalation.
- Only escalate when user EXPLICITLY requests it."
```

### Key Lessons

- Vocabulary tables placed directly in system instructions are more reliably followed
  than separate knowledge articles about vocabulary.
- "Never say X" directives are surprisingly effective. The LLM respects explicit prohibitions.
- Role definition must come first. Everything after it is interpreted through that lens.

---

## Tiered Confidence Pattern

Order response strategies from highest to lowest confidence. The agent should exhaust
each tier before falling to the next:

1. **Knowledge base** (highest confidence) -- direct answer from KB articles
2. **General domain knowledge** (medium confidence) -- helpful guidance from what the agent knows about the program
3. **Self-service channel suggestions** (fallback) -- app, website, phone line
4. **Human escalation** (absolute last resort) -- only on explicit user request

Critical instruction to include:
"Never say you don't have information without first trying to help."

Without this, the agent defaults to "I don't have that information" on any KB miss,
which is the single most common failure mode.

---

## Rotating Closings Pattern

**Problem:** Always ending with "Quieres saber mas sobre X?" sounds robotic after 2-3 exchanges.

**Solution:** Instruct the agent to rotate between four closing styles:

1. Follow-up question -- "Quieres saber mas sobre...?"
2. Soft nudge without question -- "Explora las promociones en la app"
3. Actionable tip -- "Tip: activa las notificaciones para no perderte ofertas"
4. Natural ending -- no CTA needed, just end naturally

Instruction to add in reasoning:
"Vary how you close each message. Never use the same closing style twice in a row."

This single instruction dramatically improves conversational quality without any
structural changes.

---

## Anti-Escalation Guardrails

**Problem:** Agents escalate too easily, flooding human agents with KB-answerable questions.
This was the most persistent issue across sessions 10-20.

### Three-Layer Defense

**Layer 1 -- System instructions:**
"Only escalate when user EXPLICITLY requests it, never as first option."

**Layer 2 -- Topic-level reasoning:**
"If KB lacks answer, DO NOT say you lack info. Guide to self-service channels."

**Layer 3 -- Topic selector guard:**
Use `available when @variables.escalation_requested == True` on the escalation topic
transition. This hides the escalation tool from the LLM unless the flag is explicitly set.

### Escalation Topic Description (proven wording)

"User EXPLICITLY asks to speak with an asesor, person, or human. Do NOT use if the user
just has a difficult question."

The word "EXPLICITLY" in both the system instructions and topic description is essential.
Without it, the LLM interprets frustrated tone as an implicit escalation request.

---

## "Try Harder Before Escalating" Recipe

Place this in FAQ topic reasoning instructions:

```
| If the knowledge base does not have the exact answer, DO NOT say you lack information.
| Instead:
|   1. Offer useful guidance based on what you know about the program
|   2. Suggest concrete self-service channels (app, website, phone line)
|      where the user can resolve their issue
|   3. Only mention the option to speak with an asesor as a last resort,
|      never as a first suggestion
| Never say the phrase 'no tengo informacion' or equivalent.
| Be proactive and helpful.
```

This pattern reduced false escalations by roughly 80% in testing.

---

## Brand Voice Enforcement

### Vocabulary Tables

Define correct and forbidden terms directly in system.instructions:

```
BRAND VOCABULARY (mandatory):
- "puntos" (never "millas", "creditos", "recompensas")
- "acumula" (never "gana", "obtiene")
- "redime" (never "canjea", "usa tus puntos")
- "aliado" (never "socio", "partner", "tienda asociada")
```

### Register and Tone

- Specify register explicitly: "tu" vs "usted" changes the entire personality.
- Define action verbs specific to the brand (acumula, redime, disfruta, consulta).
- These verbs should appear in both system instructions and knowledge articles for reinforcement.

### Emoji Policy

Specify which emoji are allowed, max frequency, and appropriate moments:

- Welcome message: one emoji allowed
- Farewell: one emoji allowed
- Celebratory moments (balance check with high points): one emoji allowed
- Mid-conversation: no emoji

Without an explicit emoji policy, the agent either overuses them or avoids them entirely.

---

## Channel-Specific Patterns

### WhatsApp

- Max 3-4 lines per message for readability
- No markdown formatting (WhatsApp does not render it reliably)
- Quick-reply style closing questions
- Simple language, no jargon
- Emoji used sparingly (see emoji policy above)

### Web Chat

- Slightly longer responses are acceptable
- Markdown may be supported depending on the widget
- Can include links and light formatting
- Bullet lists work well for multi-point answers

### Voice

- Keep responses very short and conversational
- Avoid lists (hard to follow in voice)
- Use natural speech patterns, not written prose
- Confirmations should be explicit ("Entendido, tu saldo es...")

---

## Topic Description Design

The LLM uses topic descriptions to decide routing. Vague descriptions cause misrouting.

**Good:** "Answers questions about the loyalty program: how it works, how to accumulate
points, how to redeem, partner stores, available services, and promotions."

**Bad:** "Handles general inquiries."

### Rules

- State the clear intent and specific triggers
- Keep scope narrow -- one topic should not cover everything
- List the categories of questions this topic handles
- If two topics overlap, the LLM will pick randomly. Eliminate overlap in descriptions.

---

## Action Description Design

Describe user intent, not implementation details. The LLM matches user messages to action
descriptions, so they must be written from the user's perspective.

**Good:** "Query the available points balance given a cedula number."

**Bad:** "Call the Get_Customer_Balance Flow."

The LLM does not know what a Flow is. It knows what users want.

### Rules

- Describe WHAT the action does for the user, not HOW it works
- Include the key input the user provides ("given a cedula number")
- Keep to one sentence
- If the action has prerequisites, state them ("Requires the user to provide their ID first")

---

## Common Pitfalls

1. **System instructions too long:** Over 500 words and the LLM starts ignoring later sections.
   Move overflow content to topic-level reasoning instead.

2. **Duplicate guidance across layers:** If system instructions say "be brief" but topic
   reasoning says "provide detailed explanations", the agent behaves inconsistently.
   Pick one location for each directive.

3. **Negative-only guardrails:** "Don't do X, don't do Y" without positive guidance on
   what TO do. Always pair prohibitions with alternatives.

4. **Testing only happy paths:** The agent breaks on edge cases like empty KB results,
   ambiguous user intent, or multi-topic questions. Test these explicitly.

5. **Assuming topic routing is deterministic:** It is not. The LLM chooses topics
   probabilistically based on descriptions. Two similar descriptions will split traffic
   unpredictably.
