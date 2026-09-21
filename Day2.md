Day 2: Tools, Service, and Schema Design

1. Why Move to a Service and Real Database?On Day 1, everything ran in local computer memory. If the process crashed or restarted, all state disappeared. In production environments, client interfaces (web or mobile apps) interact with your agent via HTTP endpoints and require real-time updates. 

2. The Core Concept: Two Different Database SchemasWe separated our database into two distinct schemas:  

+---------------------------------------------------------+
|                    POSTGRES DATABASE                    |
+----------------------------+----------------------------+
|   placement (Business)     |       agent (Runtime)      |
+----------------------------+----------------------------+
| * student                  | * thread                   |
| * drive                    | * message (Append-only)    |
| * eligibility_rule         | * run                      |
| * application              | * run_step                 |
| * interview_slot           | * tool_call                |
+----------------------------+----------------------------+


Why Not Put Everything into One Table?Imagine debugging or auditing an agent by replaying a conversation:   

If merged: Replaying a conversation might accidentally write duplicate records into the actual college database (e.g., registering a student twice for a placement drive).  

Because they are separate: You can wipe, audit, or replay conversation histories in agent without modifying business records in placement.   

Messages are Append-OnlyA conversation is an audit trail of what the user asked and what the agent executed. You never execute an UPDATE on old messages. Any correction is recorded as a new message. 


3. Real-Time Streaming with SSE (Server-Sent Events)

Because agent reasoning and database calls take seconds, traditional request-response cycles leave users with a blank screen.

Using Server-Sent Events (SSE), the server keeps the HTTP connection open and streams updates step-by-step:   

1 . run.started -> "Started run"   
2 . tool.started -> "Checking college eligibility database..."   
3 . tool.finished -> "Found matching records in 40ms"   
4 . token -> "You" ... "are" ... "eligible!"   
5 . run.finished -> "Finished"

