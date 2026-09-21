# Day 5: LangChain Integration, Document Q&A, and Agent Toolkits

## 1. Executive Summary
Day 5 focused on moving from custom, hand-coded agent loops to production-grade AI frameworks using **LangChain** and **Google Generative AI (`ChatGoogleGenerativeAI`)** in Google Colab. The practical exercises covered four primary milestones:
1. Direct model invocation and response handling with Gemini.
2. Building an interactive, context-grounded customer support bot using `pypdf` and **Gradio**.
3. Developing an autonomous PDF-reading agent using LangChain document loaders and custom tool interfaces.
4. Implementing a deterministic calculator agent that offloads mathematical expressions to Python execution rather than relying on language model token prediction.

---

## 2. Core Concepts & Architecture

### Why Frameworks Over Scratch Loops?
* **Model Abstraction:** LangChain's `ChatGoogleGenerativeAI` standardizes message schemas, parameter configurations (such as `temperature`), and token parsing.
* **Declarative Tooling (`@tool`):** Automatically converts Python functions, parameter type annotations, and docstrings into model-readable tool schemas without requiring manual JSON schema generation.
* **Agent Execution (`create_agent`):** Manages the inner execution cycle—deciding when to invoke tools, passing parameters, and feeding observation results back into conversational memory.

### Prompt Grounding vs. Dynamic Tool Calls
* **Prompt Grounding (Direct Context Injection):** Injects raw text extracted from a PDF directly into the system prompt with strict negative constraints (*"If the information is not available, say you do not have that information"*). This prevents the model from hallucinating non-existent facts.
* **Autonomous Document Tool (`read_pdf`):** Wraps document retrieval inside an agentic tool, allowing the agent to dynamically determine whether it needs to consult the document before generating an answer.
* **Deterministic Computation:** Offloads mathematical computations to an evaluation engine (`eval`) to prevent LLM arithmetic calculation errors.

---

## 3. Practical Implementations (Google Colab Code)

### Prerequisites & Dependencies
```python
'!pip install langchain langchain-community langchain-google-genai pypdf gradio'

Task 1: Direct LLM Invocation with Gemini


Demonstrates model initialization, zero-temperature configuration for deterministic answers, and safe extraction of text responses across varying output formats.

import os
from langchain_google_genai import ChatGoogleGenerativeAI

# API Key Configuration
# Best practice: Store keys securely in Colab Secrets or environment variables
os.environ["GOOGLE_API_KEY"] = "YOUR_GOOGLE_API_KEY"

# Initialize Gemini Model
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0
)

# Run Query
question = input("Ask something: ")
response = llm.invoke(question)
answer = response.content

# Safe extraction if response content is returned as a chunk list
if isinstance(answer, list):
    answer = "\n".join(
        item["text"] for item in answer
        if isinstance(item, dict) and item.get("type") == "text"
    )

print("\n" + "=" * 60)
print(f"Question: {question}")
print("=" * 60)
print(f"Answer: {answer}")
print("=" * 60)


Task 2: Restaurant Customer Support Bot (Gradio UI + PDF Grounding)


Extracts unstructured data from a PDF document using pypdf, injects that context into the prompt with guardrails, and exposes the bot via a browser-based Gradio chat interface.

import gradio as gr
from pypdf import PdfReader
from langchain_google_genai import ChatGoogleGenerativeAI

# 1. Initialize LLM
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0
)

# 2. Extract Document Context
reader = PdfReader("/content/restaurant_menu.pdf")
restaurant_info = ""
for page in reader.pages:
    text = page.extract_text()
    if text:
        restaurant_info += text + "\n"

# 3. Chatbot Handler with Negative Guardrail
def chatbot(message, history):
    prompt = f"""
You are a restaurant customer support assistant.

Restaurant Information : {restaurant_info}

Customer Question : {message}

Answer the customer clearly and politely.
If the information is not available in the restaurant information, say that you don't have that information.
"""
    response = llm.invoke(prompt)
    return response.content

# 4. Launch Web UI
demo = gr.ChatInterface(
    fn=chatbot,
    title="Restaurant Support Bot",
    description="Ask questions about our restaurant menu, timings, and policies."
)
demo.launch()   

Task 3: Autonomous PDF Reader Agent

Uses LangChain PyPDFLoader to parse a PDF file, registers a @tool function to query the document content, and constructs an autonomous agent that searches the document only when required.

Task 3: Autonomous PDF Reader Agent
Uses LangChains PyPDFLoader to parse a PDF file, registers a @tool function to query the document content, and constructs an autonomous agent that searches the document only when required.


from langchain_community.document_loaders import PyPDFLoader
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.agents import create_agent
from langchain.tools import tool

# 1. Initialize Model
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0
)

# 2. Load PDF Document
loader = PyPDFLoader("/content/Resume_6.pdf")
documents = loader.load()

# 3. Define PDF Tool
@tool
def read_pdf(question: str) -> str:
    """Answer a question using the pdf content."""
    text = '\n'.join(doc.page_content for doc in documents)
    return text

# 4. Create Agent with Bound Tool
agent = create_agent(
    model=llm,
    tools=[read_pdf],
    system_prompt="""
    You are a PDF reader agent.
    Use the read_pdf tool to find information from the document.
    Answer only using information from the PDF.
    If the answer is not available in the PDF, say:
    'I could not find this information in the PDF.'
    """
)

# 5. Invoke Agent Loop
question = input("Ask something about the PDF: ")
result = agent.invoke({
    "messages": [
        {"role": "user", "content": question}
    ]
})

# 6. Parse and Display Result
answer = result["messages"][-1].content
if isinstance(answer, list):
    answer = "\n".join(
        item["text"] for item in answer
        if isinstance(item, dict) and item.get("type") == "text"
    )

print("\n" + "=" * 60)
print("PDF Agent Answer:")
print("=" * 60)
print(answer)
print("=" * 60)


Task 4: Autonomous Math / Calculation Agent

Equips the agent with a custom Python arithmetic tool (calculator), using error-handling blocks to execute computations accurately and avoid LLM calculation inaccuracies.

from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.agents import create_agent
from langchain.tools import tool

# 1. Initialize Model
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0
)

# 2. Define Execution Tool with Safe Fallbacks
@tool
def calculator(expression: str) -> str:
    """Calculate a mathematical expression.
    execute the tool as it is, dont make any change"""
    try:
        result = eval(expression)
        return str(result)
    except Exception:
        return "Invalid mathematical expression."

# 3. Assemble Calculator Agent
agent = create_agent(
    model=llm,
    tools=[calculator],
    system_prompt="""
    You are a calculator agent.
    Use the calculator tool for mathematical calculations.
    Return the final answer clearly.
    """
)

# 4. Execute Calculation Query
question = input("Enter your calculation: ")
result = agent.invoke({
    "messages": [
        {"role": "user", "content": question}
    ]
})

# 5. Extract Output
answer = result["messages"][-1].content
if isinstance(answer, list):
    answer = "\n".join(
        item["text"] for item in answer
        if isinstance(item, dict) and item.get("type") == "text"
    )

print("\n" + "=" * 50)
print("Calculator Agent Output:")
print("=" * 50)
print(answer)
print("=" * 50)

4. Key Takeaways & Technical Summary

Tool Declarations: Using @tool simplifies tool generation by transforming docstrings into model-readable schemas.

Deterministic Computations: Offloading arithmetic to specialized tools prevents hallucinations inherent to next-token prediction models.

Strict Guardrails: Negative boundary conditions in system prompts ensure that answers remain restricted to extracted document context.

Rapid Prototyping: Combining LangChain with Gradio enables quick transformation of backend agentic loops into browser-ready web applications.