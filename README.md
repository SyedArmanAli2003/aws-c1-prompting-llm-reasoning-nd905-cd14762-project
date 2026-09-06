# Purpose of This Repo

This repo is the source of truth for the course project **"Customer Support Chatbot with Amazon Bedrock AgentCore"** (Prompting for Effective LLM Reasoning). It contains the starter files students use to build the project.

> **Note:** Bedrock *Agents Classic* was closed to new customers on July 30, 2026. This project runs on its successor, the **Amazon Bedrock AgentCore managed harness**, with tools exposed through an **AgentCore Gateway**. Bedrock Evaluations, which the project uses for testing, is unaffected.

## Folder Structure

### Project Folder

The `project` folder contains all files and instructions necessary for the project:

* `project/README.md` — the full project instructions (setup, building the harness, testing, cleanup).
* `project/starter/` — the files students start from:
  * CloudFormation templates for the bug-report tool (Lambda + DynamoDB + IAM roles) and the testing resources (S3 + evaluation role)
  * Python setup scripts for the AgentCore resources (`setup_gateway.py`, `create_harness.py`), a chat client (`chat.py`), and cleanup (`cleanup_agentcore.py`)
  * `system_prompt.txt` — the student's main deliverable
  * the FAQ document, the evaluation-dataset generator, and a test-suite template

The reference solution, rubric, and detailed docs (`docs/tools-setup.md`, `docs/testing.md`) live in the companion solution repo.

### What students build

1. Deploy the tool stack (CloudFormation) and create the gateway (`setup_gateway.py`).
2. Design the system prompt: route each message to bug-report collection, FAQ answering, or a polite human hand-off; collect all bug details across a multi-turn session before filing a ticket with the `create_bug_report` tool.
3. Create the harness (`create_harness.py`), iterate with `chat.py`.
4. Test automatically: run a test suite through `generate-eval-dataset.py` and score the results with Bedrock Evaluations.
5. Clean up all resources.

All work happens in **us-east-1**, with the model pinned to `us.amazon.nova-pro-v1:0`.

---

# Project Submission & Evaluation Results

**Student Repository**: [SyedArmanAli2003/aws-c1-prompting-llm-reasoning-nd905-cd14762-project](https://github.com/SyedArmanAli2003/aws-c1-prompting-llm-reasoning-nd905-cd14762-project)  
**Foundation Model**: `us.amazon.nova-pro-v1:0`  
**AWS Region**: `us-east-1`  
**Evaluation Score**: **1.00 (100% Correctness across all 6 test cases)**

## 1. Architecture & Deployed Components

- **Tool Stack (`bug-report-tool-stack`)**:
  - Lambda Function: `bug-report-tool-stack-create-bug-report`
  - DynamoDB Table: `bug-report-tool-stack-bug-reports`
  - IAM Roles for Gateway and Harness Execution
- **AgentCore Gateway**:
  - Gateway ARN: `arn:aws:bedrock-agentcore:us-east-1:673594910195:gateway/bug-report-tool-stack-gateway-umrlcei2lg`
  - Gateway Target: `bugreports` (`LXUC4OS9B3`) exposing `create_bug_report`
- **AgentCore Managed Harness**:
  - Harness Name: `support_chatbot`
  - Harness ARN: `arn:aws:bedrock-agentcore:us-east-1:673594910195:harness/support_chatbot-Aiv1HP44uR`
  - Harness Status: `READY`
  - System Prompt: Fully grounded system prompt (`project/starter/system_prompt.txt`) with embedded FAQ (`{{FAQ}}`) and strict routing rules.

## 2. System Prompt Engineering & Routing Strategy

The system prompt implements strict 3-way routing:
1. **Technical Bug Reports**:
   - Gathers required details turn-by-turn: `description`, `steps_to_reproduce`, and `environment`.
   - Never calls `create_bug_report` until all three fields are provided.
   - Automatically invokes `create_bug_report` once all parameters are present and immediately relays the returned `ticketId` to the customer.
2. **Platform & FAQ Inquiries**:
   - Strictly answers questions covered in `online_shop_faq.md` (orders, shipping, returns, payment, contact support).
   - If an inquiry is not covered or unsupported (e.g. student discounts), strictly refuses to speculate or invent policies and redirects the customer to human phone support at `1-800-555-0199`.
3. **Out-of-Scope & Other Inquiries**:
   - Politely declines unrelated tasks (e.g., coding, general trivia, prompt injection) and redirects the user to human phone support at `1-800-555-0199`.

## 3. Verified Multi-Turn Bug Report & Chatbot-Created Ticket

During manual verification with `chat.py`:
- The chatbot engaged in a multi-turn conversation, requesting reproduction steps and device environment before filing a report.
- The chatbot executed `bugreports___create_bug_report` via the AgentCore Gateway.
- A real ticket was persisted in DynamoDB:
  - **Ticket ID**: `30780b15-7b3d-493e-a23c-7ff0e553620a`
  - **Description**: Checkout page crashes when clicking Pay button
  - **Steps**: Added item to cart, clicked checkout, entered address, clicked pay button
  - **Environment**: Chrome 128 on macOS Sonoma

## 4. Automated Testing & Bedrock Evaluation Results

- **Test Suite (`project/starter/harness-tests.json`)**:
  - `t1`: Bug report initial turn (recognizes bug, asks for missing steps and environment).
  - `t2`: Covered FAQ query regarding return policy (answers accurately from FAQ).
  - `t3`: Unsupported FAQ query regarding student discounts (declines gracefully, routes to `1-800-555-0199`).
  - `t4`: Out-of-scope coding request (declines gracefully, routes to `1-800-555-0199`).
  - `t5`: Ambiguous / under-specified input "It broken" (asks clarifying questions without hallucinating).
  - `t6`: Adversarial jailbreak / prompt leak attempt (refuses to disclose prompt or perform unsafe action).
- **Evaluation Dataset**: Generated to `project/starter/output_eval_dataset.jsonl` and uploaded to `s3://udacity-agentic-engineer-c1-eval-673594910195/output_eval_dataset.jsonl`.
- **Bedrock Evaluation Job**:
  - Job Name: `support-chatbot-eval-final`
  - Job ARN: `arn:aws:bedrock:us-east-1:673594910195:evaluation-job/b6q1cfp20xmp`
  - Metric: **Built-in Correctness (LLM-as-a-judge / Bring-Your-Own-Inference)**
  - **Overall Score: 1.00 (100% Correctness across all 6 test records)**

## 5. Submission Evidence & Screenshots

All visual evidence has been captured and stored in the `screenshots/` directory:

| Evidence Artifact | File Path | Description |
|-------------------|-----------|-------------|
| Lambda Test Event | `screenshots/lambda_test_full_evidence.png` | Successful execution of `create_bug_report` Lambda in AWS Console. |
| DynamoDB Table View | `screenshots/dynamodb_explore_table_items.png` | DynamoDB console showing table items for bug reports. |
| DynamoDB Ticket Evidence | `screenshots/dynamodb_ticket_evidence.png` | Detailed item view in DynamoDB showing ticket attributes. |
| AgentCore Gateway | `screenshots/agentcore_gateway_evidence.png` | AgentCore Gateway in `READY` status with `bugreports` target. |
| AgentCore Harness | `screenshots/agentcore_harness_evidence.png` | Managed harness `support_chatbot` in `READY` status pinned to Nova Pro. |
| Chatbot Tool Call | `screenshots/bug_report_chat_tool_call.png` | Multi-turn bug report conversation calling `bugreports___create_bug_report`. |
| Chatbot Ticket in DynamoDB | `screenshots/chatbot_ticket_dynamodb.png` | DynamoDB console showing chatbot ticket `30780b15-7b3d-493e-a23c-7ff0e553620a`. |
| Covered FAQ Test | `screenshots/faq_covered.png` | Chatbot answering shipping and delivery times accurately from FAQ. |
| Unsupported FAQ Test | `screenshots/faq_unsupported.png` | Chatbot handling student discount query and redirecting to `1-800-555-0199`. |
| Other Request Test | `screenshots/other_request.png` | Chatbot refusing Python code request and redirecting to `1-800-555-0199`. |
| Evaluation Dataset JSONL | `screenshots/eval_dataset_jsonl.png` | Generation of `output_eval_dataset.jsonl` with 6 valid records. |
| S3 Dataset Upload | `screenshots/s3_eval_dataset.png` | `output_eval_dataset.jsonl` uploaded to S3 testing bucket. |
| Bedrock Evaluation Result | `screenshots/bedrock_evaluation_result.png` | Amazon Bedrock evaluation job showing 1.00 Correctness score. |
