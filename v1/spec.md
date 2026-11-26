# A2A Protocol Extension: Structured Input (v1 Draft)

- **URI:** `https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1`
- **Type:** Profile Extension
- **Version:** 1.0.0
- **Authors:** [Yaroslav Shevchuk](https://github.com/yarolegovich) (Google)

## Abstract

Agent frameworks implementing Human-In-The-Loop flows, such as tool call confirmations, need to
request input from users in a format the framework can consume. Adapters exposing such agents over
A2A need a way to ask clients for this input and to receive it back in a recognizable form.

This extension enables interoperable structured input handling in an SDK-agnostic fashion: agents
request input with a JSON Schema while the task is in the `input-required` state, and resume only
when the client provides data satisfying it. The extension is designed so that even
extension-unaware clients can participate in structured data exchanges.

## 1. Overview

This extension provides a standardized mechanism for agents to use `input-required` state for
requesting structured inputs from clients. An agent provides a JSON Schema, and the task is not
resumed until the client responds with data validating against that schema. We
explicitly define the error-handling loop where invalid input triggers a re-entry into the
`input-required` with additional validation feedback attached.

## 2. Extension URI

- **URI:** `https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1`
- **Constraint:** This is the only URI accepted for this extension.

## 3. Core Concepts

- **Schema Definition:** A JSON Schema object defining the expected structure for the requested
  input.
- **Input Request:** `Task` transition to `input-required` state where task status message
  contains a schema definition. An input request can have an ID which distinguishes it from other
  input requests within its task. A task has at most one pending input request at a time.
- **Continuation Request:** `Message` carrying the requested structured data sent for triggering
  task continuation.
- **Validation Loop:** Task state transition cycle in which the main task flow does not resume
  until continuation request strictly satisfies the schema definition.
- **Answer Source:** Who provided the structured input: a person, a rule configured by a person,
  or the client itself. An input request can require that a person provides the input.
- **Intermediary:** An agent which is the client of a task that issued an input request, while it
  serves a task for its own client.

## 4. Schemas

This extension relies on standard JSON Schema wrapped in specific A2A DataPart envelopes. The
specific JSON Schema draft version used by the agent is declared in the extension parameters
(e.g., `2020-12`, `draft-07`). If a schema declares its dialect with the `$schema` keyword, the
`$schema` value takes precedence over the extension parameter for that schema.

Validation errors are always reported in the JSON Schema Basic output format (introduced in
draft 2019-09), regardless of the draft used for validation.

## 5. Activation

Agents **MUST** advertise this extension in their agent card to signal that they **MAY** enter a
structured `input-required` state. Valid extension parameters **SHOULD** be included to specify
the JSON Schema draft version.

```json
{
  "capabilities": {
    "extensions": [
      {
        "uri": "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1",
        "description": "Standardizes schema-based input requests.",
        "params": {
          "jsonschema_draft": "2020-12"
        },
        "required": false
      }
    ]
  }
}
```

- `jsonschema_draft`: (Optional) The JSON Schema draft version the agent uses for validation
  (e.g., "2020-12", "draft-07"). Defaults to "2020-12" if omitted.

The extension **SHOULD NOT** be marked as required, because an extension-unaware client can still
understand the structured data request and generate a properly formatted continuation response
(see [6.4](#64-untagged-input)). The exception is an input request which requires an answer from a
person (see [6.5](#65-answer-source)): an extension-unaware client cannot answer it.

Extension-aware clients **SHOULD** activate the extension by including its URI in the
`A2A-Extensions` service parameter of their requests. Agents **MAY** use the extension in a task
even if the client did not activate it.

## 6. Data Definitions

This extension uses standard `DataPart` objects to exchange schemas, validation errors, and
structured input. To ensure extension-aware clients can correctly identify these parts, all
related `DataPart` objects **MUST** include the extension URI in their `metadata` field, with the
exception described in [6.4](#64-untagged-input).

### 6.1 Metadata Structure

The extension metadata object **MUST** contain a `type` property indicating the part's role.
Parts of type `input_schema` and `structured_input` **MAY** contain an `id` property identifying
the input request (see [6.3](#63-request-correlation)).

```json
"metadata": {
  "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
    "type": "<PART_TYPE>",
    "id": "<INPUT_REQUEST_ID>"
  }
}
```

Parts of type `input_schema` **MAY** contain the `answer_from` and `asked_by` properties, and parts
of type `structured_input` **MAY** contain the `answered_by` property (see
[6.5](#65-answer-source) and [8](#8-intermediaries)). Receivers **MUST** ignore properties of the
extension metadata object which they do not recognize.

### 6.2 Part Types

The `type` property accepts the following values:

- **`input_schema`** (Agent -> Client) Contains the JSON Schema defining the expected input
  structure. *Optional:* Agents **MAY** set the part's `mediaType` field to
  `application/schema+json` to assist extension-unaware clients in interpreting the content.
- **`validation_error`** (Agent -> Client) Contains validation errors (conforming to the JSON
  Schema Basic output format) when a client submits invalid data.
- **`structured_input`** (Client -> Agent) Contains the data object that satisfies the requested
  schema. This explicit tagging is **REQUIRED** for extension-aware clients so the agent can
  distinguish the structured payload from other context (like text).

A status message **MUST NOT** contain more than one `input_schema` part, and a continuation
request **MUST NOT** contain more than one `structured_input` part. Agents which need several inputs
at once (for example, for parallel tool calls) **SHOULD** combine them into a single schema.

### 6.3 Request Correlation

Agents **SHOULD** set an `id` when a late or duplicated answer could be mistaken for the answer to
a later input request, for example when consecutive input requests in a task share a schema. The
`id` is chosen by the agent and **MUST NOT** be reused by a different input request within the
task. It identifies the input request for the whole validation loop: a repeated input request keeps
its `id`, even if its schema was updated.

If the pending input request has an `id`, extension-aware clients **SHOULD** copy it into the
`structured_input` metadata. If a `structured_input` part carries an `id` which does not match the
pending input request, the agent **MUST NOT** use it as the input and **MUST** treat the
continuation request as invalid (see [7](#7-validation-loop)). A `structured_input` part without an
`id` is matched against the pending input request by its schema alone.

Input without an `id` (including untagged input accepted under [6.4](#64-untagged-input)) cannot be
told apart from a late answer to an earlier input request with the same schema. Agents which need
this guarantee **SHOULD NOT** accept input without an `id` for input requests which have one, at the
cost of not supporting extension-unaware clients for those requests.

### 6.4 Untagged Input

Extension-unaware clients cannot be expected to add the extension metadata. An agent **MAY**
accept a `DataPart` without the extension metadata as the structured input if all of the following
hold:

1. The continuation request contains no `structured_input` part.
1. The continuation request contains exactly one `DataPart`.
1. The data validates against the schema of the pending input request.

An untagged part which does not validate against the schema **MUST** be treated as context, not as
the structured input. If conditions 1 and 2 hold, the agent **MAY** still include validation errors
for that part in the repeated input request (see [7](#7-validation-loop)), since an
extension-unaware client cannot otherwise learn why its input was not accepted.

### 6.5 Answer Source

Some input must come from a person, for example the approval of a payment or of a deployment. When
the client of the task is itself an agent, its model can produce input which satisfies the schema,
and the agent which asked cannot tell the difference. The properties in this section let the agent
state the requirement and let the client state who provided the input.

**`answer_from`** (on `input_schema`, optional). The only defined value is `person`: the agent
requires that a person gives or confirms the structured input for this input request. If the
property is absent, the client decides how the input is produced. A repeated input request keeps
its `answer_from` value. Other values are reserved. A client which does not recognize the value
**MUST NOT** produce the input automatically.

**`answered_by`** (on `structured_input`, optional). One of:

- `person`: a person gave or confirmed this input in response to this input request.
- `policy`: a rule which a person configured in advance produced this input (for example
  "always allow this tool").
- `agent`: the client produced this input itself, for example with a model.

No rule in this extension depends on the difference between `policy` and `agent`. The two values
let an agent record how the input was produced.

A client **MUST NOT** set `answered_by` to `person` unless a person gave or confirmed the input in
response to this input request, or in response to an input request relayed from it (see
[8](#8-intermediaries)). A client **SHOULD** set `answered_by` when the input request has
`answer_from`, and **MAY** set it on any other `structured_input` part.

If the pending input request has `answer_from` set to `person`:

1. A client which cannot obtain the input from a person **MUST NOT** provide the input. It **MAY**
   cancel the task or leave the input request pending. A client which is an intermediary follows
   [8](#8-intermediaries).
1. The agent **MUST NOT** use a `structured_input` part whose `answered_by` is absent or is not
   `person`, and **MUST** treat the continuation request as invalid (see
   [7](#7-validation-loop)). The agent **SHOULD** explain the rejection in a text part, because a
   client which does not know `answer_from` cannot otherwise learn why its input was not accepted.
1. The agent **MUST NOT** accept untagged input under [6.4](#64-untagged-input), because untagged
   input cannot carry `answered_by`. Such requests are not available to extension-unaware clients.

`answered_by` is a statement made by the client. The agent cannot verify it, and it **MUST NOT**
be treated as proof that a person was involved. It prevents a correctly implemented client from
answering in place of a person by mistake. It does not protect against a client which makes a
false statement. Verifiable evidence of human approval is outside the scope of this extension.

### 6.6 Examples

**A. Agent Requests Input (Schema)**

```json
{
  "data": {
    "type": "object",
    "properties": { "foo": { "type": "string" } }
  },
  "mediaType": "application/schema+json",
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "input_schema",
      "id": "req-1"
    }
  }
}
```

**B. Client Provides Input (Structured Data)**

```json
{
  "data": { "foo": "bar" },
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "structured_input",
      "id": "req-1"
    }
  }
}
```

**C. Agent Requests Input From a Person**

```json
{
  "data": {
    "type": "object",
    "properties": { "approved": { "type": "boolean" } },
    "required": ["approved"]
  },
  "mediaType": "application/schema+json",
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "input_schema",
      "id": "charge-1",
      "answer_from": "person"
    }
  }
}
```

**D. Client Provides Input Given by a Person**

```json
{
  "data": { "approved": true },
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "structured_input",
      "id": "charge-1",
      "answered_by": "person"
    }
  }
}
```

## 7. Validation Loop

When continuation request is received the agent **MAY** acknowledge it by publishing a
`TaskStatusUpdate` with state `working`.

It then **MUST** validate that the structured input provided in the request (a `structured_input`
part, or an untagged part accepted under [6.4](#64-untagged-input)) satisfies the schema
definition from the input request. The agent checks the `id` (see
[6.3](#63-request-correlation)) and the answer source (see [6.5](#65-answer-source)) before the
schema. Input which fails one of these two checks is not validated against the schema.

If the validation fails, but the request contains other parts (e.g. text), the agent **SHOULD**
process these parts if it is able to while the input request is pending. This allows the client to
ask clarifying questions without exiting the validation loop or canceling the task. An agent which
cannot act on additional parts while the input request is pending (for example, because its
execution is suspended inside a tool call awaiting the input) **MAY** ignore them. Additional parts
**MUST NOT** be interpreted as the structured input.

If the input was invalid, missing, correlated with a different input request (see
[6.3](#63-request-correlation)), or not from the required answer source (see
[6.5](#65-answer-source)), the agent **MUST** publish a new structured input request which:

1. **MUST** contain the original (or updated) schema definition.
1. **SHOULD** contain validation errors conforming to the JSON Schema Basic output format if the
   client provided a `structured_input` part which failed schema validation.
1. **MAY** contain additional parts generated by the agent in response to the client's clarifying
   questions.

The new input request keeps the `id` of the original one, if it had one.

A `structured_input` part rejected because of an `id` mismatch or because of its answer source was
not validated against the schema, so the agent **SHOULD NOT** report it with a `validation_error`
part. The agent **MAY** explain the rejection in a text part instead.

## 8. Intermediaries

An intermediary is the client of one task (the downstream task) and serves another task for its own
client (the upstream task). For example, a travel agent which works for a personal assistant calls
an airline agent, and the airline agent asks for the approval of a charge.

If the downstream input request has no `answer_from`, the intermediary **MAY** provide the input
itself or relay the input request.

If the downstream input request has `answer_from` set to `person`, the intermediary **MUST NOT**
produce the input with a model or any other automated component. It **MUST** do one of the
following:

- obtain the input from a person directly, if it has the means to do so;
- relay the input request to its own client, as described below;
- provide no input, and cancel the downstream task or leave the input request pending. This is the
  only option when no person can be reached, for example when a schedule started the upstream task.

To relay an input request, the intermediary:

1. **MUST** publish an input request in the upstream task with the schema definition and the
   `answer_from` value of the downstream input request.
1. **MUST NOT** assume that the downstream `id` is valid in the upstream task. An `id` is unique
   only within one task (see [6.3](#63-request-correlation)), so the intermediary chooses an `id`
   for the upstream input request and keeps the association between the two.
1. **SHOULD** set `asked_by` to a name by which a person can identify the agent that issued the
   downstream input request, for example the `name` from its agent card. If the downstream input
   request already has `asked_by`, the intermediary **MUST** keep that value. `asked_by` is a
   statement made by the intermediary. A client **MUST NOT** present it as a verified identity.
1. **SHOULD** include the parts of the downstream status message which explain the request, because
   the person needs them to decide.

When the intermediary receives the continuation request in the upstream task, it processes it as
any agent does (see [7](#7-validation-loop)). If the input is accepted, the intermediary sends a
continuation request to the downstream task with a `structured_input` part which carries:

1. the `id` of the downstream input request;
1. the data it received, unchanged;
1. the `answered_by` value it received, unchanged. When it relays, the intermediary **MUST NOT**
   add or change `answered_by`.

If the downstream agent repeats its input request, the intermediary repeats the upstream input
request under the same upstream `id`.

**Example.** The airline agent publishes this part in the downstream task:

```json
{
  "data": {
    "type": "object",
    "properties": { "approved": { "type": "boolean" } },
    "required": ["approved"]
  },
  "mediaType": "application/schema+json",
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "input_schema",
      "id": "charge-1",
      "answer_from": "person"
    }
  }
}
```

The travel agent publishes this part in the upstream task:

```json
{
  "data": {
    "type": "object",
    "properties": { "approved": { "type": "boolean" } },
    "required": ["approved"]
  },
  "mediaType": "application/schema+json",
  "metadata": {
    "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
      "type": "input_schema",
      "id": "relay-7",
      "answer_from": "person",
      "asked_by": "Airline Agent"
    }
  }
}
```

The personal assistant shows the question to its user and sends the answer with `"id": "relay-7"`
and `"answered_by": "person"`. The travel agent sends the same data to the airline agent with
`"id": "charge-1"` and `"answered_by": "person"`.

## 9. Example

1. **Input required**

The agent initiates a sensitive operation (deploying to production) and requires explicit human
confirmation. It sends the details of the intended tool call, the intermediate response (ticket
created), and a schema enforcing a strict `FunctionResponse` structure for the final approval.

```json
{
  "status": {
    "state": "input-required",
    "message": {
      "role": "agent",
      "parts": [
        {"text": "A deployment requires manual verification. I have initiated a call to request_deployment_approval:"},
        {
          "data": {
            "service": "payment-processing",
            "version": "v2.5.0-rc1",
            "region": "us-east-1"
          }
        },
        {"text": "And got the following result:"},
        {
          "data": {
            "status": "pending",
            "ticket_id": "TICKET-4092"
          }
        },
        {"text": "Please provide the approval response matching the following schema:"},
        {
          "data": {
            "type": "object",
            "title": "request_deployment_approval_response",
            "properties": {
              "status": { "type": "string" },
              "ticket_id": { "type": "string" },
              "approver_id": { "type": "string" }
            },
            "required": ["status", "ticket_id", "approver_id"]
          },
          "mediaType": "application/schema+json",
          "metadata": {
            "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
              "type": "input_schema",
              "id": "call_prod_deploy_99a"
            }
          }
        }
      ]
    }
  }
}
```

2. **Missing Information**

The client (or human operator) approves the deployment but fails to provide the `approver_id` in
the response.

```json
{
  "role": "user",
  "parts": [
    {"text": "Deployment approved."},
    {
      "data": {
        "status": "approved",
        "ticket_id": "TICKET-4092"
      },
      "metadata": {
        "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
          "type": "structured_input",
          "id": "call_prod_deploy_99a"
        }
      }
    }
  ]
}
```

3. **Validation Feedback**

The agent matches the input against the schema, detects the missing `approver_id`, and responds
with a new `input-required` status under the same input request `id`. This response includes the
validation errors in the standard basic output format, guiding the client to fix the issue.

```json
{
  "status": {
    "state": "input-required",
    "message": {
      "role": "agent",
      "parts": [
        {"text": "A deployment requires manual verification. I have initiated a call to request_deployment_approval:"},
        {
          "data": {
            "service": "payment-processing",
            "version": "v2.5.0-rc1",
            "region": "us-east-1"
          }
        },
        {"text": "And got the following result:"},
        {
          "data": {
            "status": "pending",
            "ticket_id": "TICKET-4092"
          }
        },
        {"text": "Please provide the approval response matching the following schema:"},
        {
          "data": {
            "type": "object",
            "title": "request_deployment_approval_response",
            "properties": {
              "status": { "type": "string" },
              "ticket_id": { "type": "string" },
              "approver_id": { "type": "string" }
            },
            "required": ["status", "ticket_id", "approver_id"]
          },
          "mediaType": "application/schema+json",
          "metadata": {
            "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
              "type": "input_schema",
              "id": "call_prod_deploy_99a"
            }
          }
        },
        {"text": "Your response did not match the schema:"},
        {
          "data": {
            "valid": false,
            "errors": [
              {
                "keywordLocation": "/required",
                "instanceLocation": "",
                "error": "missing properties: 'approver_id'"
              }
            ]
          },
          "metadata": {
            "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
              "type": "validation_error"
            }
          }
        }
      ]
    }
  }
}
```

4. **Correct Input**

The client provides a `DataPart` that strictly conforms to the requested schema, now including the
`approver_id`.

```json
{
  "role": "user",
  "parts": [
    {"text": "Deployment approved by alice."},
    {
      "data": {
        "status": "approved",
        "ticket_id": "TICKET-4092",
        "approver_id": "ldap:alice"
      },
      "metadata": {
        "https://github.com/a2aproject/experimental-ext-structured-input/tree/main/v1": {
          "type": "structured_input",
          "id": "call_prod_deploy_99a"
        }
      }
    }
  ]
}
```

5. **Task completion**

The agent receives the valid structured input, treats it as the result of the
`request_deployment_approval` tool, and proceeds to complete the task.

```json
{
  "status": {
    "state": "completed",
    "message": {
      "role": "agent",
      "parts": [
        {"text": "Deployment initiated successfully for payment-processing v2.5.0-rc1."}
      ]
    }
  }
}
```
