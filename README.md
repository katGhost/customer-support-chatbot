# customer-support-chatbot

A customer support chatbot that classifies incoming customer requests and routes them to the appropriate support path.

## Architecture

The chatbot follows a simple routing architecture:

```text
           Customer
              │
              ▼
        ┌───────────┐
        │ Guardrail │
        └─────┬─────┘
              │
    ┌─────────┴─────────┐
    │                   │
   Block              Allowed
    │                   │
    ▼                   ▼
Blocked           Classifier
message                 │
               ┌────────┼────────┐
               ▼        ▼        ▼
              bug    platform   other
```

### 1. Classifier

The classifier determines which category the customer's message belongs to:

* `bug` — bug reports or technical issues
* `platform` — questions about the platform, orders, shipping, returns, or payments
* `other` — requests that do not fall into the Bug Reports or Platform Questions categories

The classifier returns only the applicable label.

### 2. Condition

The classification result determines which support path handles the request.

### 3. Bug Report Path

Bug-related requests are routed to the bug reporting specialist.

The specialist:

1. Acknowledges the reported issue with an apology.
2. Collects the relevant details through conversation.
3. Creates a support ticket.
4. Explains what happens next after ticket creation.

### 4. Platform FAQ Path

Platform questions are answered using the embedded FAQ.

The assistant is restricted to the information contained in the FAQ. If the requested information is not available, the assistant responds:

> "That information is not available in the FAQ."

The assistant does not guess or add information outside the FAQ.

### 5. Other Path

Requests that are neither Bug Reports nor Platform Questions are redirected to the human support helpline.

**Human support helpline:**
`+1 (202) 555-0147`

## Prompts

The prompts used for the chatbot are stored in the `prompts/` directory:

```text
prompts/
├── classifier.md
├── bug-report.md
├── platform-faq.md
├── other.md
├── prompt-injection-test.md
└── tests.md
```

These files contain the prompt text used to configure each stage of the support flow.

## Screenshots

Screenshots documenting the implementation and testing are stored in the `screenshots/` directory.

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

They provide visual evidence of:

* The chatbot flow
* The configured prompts
* Test inputs and outputs

## Credential Blocker

Authorisation through Integrated Workspaces was blocked because the available credentials could not be successfully authorised.

As a workaround, the implementation was completed directly in the AWS console.

The credential issue therefore affected the Integrated Workspaces authorisation path rather than preventing completion of the chatbot implementation in the AWS console.

## Bug-Report Tool Status

The chatbot routing and bug-report path were implemented successfully in the AWS console.

The remaining limitation is the credential-dependent Integrated Workspaces authorisation. Any functionality that specifically requires those credentials remains blocked until valid credentials can be authorised.

The completed implementation and the credential-dependent limitation are documented separately to distinguish implementation status from the external authorisation blocker.

### Disclaimer

Marvin, Udacity's own AI agent helped assisted on some of the work in a "Co-work" manner where ideas and originality of the work is my own but implementation is corrected through discussion with Marvin.
