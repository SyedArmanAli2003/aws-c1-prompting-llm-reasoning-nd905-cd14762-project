# Customer Support Chatbot with Amazon Bedrock AgentCore

In this project you will build a customer support chatbot using the **Amazon Bedrock AgentCore managed harness**. The chatbot will handle customers' messages for a fictional online shop and must handle each of the following types of messages:

* **bug reports** — collect the details over the conversation, then file a ticket
* **platform questions** — answer from the shop's FAQ
* **anything else** — politely hand the customer off to the human support line

The centerpiece of the project is **prompt engineering**: all of the routing, information gathering, and grounding behavior lives in a single system prompt that you design. The harness supplies the agent loop (model calls, session memory, tool execution) — your prompt supplies the behavior.

> **Why AgentCore?** Bedrock *Agents Classic* was closed to new customers on July 30, 2026, so this course uses its successor, the AgentCore managed harness. Bedrock Evaluations — which you'll use for testing — is unaffected.

There are a number of resources available to you to develop this application:

* `create_bug_report` — a tool (Lambda function) that creates a ticket in a database, exposed to your chatbot through an **AgentCore Gateway**
* `online_shop_faq.md` — a fictional FAQ your application should use to respond to customer questions
* ready-made setup scripts and CloudFormation templates, so you spend your time on the prompt, not the plumbing

You will create the harness, iterate on its system prompt, and then test it in various scenarios.

## Student Submission Results

This submission implements a complete, evaluated customer support chatbot using the **Amazon Bedrock AgentCore managed harness**.

- **Model**: `us.amazon.nova-pro-v1:0`
- **AWS Region**: `us-east-1`
- **Submission Repository**: [SyedArmanAli2003/aws-c1-prompting-llm-reasoning-nd905-cd14762-project](https://github.com/SyedArmanAli2003/aws-c1-prompting-llm-reasoning-nd905-cd14762-project)

### Core Architecture & Routing Behavior

The system prompt (`project/starter/system_prompt.txt`) cleanly classifies incoming customer messages into three distinct routes:

1. **Bug Report Route**:
   - Collects `description`, `stepsToReproduce`, and `environment` across the conversation turns.
   - Strictly enforces that the chatbot does **not** call the tool until all three fields exist.
   - Once all required information is gathered, it invokes `bugreports___create_bug_report` via the AgentCore Gateway.
   - Successfully persists the bug ticket to DynamoDB (`bug-report-tool-stack-bug-reports`).
   - Returns the real DynamoDB ticket ID (e.g., `30780b15-7b3d-493e-a23c-7ff0e553620a`) to the customer.

2. **FAQ / Platform Question Route**:
   - Answers customer questions exclusively using the embedded FAQ (`online_shop_faq.md`) covering orders, shipping, returns, payment, and account support.
   - For unsupported or uncovered questions (such as inquiries about student discounts), it strictly avoids fabricating information and politely redirects the customer to the human support hotline at `1-800-555-0199`.

3. **Other Request Route**:
   - Does not perform unrelated tasks (e.g., writing Python web scrapers, general trivia, code generation).
   - Politely informs the customer that it cannot handle out-of-scope tasks and redirects to the human support hotline at `1-800-555-0199`.

### Security & Guardrails

- **Prompt-Injection Resistance**: Robust against adversarial instructions, jailbreak attempts, and internal system prompt extraction attempts.
- **Strict Anti-Hallucination**: Does not reveal system prompt internals and does not fabricate FAQ facts or store policies.

### Automated Testing (`harness-tests.json`)

The automated evaluation suite contains 6 rigorous test cases covering all routes and edge cases:
1. **Incomplete bug report**: Customer reports a checkout crash; assistant recognizes the bug report, acknowledges the issue without claiming a ticket was filed, and asks for missing reproduction steps and environment.
2. **Return-policy FAQ**: Customer inquires about returns; assistant grounds response in the FAQ (30-day window, unused condition, original packaging).
3. **Unsupported student discount**: Customer asks about student discounts; assistant states FAQ does not cover discounts without inventing policy and redirects to `1-800-555-0199`.
4. **Unrelated web scraper**: Customer requests a Python scraper; assistant declines out-of-scope coding and redirects to `1-800-555-0199`.
5. **Ambiguous short request**: Customer says "It broken"; assistant clarifies what is not working rather than guessing or fabricating details.
6. **Prompt injection**: Malicious input attempts to override instructions and leak system prompt; assistant refuses and remains safely in customer support scope.

### Bedrock Evaluation Results

Automated evaluation was conducted using **Amazon Bedrock Evaluations** (LLM-as-a-judge / Bring-Your-Own-Inference):

- **Bedrock Evaluation Job**: `support-chatbot-eval-final`
- **Evaluation Job ARN**: `arn:aws:bedrock:us-east-1:673594910195:evaluation-job/b6q1cfp20xmp`
- **Status**: `Completed`
- **Metric**: `Builtin.Correctness`
- **Correctness Score**: **1.00**
- **JSONL Records**: `6`
- **HARNESS_ERROR Count**: `0`

> **Note on Evaluation Score**: All 6 records received a correctness score of 1.0 (100% across all evaluated prompts).

### Implementation Evidence & Visual Verification

All visual evidence screenshots are saved in the `screenshots/` directory:

| Evidence Milestone | File Link | Description |
|:---|:---|:---|
| **Lambda Tool Test** | [lambda_test_full_evidence.png](../screenshots/lambda_test_full_evidence.png) | Successful standalone execution of `create_bug_report` Lambda in AWS Console. |
| **DynamoDB Table Items** | [dynamodb_explore_table_items.png](../screenshots/dynamodb_explore_table_items.png) | DynamoDB explorer showing table items in `bug-report-tool-stack-bug-reports`. |
| **AgentCore Gateway** | [agentcore_gateway_evidence.png](../screenshots/agentcore_gateway_evidence.png) | AgentCore Gateway in `READY` state with target `bugreports`. |
| **AgentCore Harness** | [agentcore_harness_evidence.png](../screenshots/agentcore_harness_evidence.png) | Managed harness `support_chatbot` in `READY` status pinned to `us.amazon.nova-pro-v1:0`. |
| **Multi-Turn Bug Chat** | [bug_report_chat_tool_call.png](../screenshots/bug_report_chat_tool_call.png) | Real conversation collecting bug details and invoking `bugreports___create_bug_report`. |
| **Chatbot DynamoDB Ticket** | [chatbot_ticket_dynamodb.png](../screenshots/chatbot_ticket_dynamodb.png) | Real ticket `30780b15-7b3d-493e-a23c-7ff0e553620a` created by chatbot verified in DynamoDB. |
| **Covered FAQ Question** | [faq_covered.png](../screenshots/faq_covered.png) | Chatbot accurately answering delivery and shipping questions from FAQ. |
| **Unsupported FAQ Question** | [faq_unsupported.png](../screenshots/faq_unsupported.png) | Chatbot declining student discount query and redirecting to `1-800-555-0199`. |
| **Other Request** | [other_request.png](../screenshots/other_request.png) | Chatbot politely refusing Python code request and redirecting to `1-800-555-0199`. |
| **Eval Dataset JSONL** | [eval_dataset_jsonl.png](../screenshots/eval_dataset_jsonl.png) | Dataset generation producing 6 records with 0 harness errors. |
| **S3 Dataset Upload** | [s3_eval_dataset.png](../screenshots/s3_eval_dataset.png) | `output_eval_dataset.jsonl` stored in S3 testing bucket. |
| **Bedrock Evaluation Result** | [bedrock_evaluation_result.png](../screenshots/bedrock_evaluation_result.png) | Amazon Bedrock console showing evaluation job completed with 1.00 Correctness. |

## Getting Started

### Dependencies

- An AWS account with Amazon Bedrock and Amazon Bedrock AgentCore access enabled.
- AWS CLI configured with appropriate credentials.
- Python 3.9+ with `boto3` 1.43+ installed (`pip install -r requirements.txt`).
- Access to the Amazon Nova Pro model. **This project pins `us.amazon.nova-pro-v1:0` everywhere** — do not rely on the harness default model, which requires an AWS Marketplace subscription that lab accounts cannot complete.
- Work in **us-east-1**; the scripts and templates assume it.

### Project Files

| File | Description |
|------|-------------|
| `docs/tools-setup.md` | Step-by-step guide for creating the bug report tool and gateway. |
| `docs/testing.md` | Step-by-step guide for automated testing and running Bedrock Evaluations. |
| `solution/` | Reference solution with the complete system prompt and test suite. |
| `cloudformation-tool.yaml` | Creates the DynamoDB table, the `create_bug_report` Lambda, and the IAM roles for the gateway and the harness. |
| `cloudformation-testing.yaml` | Creates the resources used to test your final application (S3 bucket + evaluation role). |
| `create_bug_report.py` | The Lambda function code (also embedded in the tool template) that stores bug reports in DynamoDB. |
| `setup_gateway.py` | Creates the AgentCore Gateway and registers the Lambda as the `create_bug_report` tool. |
| `system_prompt.txt` | **Your main deliverable** — the system prompt for the chatbot. |
| `create_harness.py` | Creates (or updates) the managed harness from `system_prompt.txt`. Re-run it every time you change your prompt. |
| `chat.py` | A terminal chat client for trying out your chatbot in a multi-turn session. |
| `generate-eval-dataset.py` | Runs your harness against a test suite and produces a JSONL file for Bedrock Evaluations. |
| `harness-tests-template.json` | Template for developing your test suite. |
| `cleanup_agentcore.py` | Deletes the harness, gateway target, and gateway when you're done. |

## Project Instructions

### Step 1: Create Resources for your application

First you will deploy the tool your application needs to create bug reports, plus the IAM roles AgentCore requires.

When a customer reports a bug, the chatbot needs to persist it somewhere so the engineering team can follow up. In this project we use a DynamoDB table as a simple ticket store, and a Lambda function as the tool implementation. The chatbot reaches the Lambda through an **AgentCore Gateway** — the gateway presents the Lambda to the model as a callable tool named `create_bug_report`.

**1. Deploy the tool stack** (DynamoDB table + Lambda + IAM roles):

```bash
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

The `--capabilities CAPABILITY_NAMED_IAM` flag is required because the template creates named IAM roles. Besides the Lambda and the table, the stack creates two AgentCore roles whose ARNs appear in the stack outputs: a **gateway role** (lets the gateway invoke the Lambda) and a **harness execution role** (lets the harness call Bedrock models and invoke the gateway).

**2. Create the gateway and register the tool:**

```bash
python setup_gateway.py
```

The script reads the stack outputs itself and saves everything the later steps need to `agentcore_config.json`. For a deeper walkthrough (including how to test the Lambda in isolation), follow [Tools Setup](docs/tools-setup.md).

> If `setup_gateway.py` fails right after the stack finishes with an access or validation error mentioning the role, that's IAM propagation delay — the script already retries, but if it still fails just run it again a minute later.

### Step 2: Build the harness — design the system prompt

Now the fun part. Open `system_prompt.txt` and write the system prompt for your chatbot. Your application needs to handle three different types of requests, and **the routing between them is done entirely by your prompt** — there are no condition nodes or classifiers, just instructions:

- **Bug reports** — if a customer reports a bug on the website, the application needs to collect additional information and then create a ticket using the `create_bug_report` tool.
- **Platform questions** — the application should answer common questions about orders, shipping, returns, and payments using the FAQ.
- **Other requests** — if the message is neither a bug report nor answerable from the FAQ, the application should politely redirect the customer to a human support phone line.

The `create_bug_report` tool accepts three parameters:

* Bug description
* Steps to reproduce
* Environment where the user experienced the bug

Customers rarely provide all three up front. Because the harness keeps **session state** across turns, your chatbot can simply *ask* for what's missing — make sure your prompt tells it to collect all three fields (and how to behave while collecting them, e.g. one question at a time) before calling the tool, and to give the customer their ticket ID afterwards.

Platform questions (orders, shipping, returns, payments) need to be answered from the shop's FAQ. Here we will use the simplest approach and embed the document directly in the prompt — the model sees it at inference time and answers from it. Keep the `{{FAQ}}` placeholder in `system_prompt.txt`; `create_harness.py` replaces it with the contents of `online_shop_faq.md` automatically.

> **Note:** Embedding documents in the prompt works well for short, stable content like a FAQ. For large documents, embedding the full text in every prompt becomes expensive and hits context limits. The standard solution is **Retrieval-Augmented Generation (RAG)**, which retrieves only the relevant passages at query time using a vector index. RAG with Amazon Bedrock Knowledge Bases is outside the scope of this course.

When your prompt is ready, create the harness and chat with it:

```bash
python create_harness.py     # first run takes ~2-3 minutes
python chat.py               # each run = one fresh conversation
```

Iterating is fast: edit `system_prompt.txt`, re-run `create_harness.py` (it updates the existing harness), and start a new `chat.py` session.

#### Some suggestions

Here are some things to keep in mind while working on your application:

* Treat routing as a classification problem inside your prompt: describe the three categories crisply and tell the model to pick exactly one before doing anything else. Vague category definitions produce vague routing.
* Be explicit about the bug-report checklist (description, steps to reproduce, environment) and tell the model **not** to call the tool until every item is collected. Asking one question at a time works noticeably better than asking for everything at once.
* Tell the model to answer platform questions *only* from the FAQ, and what to do when the FAQ doesn't cover the question (that's the hand-off case).
* When the tool succeeds it returns a `ticketId` — instruct the model to relay it to the customer, so you can find the ticket in DynamoDB later.
* Verify tickets really land in the database: `aws dynamodb scan --table-name <BugReportsTableName from stack outputs> --region us-east-1`.
* The tool call appears in `chat.py` as a `[tool call] bugreports___create_bug_report` line — if you never see it, your prompt probably isn't telling the model clearly when to use the tool. The Lambda also logs every event it receives to CloudWatch Logs (`/aws/lambda/bug-report-tool-stack-create-bug-report`), which is the ground truth for what actually reached it.
* There is no "prepare" step and nothing to redeploy: the harness picks up prompt changes as soon as `create_harness.py` finishes.
* Try to implement and test your solution step by step.
* Use the us-east-1 region, as some smaller regions might not have all Bedrock AgentCore features.

### Step 3: Testing

Once your chatbot works, you can keep testing it manually with `chat.py`. However, this approach is tedious and not scalable. Ideally we want an automated way to test the application.

To test your application you will do the following:

* Create a set of test prompts and define expected results — copy `harness-tests-template.json` (e.g. to `harness-tests.json`) and fill in your test cases. Cover all three routes: at least one FAQ question, one bug report, and one out-of-scope request.
* Run your application programmatically on this set of prompts:

  ```bash
  python generate-eval-dataset.py --tests-json harness-tests.json
  ```

  Each test case runs in a fresh session and the final responses are written to `output_eval_dataset.jsonl` in the Bedrock Evaluations input format.
* Deploy the testing stack (S3 bucket + evaluation role), upload the JSONL, and use **Bedrock Evaluations** (LLM-as-a-judge) to score your application's outputs:

  ```bash
  aws cloudformation deploy \
    --template-file cloudformation-testing.yaml \
    --stack-name bug-report-testing-stack \
    --capabilities CAPABILITY_NAMED_IAM \
    --region us-east-1
  ```

Follow the steps in the [Testing and Evaluation](docs/testing.md) document to upload the dataset and create the evaluation job.

## Cleanup

When you are done with the project, delete the AgentCore resources and the CloudFormation stacks to avoid ongoing charges. **Empty the evaluation S3 bucket first** — CloudFormation cannot delete a bucket that still contains objects, so if you skip that step the testing stack ends up in `DELETE_FAILED`:

```bash
python cleanup_agentcore.py
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
aws s3 rm s3://udacity-agentic-engineer-c1-eval-<YOUR_ACCOUNT_ID> --recursive
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```

(The bucket name is in the testing stack's outputs. If the testing stack already shows `DELETE_FAILED`, empty the bucket and run its `delete-stack` command again.)

This removes the harness, gateway, Lambda function, DynamoDB table, IAM roles, and S3 bucket created during the project.

## Built With

* [Amazon Bedrock AgentCore managed harness](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness.html) - Runs the chatbot: agent loop, sessions, and tool execution
* [Amazon Bedrock AgentCore Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html) - Exposes the bug report Lambda as a tool
* [Amazon Bedrock Evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html) - LLM-as-a-judge evaluation
* [AWS Lambda](https://aws.amazon.com/lambda/) - Bug report tool runtime
* [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) - Bug report storage

## License

[License](../LICENSE.md)
