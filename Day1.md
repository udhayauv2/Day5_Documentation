# Day 1: Agent Foundations & Building an Agent from Scratch

## 1. What is an AI Agent? (In Plain English)
A standard Large Language Model (LLM) like ChatGPT or Gemini is like an extremely knowledgeable friend who is locked inside an empty room. If you ask them a grammar question or general knowledge, they answer brilliantly. But if you ask them: *"What is my college fee balance?"*, they cannot check your college database, so they will simply make up a confident guess (a hallucination).

An **Agent** gives that locked-in friend hands and eyes. An agent combines:
1. **The Model:** The brain that decides what needs to be done.
2. **Instructions:** The rules and boundaries (the system prompt).
3. **Tools:** Actual Python functions the model is allowed to request (like checking a database or sending an email)[cite: 1].
4. **Memory:** The history of what was said and done so far[cite: 1].

### Workflow vs. Agent
* **Workflow:** Like an ATM machine[cite: 1]. Fixed steps, fixed order, completely predictable[cite: 1]. If a step fails or something unusual happens, it crashes or stops[cite: 1].
* **Agent:** Like an office assistant[cite: 1]. You give them a goal (e.g., *"Find the cheapest flight and book it"*), and **they decide the path**[cite: 1]. They might search, get an error, search again with different filters, and finish the job[cite: 1].

---

## 2. The Core Loop: Perceive $\rightarrow$ Plan $\rightarrow$ Act $\rightarrow$ Observe

An agent runs inside a simple loop[cite: 1]:
1. **Perceive:** The agent reads what the user asked and reviews past messages[cite: 1].
2. **Plan:** The model decides: *"Can I answer directly, or do I need to call a tool?"*[cite: 1]
3. **Act:** If it needs a tool, our Python code executes that specific function[cite: 1].
4. **Observe:** The result of the tool is fed back to the model as a new message[cite: 1].
5. This loop repeats until the model produces a final text answer without asking for any more tools[cite: 1].

---

## 3. How Tools Work (The Prescriptions Analogy)
**The LLM never runs your Python code directly[cite: 1].** 
Think of a doctor writing a prescription[cite: 1]. The doctor doesn't make the medicine or inject it; they write a note saying *"Give Patient X 500mg Paracetamol"*[cite: 1]. The pharmacist (your Python code) reads the note, checks if it makes sense, executes the real action, and reports back[cite: 1].

A tool is made of two parts:
1. A **Python function** that does the work[cite: 1].
2. A **JSON Schema** (a simple description) that tells the model what the tool is named, what it does, and what inputs it expects[cite: 1].

---

## 4. Basic Working Code Example (Agent Loop from Scratch)

Here is how a real agent loop works in roughly 50 lines of clear Python code:

```python
import json

# 1. Define real Python functions (Tools)
def get_fee_balance(roll_number: str) -> dict:
    """Look up outstanding college fees for a student."""
    mock_db = {"21CS045": 18500, "21IT012": 0}
    balance = mock_db.get(roll_number)
    if balance is None:
        return {"error": f"Student {roll_number} not found. Please check the roll number."}
    return {"roll_number": roll_number, "balance_inr": balance}

# 2. Tool Registry and Dispatcher
TOOLS = {
    "get_fee_balance": get_fee_balance
}

# 3. Simulated Model Call (In real life, this calls Gemini/OpenAI API)
def mock_llm_call(messages):
    last_msg = messages[-1]["content"]
    # If the user asks for balance, model decides to call tool
    if "balance for 21CS045" in last_msg.lower():
        return {
            "type": "tool_call",
            "tool_name": "get_fee_balance",
            "arguments": {"roll_number": "21CS045"}
        }
    # Once tool result is observed, model gives final answer
    elif "balance_inr" in last_msg:
        data = json.loads(last_msg)
        return {
            "type": "text",
            "content": f"Your outstanding fee balance is Rs. {data['balance_inr']}."
        }
    return {"type": "text", "content": "How can I help you today?"}

# 4. The Agent Loop
def run_agent(user_prompt: str, max_steps: int = 5):
    messages = [
        {"role": "system", "content": "You are a helpful campus assistant."},
        {"role": "user", "content": user_prompt}
    ]
    
    print(f"User: {user_prompt}\n" + "-"*40)
    
    for step in range(max_steps):
        reply = mock_llm_call(messages)
        
        # Natural exit: Model decided it has enough info to answer
        if reply["type"] == "text":
            print(f"Assistant: {reply['content']}")
            return reply['content']
        
        # Tool call requested: Act & Observe
        tool_name = reply["tool_name"]
        tool_args = reply["arguments"]
        print(f"[Step {step+1}] Model requested tool: {tool_name}({tool_args})")
        
        try:
            # ACT: Run our local python code safely
            tool_fn = TOOLS[tool_name]
            result = tool_fn(**tool_args)
        except Exception as exc:
            # Self-healing: Caught error is fed back to the model as information
            result = {"error": str(exc), "hint": "Check parameter format and retry"}
        
        # OBSERVE: Feed result back into conversation history
        messages.append({
            "role": "tool",
            "content": json.dumps(result)
        })
    
    print("Error: Reached maximum allowed loop iterations!")

# Run it
run_agent("Can you check the balance for 21CS045?")


5. Self-Healing Errors & Loop Termination
Self-Healing: When a tool fails (e.g., student ID doesn't exist), don't let Python crash[cite: 1]! Catch the error with try/except and hand the error string back to the model as a tool message[cite: 1]. The model reads: {"error": "invalid format, expected 21CS045"} and tries again with corrected inputs[cite: 1].

Two Exits (Safety First):

Natural exit: The model stops requesting tools and prints the answer[cite: 1].

Hard limit (max_steps): A strict counter (e.g., maximum 5 iterations)[cite: 1]. If a tool is broken or network is down, this stops the agent from running forever and draining your API budget[cite: 1].