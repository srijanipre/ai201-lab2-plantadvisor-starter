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
So the way im handling this is i wrap the whole loop in a for loop that runs at most MAX_TOOL_ROUNDS times using range(MAX_TOOL_ROUNDS), that way it physically cant loop forever even if the model keeps asking for tools over and over. inside each round i make the LLM call and grab assistant_message = response.choices[0].message and then check assistant_message.tool_calls. for the first exit condition, if tool_calls is empty or falsy that means the model is done and just has a normal answer for me, so i break out and return its text content. for the second condition, if i actually burn through all the rounds and the model is STILL asking for tools, the for loop just ends on its own and at that point i dont want to hand back nothing, so i fall back to returning whatever text content i have or a hardcoded message so the function never returns an empty response. when i described my first plan to my AI tool it flagged three failure modes i had to fix: looping forever which the range cap takes care of, returning an empty string becuase the content field can actually come back as None on a tool call turn so i guard it with content or a fallback, and raising an exception since json.loads on the tool arguments or the groq api call itself can blow up, so i wrap the loop in a try/except and return a friendly error string instead of crashing the whole gradio app.
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
The text i actually want to return lives on response.choices[0].message.content. so choices is a list and i grab index 0, then .message gives me the assistant message object, and .content is the actual string the model wrote out for the user. the one gotcha i learned is that content can be None on a turn where the model only asked for tools and didnt write any text, so when im returning it i dont just blindly hand it back, i do something like return assistant_message.content or "<some fallback message>" so i never end up returning a None or an empty string to gradio which would definitly look broken to the user.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: lookup_plant({"plant_name": "calathea"})
Round 2 tool call: None needed — the one lookup gave the model everything it required, so it stopped calling tools and just answered.
Final response: A grounded care summary that opened with "According to the care data for your calathea..." and pulled straight from the database — keep the soil consistently moist with filtered water, low to medium indirect light away from direct sun, high humidity around 50%+, and 60–80°F.
```

**What happens when you ask about a plant that isn't in the database?**

```
When I asked about a plant that isnt in the database, like string of pearls, lookup_plant came back with found False and my not-found message instead of a plant dict, and the agent handled it pretty gracefully. It told the user up front that the plant isnt in the curated database, then still gave actual general care advice organized by watering, light and humidity instead of just dead-ending with "I dont know." One thing i noticed is that the model even retried the lookup on its own with the scientific name (Senecio rowleyanus) before falling back to general guidance, which i think got triggered by the line in my message suggesting it try the common or scientific name.
```

**One thing about the tool call API that surprised you:**

```
The thing that surprised me most was how much the model's behavior is shaped by the strings i hand back to it, not just by the system prompt. When a tool call has no arguments the API sends the arguments field as the literal string "null" instead of an empty object, which i didnt expect and had to coerce into an empty dict so it wouldnt crash dispatch_tool. I was also surprised that just rewording the not-found message, without touching the prompt or the model at all, visibly changed how structured and useful the fallback answer was, which really drove home that the tool return value is its own control surface and not just data.
```
