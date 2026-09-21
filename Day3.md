Day 3: Durable Execution & Worker Recovery

1. The Core Problem: Surviving Process Crashes

Consider the following workflow:

1 . An agent starts a task: "Apply student to company drive, book an interview slot, and send an SMS notification."   

2 . The agent successfully books the interview slot.   

3 . The worker server terminates unexpectedly (power loss or crash).   

4 . The system restarts and retries the run.   

5 . Without durable execution, the agent runs from step 1 again: booking a second interview slot and dispatching duplicate SMS messages.   

Day 3 solved this: An agent run is treated as an asynchronous background job, not a transient web request. If a worker dies halfway through, another worker picks up where it left off and finishes the remaining work without re-executing completed side effects.  


2. The Core Mechanics

[User Request] ---> Insert into Queue Table (status = 'queued')
                             |
                             v
  [Worker 1] Claims run with a 30s Lease (status = 'running')
        |
        |--- Step 1: Check eligibility  ---> Saved to DB
        |--- Step 2: Book slot          ---> Saved with Idempotency Key!
        v
   [CRASH! Worker 1 dies]
        |
  (Lease expires after 30 seconds... Reaper resets it to 'queued')
        |
        v
  [Worker 2] Claims the abandoned run
        |
        |--- Checks Step 1: Already completed in DB (Skips)
        |--- Checks Step 2: Key exists in DB (Returns saved result, DOES NOT re-book)
        |--- Step 3: Send SMS          ---> Saved to DB
        v
     [Success!] (status = 'succeeded')



What is an Idempotency Key?

An idempotency key is a deterministic digital fingerprint calculated from the execution 

Before executing any write operation, the system checks:

Has this exact key been recorded already?

YES: Do not call the function again; return the previously stored result.   
NO: Run the function and save the result alongside the key in the exact same database transaction.   

3. Working Code Example

import hashlib
import json

# Simulated database tables
IDEMPOTENCY_STORE = {} # key -> result
BOOKED_SLOTS = []      # Real business data

def calculate_key(run_id: str, step_seq: int, tool_name: str, args: dict) -> str:
    # Sort keys so {"a": 1, "b": 2} matches {"b": 2, "a": 1}
    payload = json.dumps([run_id, step_seq, tool_name, args], sort_keys=True)
    return hashlib.sha256(payload.encode()).hexdigest()

def execute_once(key: str, tool_name: str, action_fn):
    """Guarantees an action runs exactly once, even during replays."""
    # 1. If key exists, return stored result (Replay)
    if key in IDEMPOTENCY_STORE:
        print("--> [Idempotency Guard] Step already completed. Replaying saved result.")
        return IDEMPOTENCY_STORE[key], False
    
    # 2. Otherwise execute the real action
    result = action_fn()
    
    # 3. Store result with the key atomically
    IDEMPOTENCY_STORE[key] = result
    return result, True

# Real action with side effects
def book_slot(student_id: str, slot_id: int):
    BOOKED_SLOTS.append({"student_id": student_id, "slot_id": slot_id})
    return {"status": "confirmed", "slot_id": slot_id}

# --- SIMULATION ---
RUN_ID = "run-uuid-101"
STEP = 2
ARGS = {"student_id": "21CS045", "slot_id": 4}

# First attempt by Worker A
key_a = calculate_key(RUN_ID, STEP, "book_slot", ARGS)
res1, acted1 = execute_once(key_a, "book_slot", lambda: book_slot(**ARGS))
print("Attempt 1 Result:", res1, "| Actually wrote to DB?:", acted1)

# Worker A dies right here! Worker B picks up the exact same step:
key_b = calculate_key(RUN_ID, STEP, "book_slot", ARGS)
res2, acted2 = execute_once(key_b, "book_slot", lambda: book_slot(**ARGS))
print("Attempt 2 Result:", res2, "| Actually wrote to DB?:", acted2)

print("\nFinal Booked Slots Count in Database:", len(BOOKED_SLOTS))
# Notice count is 1, NOT 2!