# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
Structure: a `for _ in range(MAX_TOOL_ROUNDS)` loop. Each iteration makes one LLM
call and inspects the response.

(a) No tool calls — natural exit:
    assistant_message = response.choices[0].message
    if not assistant_message.tool_calls:
        return assistant_message.content or <fallback string>
    This is the normal, expected exit: the LLM has produced a final text answer.

(b) MAX_TOOL_ROUNDS reached — safety exit:
    If the loop completes all MAX_TOOL_ROUNDS iterations and the LLM was STILL
    asking for tools on the last round (so we never hit the `return` in (a)),
    control falls through past the loop. There I make one final LLM call WITHOUT
    tools (tool_choice="none" / no tools=), forcing the model to summarize what it
    has into a text answer rather than request yet another tool. I return that
    content, or a user-readable fallback if it is somehow empty. This guarantees
    the function always returns a non-empty string and never loops forever.

Edge cases this handles:
  - content is None when the message only has tool_calls → guarded with `or fallback`.
  - An empty/garbled tool result that makes the LLM retry endlessly → capped by
    MAX_TOOL_ROUNDS, then forced to answer.
  - An exception anywhere (API error, bad JSON) → wrapped so a friendly string is
    returned instead of a crash.
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
The final text lives at:

    response.choices[0].message.content

`choices` is a list (index 0 is the primary completion); `.message` is the
assistant message object; `.content` is the generated string. When the message
contains tool_calls instead of a final answer, `.content` is typically None — which
is exactly why the termination check keys off `.tool_calls`, not off `.content`.
I return `assistant_message.content or "<fallback>"` so a None/empty content never
propagates back to the UI as a blank reply.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**
*(Actual observed run, GROQ_API_KEY set, June / summer.)*

```
Query: "How should I care for my monstera this time of year?"
Round 1 tool call: lookup_plant({'plant_name': 'monstera'})   → found: True (Monstera)
Round 1 tool call: get_seasonal_conditions({})                → name: "Summer"
Round 2: no tool_calls → final text answer
Final response: cited the Monstera's watering (every 1–2 weeks), light, humidity and
temp data AND tied it to summer (water more often, watch afternoon sun, spider
mites/fungus gnats). Both tools fired in a single round.
```

**What happens when you ask about a plant that isn't in the database?**

```
"bird of paradise" → lookup_plant returns {"found": False, "message": "...not in the
plant care database. Do not invent specific care instructions..."}. The agent
degraded gracefully every time: acknowledged it wasn't in the database, gave general
tropical-plant guidance (bright indirect light, consistent moisture), and suggested
confirming with a specialized source — without fabricating specific numbers.
```

**One thing about the tool call API that surprised you:**

```
Two things:
1. The assistant message's .content is None (not "") on a tool-call turn, so
   termination must key off .tool_calls, not .content.
2. For the no-argument tool, llama-3.3-70b sometimes sends arguments as the JSON
   string "null", so json.loads() yields None and a naive .get() crashes — the loop
   has to coerce tool_args to a dict. The model also occasionally emits a malformed
   tool call (Groq 400 tool_use_failed); the try/except returns a fallback instead
   of crashing, and a retry succeeds.
```
