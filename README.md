# NovaMart Multi-Agent E-commerce RAG

Capstone project for Udacity's **Future AWS Agent Engineer** Nanodegree (Course 3, Project 3).
A customer-support system for a fictional retailer, NovaMart, built with the **Strands Agents SDK**
on **Amazon Bedrock AgentCore**. One orchestrator routes each customer request to specialist agents
(inventory, policy RAG, refund, communication) that share state through DynamoDB.

**Result:** `python tests/test_agent.py all` scores **120 / 120**.

---

## Architecture

```
Customer request  ->  OrchestratorAgent (Claude Haiku 4.5, temp 0.0)
                          |
      +-------------------+--------------------+----------------------+
      |                   |                    |                      |
 InventoryAgent       PolicyAgent          RefundAgent        CommunicationAgent
 (DynamoDB)        (parallel RAG over 3    (DynamoDB)         (always the last call,
 Sonnet 4.5, 0.1    Knowledge Bases)       Sonnet 4.5, 0.1     writes the customer reply)
                    Sonnet 4.5, 0.2                            Sonnet 4.5, 0.3
                          |
        +-----------------+------------------+
        |                 |                  |
 ReturnsRetriever   ShippingRetriever   WarrantyRetriever     (Sonnet 4.5, temp 0.0,
 (returns KB)       (shipping KB)       (warranty KB)          ThreadPoolExecutor(max_workers=3))
```

All agents read and write one DynamoDB **WorkflowState** record per session. Every write is a
conditional update on a `version` counter (optimistic locking) with a 24-hour TTL.

| Piece | What it does |
|---|---|
| **Orchestrator** | Creates the session, routes by six rules, never decides eligibility and never writes the final reply. `route_to_communication_agent` is always the last tool call. |
| **InventoryAgent** | Looks up order facts and customer tier (`check_order_status`, `get_customer_tier`, `list_customer_orders`). Gathers facts only. |
| **PolicyAgent** | `search_all_policies` queries the three Knowledge Bases in parallel (`ThreadPoolExecutor(max_workers=3)` + `as_completed`) and synthesizes one grounded answer. A failing KB does not break the other two. |
| **RefundAgent** | Applies the return policy (Standard 30 days, Premium 60 days, delivered orders only) using the facts in WorkflowState and today's date; `initiate_refund` records the return. |
| **CommunicationAgent** | Reads the full WorkflowState and writes the customer message. For general questions with no findings (e.g. arithmetic) it answers directly. |

### AWS services used

- **Bedrock AgentCore Runtime** (`udacity_agentcore_runtime`, PUBLIC network, HTTP protocol), deployed with the AgentCore CLI
- **Bedrock AgentCore Memory** (`udacity_agentcore_memory`): `summaryMemoryStrategy` "SessionSummary", 7-day event expiry
- **Bedrock Guardrail** (`udacity-agentcore-guardrail`, version 1, STANDARD tier with cross-region profile): content filters (sexual/violence/hate HIGH, insults/misconduct MEDIUM), PII (card and SSN blocked, email and phone anonymized), three denied topics (competitor products, pricing negotiations, legal threats), managed profanity list
- **Bedrock Knowledge Bases** (3, self-managed): Titan Text Embeddings v2, vector store on **S3 Vectors**, one index per KB
- **DynamoDB** (Orders, Customers, WorkflowState), **S3** (policy documents), **CloudWatch Logs** (INFO), **X-Ray** (100% sampling, Transaction Search)

---

## What is mine and what is starter code

Only `src/agent_orchestrator.py` is meant to be edited. Everything else is the provided starter and is used unchanged:
`config.py`, `src/agent_utils.py`, `src/agent_observability.py`, `src/agentcore_cli.py`, `src/bedrock_kb_retrieval.py`,
`infrastructure/`, `tests/`, and the starter README (kept as `STARTER_README.md`).

**Implemented in `src/agent_orchestrator.py`:**

| Task | Implementation |
|---|---|
| 2.A | `build_inventory_agent` and its three tools |
| 2.B | `build_refund_agent`, `get_inventory_context`, `initiate_refund` (conditional update so a wrong key cannot create an order) |
| 2.C | `build_policy_agent`, three retriever sub-agents, parallel `search_all_policies` |
| 2.D | `build_communication_agent`, `get_full_workflow_context` |
| 2.E | `build_orchestrator_agent`, five tools, a shared `_run_worker` helper that reads state, runs the worker, and writes back with `expected_version` |
| 3 | `create_guardrail` (waits until READY, publishes a numbered version), `deploy_to_agentcore_runtime` |
| 4 | `configure_memory` |
| 6 | `configure_observability` |

**Done by hand in the AWS Console (Task 5):** the three Knowledge Bases (`novamart-returns-kb`, `novamart-shipping-kb`,
`novamart-warranty-kb`), each synced, with their IDs in `.env`.

### How this was built (AI assistance disclosure)

I worked through the project with Claude (Anthropic's AI assistant) as a mentor. Claude drafted the implementation of the
functions above and explained the reasoning behind each one; I created the AWS resources, ran every test and deployment on my own
account, reviewed the code, and debugged the results. The prompts, tool design and settings follow the project specification.

---

## Evidence for the rubric

| Requirement | Where to see it |
|---|---|
| `python tests/test_agent.py all` shows 120/120 | `screenshots/01_tests_all_part1_tasks2-5.png`, `screenshots/02_tests_all_part2_score_120.png` |
| X-Ray Service Map: `NovaMart-Orchestrator` -> workers -> Knowledge Bases | `screenshots/03_xray_service_map_overview.png`, `screenshots/04_xray_trace_map_node_list.png` |
| Completed `agent_orchestrator.py` | `src/agent_orchestrator.py` |
| Populated `.env` (KB IDs, runtime ARN, guardrail ID/version; no AWS keys) | `.env` |

Score breakdown: Task 2 **40/40**, Task 3 **20/20**, Task 4 **15/15**, Task 5 **25/25**, Task 6 **20/20**.

Rubric points that the automated tests do not check, and where they are met in the code:

- No hardcoded model IDs: every `BedrockModel` uses `config.ORCHESTRATOR_MODEL_ID` or `config.WORKER_MODEL_ID`
- `ThreadPoolExecutor(max_workers=3)` with `as_completed` in `search_all_policies`
- Six routing rules in the orchestrator system prompt
- Correct `expected_version` on every WorkflowState write (`_run_worker`)
- Docstrings on every agent builder and tool
- `try/except` in `configure_observability` that prints `[Note] Observability configuration failed: <e>`

### Live results (local `test` run and deployed `invoke`)

| Scenario | Result |
|---|---|
| CUST-001 returns order ORD-27176 | Inventory (delivered, Premium) -> Refund (12 days vs 60-day window, **approved**, `RET-...` reference) -> Communication. WorkflowState version 0 -> 3. |
| CUST-002 asks the Premium return policy | Parallel retrieval from all three KBs (about 8 s), grounded answer: 60 days Premium, 30 days Standard. |
| CUST-003 asks 5 items at $29.99 with 10% off | Orchestrator skips all workers; Communication answers **$134.96**. The guardrail does not block arithmetic. |
| "Is Amazon cheaper than NovaMart?" | Blocked by the guardrail's competitor topic before reaching any worker agent. |
| `python src/agent_orchestrator.py invoke "..."` | The deployed runtime in AWS returns the correct Premium answer. |

---

## Problems found and fixed

1. **Guardrail masked contact details.** The email/phone ANONYMIZE rule turned the support email and phone number I had put in the
   Communication prompt into `{EMAIL}` and `{PHONE}` in customer replies. The guardrail stays as specified; the prompt now says never to
   write email addresses or phone numbers and to say "our customer support team" instead. Fixed locally and redeployed.
2. **Pricing-negotiation topic vs. arithmetic.** The starter warned that the older guardrail tier blocked the math scenario. I used the
   STANDARD tier and defined the topic as haggling over an advertised price, explicitly excluding calculations from a price and discount.
   Verified live: arithmetic passes, negotiation is blocked.
3. **The model has no clock.** The RefundAgent could not judge a return window without today's date, so `get_inventory_context` returns it
   alongside the order facts.
4. **`update_item` can create rows.** `initiate_refund` uses `attribute_exists(order_id)` so a wrong order ID returns "not found" instead of
   silently creating a new order.
5. **Tests change data.** A successful return changes the order to `return_initiated`, so re-running the scenario would deny it. Re-run
   `python infrastructure/seed_data.py` to reset between runs.
6. **Expired lab credentials** (`ExpiredTokenException`) showed up several times; refreshing the lab credentials fixed each one.

### Differences between the starter's documents and the repo

- The architecture diagrams still say Claude 3 Haiku/Sonnet; the repo's `config.py` uses **Claude Haiku 4.5** and **Sonnet 4.5**. I followed the repo.
- The diagram's routing-tool parameter names (`query`) differ from the code stubs (`request`); I followed the stubs.
- During parallel retrieval the terminal can show interleaved streamed text from the sub-agents before the clean results. This comes from
  the starter's trace output, not from the agent logic, and the final answers are unaffected.
- The optional AgentCore Gateway step is skipped (no Lambda tool functions configured); it is not required by the project.

---

## Run it

Prerequisites: Python 3.12, Node.js 20+, `uv`, the AgentCore CLI (`npm i -g @aws/agentcore`), AWS credentials for us-east-1.

```bash
python -m venv venv && source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
python infrastructure/seed_data.py                    # sample customers, orders, policy documents
python tests/test_agent.py task2                      # check each task as you go (task2..task6, or all)
python src/agent_orchestrator.py test                 # three local scenarios, traced to X-Ray
python src/agent_orchestrator.py deploy               # guardrail, runtime, memory, observability
python src/agent_orchestrator.py invoke "What is the return policy for premium customers?"
python tests/test_agent.py all
```

`.env` holds the three Knowledge Base IDs, the runtime ARN and the guardrail ID and version (printed by `deploy`). It contains no AWS credentials.

After the project is reviewed, remove the AWS resources: `python infrastructure/cleanup.py`, then again with `--yes`.

---

## Attribution

The starter code, infrastructure templates, sample data and tests come from Udacity's course repository
(`udacity/cd14764-aws-agentic-c3-classroom`, folder `project/starter`) and are used as provided. This README and the contents of
`src/agent_orchestrator.py` described above are my submission for the Udacity Honor Code.