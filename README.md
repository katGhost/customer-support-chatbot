# customer-support-chatbot

A routed customer-support chatbot built with AWS services. The system classifies incoming customer requests and routes them to the appropriate support path for bug reporting, platform FAQ questions, or unsupported requests.

## Overview

The chatbot implements three support paths:

* **Bug Report Path** — collects the information required to create a bug report and persists the completed ticket through an AgentCore Gateway-connected Lambda tool.
* **Platform FAQ Path** — answers platform questions using only approved FAQ content.
* **Other Request Path** — redirects unsupported requests to human support.

The implementation was completed primarily through the AWS Console because the provided lab credentials could not be successfully authenticated in the Integrated Workspace terminal.

The bug-report implementation uses the **Amazon Bedrock AgentCore managed Harness**. The bug-report behavior is defined in the Harness system prompt rather than through a separate agent resource.

---

## Architecture

```text
                              Customer
                                 │
                                 ▼
                           ┌───────────┐
                           │ Guardrail │
                           └─────┬─────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                  Block                     Allowed
                    │                         │
                    ▼                         ▼
              Blocked message             Classifier
                                              │
                         ┌────────────────────┼────────────────────┐
                         │                    │                    │
                         ▼                    ▼                    ▼
                       bug                platform               other
                         │                    │                    │
                         ▼                    ▼                    ▼
                 AgentCore Managed       FAQ Path          Human Support
                     Harness
                         │
                         │ create_bug_report
                         ▼
                  AgentCore Gateway
                         │
                         ▼
                      Lambda
                         │
                         │ PutItem
                         ▼
                    DynamoDB
```

---

## Routing Logic

### 1. Classifier

The classifier assigns each allowed incoming request to one of three labels:

* `bug` — bug reports or technical issues
* `platform` — platform questions that should be answered from the approved FAQ
* `other` — requests outside the supported bug-report and FAQ paths

The classifier returns only the classification label.

---

## 2. Bug Report Path

Bug reports are handled through the **AgentCore managed Harness**.

There is no separate Bug Report Agent resource. The bug-report route and its collection rules are defined in the Harness system prompt.

The Harness is configured with the AgentCore Gateway as a tool provider. The Gateway exposes the Lambda-backed `create_bug_report` tool.

### Collection requirements

Before invoking the tool, the Harness must collect:

* `description`
* `stepsToReproduce`
* `environment`

The Harness is instructed to:

1. Acknowledge the reported issue.
2. Collect the required bug-report information.
3. Ask only one follow-up question at a time when information is missing.
4. Avoid calling the bug-report tool while required information is incomplete.
5. Call `create_bug_report` only after all required fields have been collected.
6. Confirm successful ticket creation after the tool returns successfully.
7. Provide the generated ticket ID to the customer.

The resulting flow is:

```text
Customer reports bug
        │
        ▼
Harness collects description
        │
        ▼
Harness collects reproduction steps
        │
        ▼
Harness collects environment
        │
        ▼
All required information available?
        │
       Yes
        │
        ▼
AgentCore Gateway
        │
        ▼
create_bug_report Lambda
        │
        ▼
DynamoDB
        │
        ▼
Ticket ID returned
        │
        ▼
Harness confirms ticket creation
```

---

## 3. Platform FAQ Path

Platform questions are routed to an FAQ-only support path.

The FAQ path is configured to:

* answer only from the approved FAQ content
* avoid external knowledge and unsupported assumptions
* avoid inventing information
* return a defined fallback when the requested information is not covered

Fallback response:

> That information is not available in the FAQ.

---

## 4. Other Request Path

Requests that are neither bug reports nor supported platform FAQ questions are redirected to human support.

Human support helpline:

`+1 (202) 555-0147`

The Other Request path does not attempt to answer unsupported requests.

---

## Bug-Report Tool Configuration

The `create_bug_report` tool accepts the three fields collected by the Harness:

```json
[
  {
    "name": "create_bug_report",
    "description": "Create a bug report ticket after collecting description, steps to reproduce, and environment.",
    "inputSchema": {
      "type": "object",
      "properties": {
        "description": {
          "type": "string"
        },
        "stepsToReproduce": {
          "type": "string"
        },
        "environment": {
          "type": "string"
        }
      },
      "required": [
        "description",
        "stepsToReproduce",
        "environment"
      ]
    }
  }
]
```

The tool is exposed through the **AgentCore Gateway** and backed by a Lambda function.

Depending on the Gateway target configuration, the runtime tool name may appear with a namespace. The submission evidence may therefore show a name such as:

```text
bugreports___create_bug_report
```

---

# Lambda and DynamoDB

The Lambda function is responsible for persisting completed bug reports.

The Lambda receives the three required fields from the Gateway tool call, generates a ticket ID, and creates an item in:

```text
bug-report-tool-stack-bug-reports
```

The DynamoDB table uses:

```text
ticket_id
```

as the partition key.

A completed record contains the ticket identifier and the collected bug-report information.

The intended persistence path is:

```text
AgentCore Harness
      │
      ▼
AgentCore Gateway
      │
      ▼
Lambda
      │
      ▼
DynamoDB
```

---

# System Prompt Behavior

The primary prompt requirements are implemented through the support-path prompts stored in `prompts/`.

### Bug-report prompt

The bug-report system prompt instructs the Harness to:

* collect all three required fields
* ask one follow-up question at a time
* avoid calling the tool before all required information is available
* invoke the bug-report tool after collection is complete
* confirm successful submission
* provide the generated ticket ID

### FAQ prompt

The FAQ prompt instructs the support path to:

* answer only from the provided FAQ
* avoid outside knowledge
* return the defined fallback for uncovered questions

### Other-request prompt

The Other Request prompt instructs the support path to:

* avoid attempting to answer unsupported requests
* redirect the customer to human support

---

# Project Files

Prompt files are stored in:

```text
prompts/
├── classifier.md
├── bug-report.md
├── platform-faq.md
├── other.md
├── prompt-injection-test.md
└── tests.md
```

Test configuration:

```text
flow-tests.json
```

---

# Screenshots

Implementation and test evidence is stored in:

```text
screenshots/
├── flow.png
├── classifier.png
├── condition-node-expressions.png
├── bug-report.png
├── guardrail.png
├── platform.png
├── other.png
└── test-outputs.png
```

The screenshots document:

* overall chatbot flow
* classifier configuration
* routing conditions
* AgentCore Harness configuration
* bug-report system prompt
* Gateway/tool configuration
* DynamoDB persistence
* FAQ configuration
* Other Request configuration
* test outputs and evaluation evidence

---

# Test Coverage

The project covers the following support behaviors.

### Bug-report path

The expected behavior is:

1. Customer reports a bug.
2. Harness identifies the request as a bug report.
3. Harness collects the bug description.
4. Harness collects reproduction steps.
5. Harness collects environment information.
6. Harness asks only for missing information when necessary.
7. Harness invokes `create_bug_report` only when all required fields are available.
8. Lambda creates the DynamoDB record.
9. Harness confirms creation and provides the ticket ID.

The submission evidence includes the system prompt, conversation transcript, Gateway tool call, and resulting DynamoDB record.

### FAQ path

Tests cover:

* FAQ questions with known answers
* FAQ questions outside the approved content

For an uncovered question, the expected response is:

```text
That information is not available in the FAQ.
```

### Other Request path

Tests verify that unsupported requests are redirected to human support rather than answered by the chatbot.

### Flow tests

`flow-tests.json` contains test cases covering:

* bug reports
* incomplete bug reports
* FAQ-covered questions
* FAQ-uncovered questions
* unsupported requests

The test file is the remaining item being finalized.

---

## Verified Results

The following implementation components have been successfully configured and/or captured as submission evidence:

* Overall chatbot flow
* Classifier configuration
* Routing conditions
* Guardrail configuration
* AgentCore managed Harness configuration
* Bug-report system prompt
* Platform FAQ path
* Other Request path
* DynamoDB table and persisted bug-report data

The Harness configuration is complete, with the remaining limitation being an IAM permission issue affecting the Gateway/Lambda tool path in the provided lab environment.

---

## Environment Limitations

The provided lab environment introduced two access restrictions during implementation.

### Integrated Workspace credentials

The provided lab credentials could not be successfully authenticated in the Integrated Workspace terminal.

As a result:

* terminal-based AWS setup could not be completed using the provided credentials
* console-based configuration was used instead

### IAM restrictions

The lab environment also restricted some IAM role updates required for complete Gateway/Lambda permission configuration and troubleshooting.

This affected environment permissions rather than the chatbot's routing design, prompts, Harness configuration, or console-based implementation.

The project documentation therefore distinguishes between:

1. **Implemented configuration** — components successfully configured and verified in the AWS Console.
2. **Environment-dependent configuration** — actions requiring IAM or terminal access that were restricted by the provided lab environment.

---

## Current Project Status

### Completed

* Customer-support routing design
* Guardrail configuration
* Classifier configuration
* Bug-report path
* AgentCore managed Harness configuration
* Bug-report system prompt
* AgentCore Gateway configuration
* Bug-report tool schema
* Lambda/DynamoDB persistence configuration
* Platform FAQ path
* Other Request path
* Submission screenshots
* Bug-report evidence and DynamoDB persistence evidence

### Remaining

* Finalize `flow-tests.json`
* Resolve the remaining Gateway/Lambda IAM permission issue if the lab environment permits the required IAM changes

---

## Disclaimer

Marvin, Udacity's AI assistant, was used in a Co-Work capacity during development. The project architecture, ideas, and implementation decisions remain my own, while Marvin was used for discussion, troubleshooting, and implementation guidance.
