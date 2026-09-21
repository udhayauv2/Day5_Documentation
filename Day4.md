Day 4: Multi-Agent Systems & Least Privilege


1. Why Not Use a Single Large Agent?

When an agent grows in scope, putting 20 different tools and large prompt instructions into a single model leads to structural issues:   

Tool Confusion: The model starts confusing overlapping tools (e.g., calling search_books instead of reserve_book).   

Permission Risks: If an agent has both read tools (looking up books) and write tools (modifying balances or bookings), prompt injection or misinterpretation can trigger unintended side effects.   

Context Bloat: Conversational history becomes unnecessarily large, leading to higher token latency and costs.   

The Solution: Split the system into a Supervisor and focused Specialists

2. The Architecture: Agent-as-a-Tool

                                +--------------------------------+
                                |        SUPERVISOR AGENT        |
                                |  * Talks to the user           |
                                |  * Decides which expert to call|
                                |  * HAS NO DIRECT WRITE ACCESS  |
                                +---------------+----------------+
                                                |
                            +------------------+------------------+
                            |                                     |
                            v                                     v
                +-------------------------+           +-------------------------+
                |   CATALOGUE AGENT       |           |   CIRCULATION AGENT     |
                |   (Read-Only Specialist)|           |   (Action Specialist)   |
                | * search_books()        |           | * check_fines()         |
                | * get_book_details()    |           | * reserve_book()        |
                | CANNOT change any data! |           | Acts ONLY on verified   |
                |                         |           | active user session     |
                +-------------------------+           +-------------------------+


The Principle of Least PrivilegeLeast privilege means granting each module only the minimum capabilities required for its task.   

Do not rely solely on system prompt warnings like "Please do not modify records".   

Enforce permissions in Python code: Do not attach write tools to the Catalogue Agent. If the model is manipulated, it lacks callable functions to alter records. 

3. Working Code Example (Supervisor & Specialist Delegation)

# 1. Specialist Toolset (Read-Only)
def search_books(query: str) -> list:
    catalog = [{"id": 1, "title": "Clean Code"}, {"id": 2, "title": "Data Intensive Apps"}]
    return [b for b in catalog if query.lower() in b["title"].lower()]

# 2. Specialist Runner
def run_catalogue_specialist(task: str) -> str:
    print(f"   [Catalogue Specialist] Handling: '{task}'")
    # Isolated context: The specialist only sees this specific task
    results = search_books(task)
    if results:
        return f"Found matching book: ID {results[0]['id']} - '{results[0]['title']}'"
    return "No books found."

# 3. Supervisor Tools (The specialist is wrapped as a tool!)
def ask_catalogue(query: str) -> str:
    """Use this tool to search the library catalog for book titles."""
    return run_catalogue_specialist(query)

SUPERVISOR_TOOLS = {
    "ask_catalogue": ask_catalogue
}

# 4. Supervisor Logic
def run_supervisor(user_request: str):
    print(f"User Request: {user_request}")
    
    # Supervisor delegates task to Catalogue Specialist
    if "clean code" in user_request.lower():
        print("Supervisor: Delegating lookup to Catalogue Specialist...")
        info = SUPERVISOR_TOOLS["ask_catalogue"]("Clean Code")
        
        # Supervisor receives result and constructs final response
        final_reply = f"Here is what I found for you: {info}"
        return final_reply

# Execute the multi-agent system
reply = run_supervisor("Do you have the book Clean Code in the library?")
print("\nFinal Reply to User:\n" + reply)