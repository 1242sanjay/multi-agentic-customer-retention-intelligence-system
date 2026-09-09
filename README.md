# Customer Retention Intelligence using Multi-Agent AI System
## Objective

In this notebook, we are going to build a Multi-Agent Customer Retention Intelligence System using LangChain & LangGraph. The system takes a Customer ID as input and uses three specialized, **tool-bound** sub-agents plus a Supervisor to profile the customer, assess churn risk, retrieve the appropriate retention strategy, and generate a structured Next Best Action (NBA) recommendation.
- **Customer Profiler Agent:** calls `fetch_customer_record_tool` to retrieve customer information and creates a profile summary.
- **Churn Diagnostician Agent:** calls `classify_risk_segment_tool` to apply deterministic business rules to classify the customer into a churn risk segment.
- **Retention Strategist Agent:** calls `recommend_retention_action_tool` to retrieve (via RAG over ChromaDB) the most appropriate retention playbook.
- **Supervisor Agent:** Coordinates the LangGraph workflow and assembles the final structured NBA recommendation from the three sub-agent's verified outputs.

Together, these agents form a modular and transparent multi-agent system that helps businesses identify at-risk customers and recommend effective retention strategies.

| S.No. | Stage | Agent | Tool | Type |
|---|---|---|---|---|
| 1 | Profile | Customer Profiler Agent | `fetch_customer_record()` | Dict lookup + LLM formatting |
| 2 | Diagnose | Churn Diagnostician Agent | `classify_risk_segment()` | Deterministic Python rules |
| 3 | Recommend | Retention Strategist Agent | `recommend_retention_action()` | RAG retrieval (ChromaDB) |
| 4 | Decide | Supervisor Agent | — | Combines verified sub-agent outputs to produce the final six-field decision |

---

## Full Architecture Diagram
<pre>

                                                    CUSTOMER RETENTION INTELLIGENCE
                                                    Input: Customer ID (e.g. "V001")
                                                                    |
                                                                    v
            +------------------------------------------------------------------------------------------------------+
            |  NODE: supervisor  (ROUTING MODE)                                                                    |
            |  +------------------------------------------------------------------------------------------------+  |
            |  |   Reads shared state. Checks which sub-agents have already run                                 |  |
            |  |   and whether their output is complete.                                                        |  |
            |  |                                                                                                |  |
            |  |   Decision Logic:                                                                              |  |
            |  |    * profiler_output missing/incomplete?  --> route to profiler                                |  |
            |  |    * diagnostician_output missing?        --> route to diagnostician                           |  |
            |  |    * strategist_output missing?           --> route to strategist                              |  |
            |  |    * ALL THREE complete?                  --> FINAL RESPONSE MODE                              |  |
            |  +------------------------------------------------------------------------------------------------+  |
            +------------------------------------------------------------------------------------------------------+
                |                                 |                                       |                    |
                | next='profiler'                 | next='diagnostician'                  | next='strategist'  | next='END'(all 3 done)
                v                                 v                                       v                    +--------------+
+--------------------------------+  +--------------------------------+  +------------------------------------------+          |
| NODE:profiler                  |  | NODE:diagnostician             |  | NODE: strategist                         |          |
|(Customer Profiler Agent)       |  |(Churn Diagnostician Agent)     |  |(Retention Strategist Agent)              |          |
|                                |  | Agent)                         |  | Agent)                                   |          |
|                                |  |                                |  |                                          |          |
| Tool:                          |  | Tool:                          |  | Tool:                                    |          |
|  fetch_customer_record(cust_id)|  |  classify_risk_segment(cust_id)|  |  recommend_retention_action(segment_name)|          |
| --> dict lookup                |  | --> DETERMINISTIC Python rules,|  | --> RAG: queries ChromaDB vector store   |          |
| --> LLM formats profile summary|  |     NOT LLM reasoning          |  |     built from retention_playbooks.json  |          |
|                                |  | --> segment,NBA, rules fired,  |  | --> reads CLV tier from profiler to pick |          |
|                                |  |     policy                     |  |      variant                             |          |
|                                |  |                                |  |                                          |          |
| MUST NOT:                      |  | MUST NOT:                      |  | MUST NOT:                                |          |
|  - invent data,                |  |  - guess segment               |  |  - override segment/decision             |          |
|  - recommend actions           |  |  - paraphrase rules            |  |  - invent steps                          |          |
|                                |  |  - recommend actions           |  |  - omit policy                           |          |
+--------------------------------+  +--------------------------------+  +------------------------------------------+          |
                |                                  |                                         |                                |
                | Loops back!                      | Loops back!                             | Loops back!                    |
                +----------------------------------+-----------------------------------------+--------------------------------+
                                                                  |
                                                                  v
                                                        Back to NODE: supervisor
                                                     (checks completeness again,
                                                      routes to next missing agent)
                                                                  |
                                                                  |
                                                     (repeats until all 3 complete)
                                                                  |
                                                                  v
                                    +----------------------------------------------------------------------+
                                    |  NODE: supervisor  (FINAL RESPONSE MODE)                              |
                                    |  +----------------------------------------------------------------+  |
                                    |  | Triggered only when profiler_output AND diagnostician_output    |  |
                                    |  | AND strategist_output are all present.                          |  |
                                    |  |                                                                 |  |
                                    |  | Combines the three sub-agent outputs into the 6-field decision: |  |
                                    |  |  - Customer ID          (from profiler)                        |  |
                                    |  |  - Risk Segment         (from diagnostician, exact name)        |  |
                                    |  |  - Next Best Action     (ESCALATE / NURTURE / HOLD)             |  |
                                    |  |  - Action Rationale     (2 sentences, cites actual signal       |  |
                                    |  |                          values from profiler output)           |  |
                                    |  |  - Treatment Steps      (1-2-3, from strategist)                |  |
                                    |  |  - Engagement Policy    (exact policy, from strategist)         |  |
                                    |  +----------------------------------------------------------------+  |
                                    +----------------------------------------------------------------------+
                                                                        |
                                                                        v
                                                            +--------------------------+
                                                            |          END             |
                                                            | (Final 6-field decision) |
                                                            +--------------------------+


</pre>
