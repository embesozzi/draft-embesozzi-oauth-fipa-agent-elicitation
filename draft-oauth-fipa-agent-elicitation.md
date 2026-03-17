---
title: "OAuth 2.0 First-Party Native Authorization for AI Agents via Structured Elicitation"
abbrev: "OAuth 2.0 FiPA Agent Elicitation"
docname: draft-embesozzi-oauth-fipa-agent-elicitation
date: 2026-03-09
category: info
submissiontype: independent
ipr: trust200902
v: 3

area: Security
workgroup: Independent Submission

author:
  -
    fullname: "Martin Besozzi"
    organization: TwoGenIdentity
    email: embesozzi@twogenidentity.com

normative:
  RFC2119:
  RFC6749:
  RFC8174:
  RFC9700:
  FiPA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-first-party-apps/
    title: "OAuth 2.0 First-Party Applications"
    author:
      org: IETF OAuth Working Group
  MCP-Elicitation:
    target: https://spec.modelcontextprotocol.io
    title: "Model Context Protocol Specification — Elicitation"
    author:
      org: Anthropic
  WebAuthn:
    target: https://www.w3.org/TR/webauthn/
    title: "Web Authentication: An API for accessing Public Key Credentials, Level 3"
    author:
      org: W3C

informative:
  RFC8628:
  CIBA:
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow — Core 1.0"
    author:
      org: OpenID Foundation
  FIDO-CTAP:
    target: https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html
    title: "Client to Authenticator Protocol (CTAP) 2.2"
    author:
      org: FIDO Alliance
  EAP-ACR:
    target: https://openid.net/specs/openid-connect-eap-acr-values-1_0.html
    title: "OpenID Connect Extended Authentication Profile (EAP) ACR Values 1.0"
    author:
      org: OpenID Foundation

...

--- abstract

AI Agents executing on behalf of users may encounter operations that require
step-up authentication or explicit user approval — a just-in-time (JIT)
authorization gate.  Upon successful completion of the FiPA challenge/response
cycle, the authorization server issues a token that serves as cryptographic
proof of the user's consent, bound to the specific auth session.

This document defines a Structured Elicitation extension to the OAuth 2.0
First-Party Applications (FiPA) specification [FiPA], establishing a standard
metadata format for FiPA authorization challenge responses.  FiPA intentionally
leaves authenticator metadata out of scope: the format in which the
authorization server describes available authenticators and the inputs they
require is undefined.  This gap prevents interoperable implementation by AI
Agents and other non-browser clients.

Model Context Protocol (MCP) Elicitation [MCP-Elicitation] serves as the
reference runtime binding and is the only binding normatively defined in
this specification.

--- middle

# Introduction

## Applicability

AI Agents operate autonomously and do not inherently distinguish routine
operations from sensitive ones.  When an agent encounters an operation
requiring step-up authentication or explicit user approval, a just-in-time
(JIT) authorization gate is needed: the agent must pause, surface a structured
challenge to the human user, and obtain a cryptographic proof — an OAuth token
— before proceeding.  The FiPA specification ([FiPA]) provides the
challenge/response wire protocol for this gate.  This extension defines the
metadata format that makes FiPA challenges machine-readable for agent runtimes.

This extension applies when the client is an AI Agent acting on behalf of a
human user (see Section 1.2), the authorization server supports the FiPA
challenge/response protocol, and the agent runtime supports a Structured
Elicitation mechanism (Section 4), with MCP Elicitation [MCP-Elicitation] as
the normative reference implementation.

## Human-to-Agent (H2A) Communication Model

This extension addresses the Human-to-Agent (H2A) communication pattern.
In this pattern, a human user interacts with an AI Agent
that acts as a first-party OAuth client on their behalf.  The AI Agent
orchestrates the FiPA challenge/response cycle but cannot independently
satisfy authentication challenges that require human interaction — such as
entering a TOTP code or completing a WebAuthn ceremony.

The Structured Elicitation mechanism defined in Section 4 is the channel
through which the authorization server communicates challenge requirements back
to the human user via the agent runtime.  The agent runtime presents the
challenge to the human, collects the response, and forwards it to the
authorization server as an Authorization Challenge Request.

This contrasts with Agent-to-Agent (A2A) patterns, in which an autonomous
agent satisfies authorization challenges without human involvement.  The
present document is scoped to H2A interactions; A2A authorization is outside
the scope of this specification.

## Limitations

This document does not define:

- Capability negotiation between the agent runtime and the authorization
  server. Implementations MUST use application-specific mechanisms until FiPA
  defines a negotiation mechanism.
- Changes to the FiPA wire format beyond the `elicitations` array extension.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

## Terminology

This document uses terminology defined in [RFC6749], [FiPA], and
[MCP-Elicitation]. The following terms are used as defined:

- **Authorization Server (AS):** As defined in [RFC6749].
- **Structured Elicitation:** The abstract mechanism by which an agent runtime
  requests structured user input mid-flow and returns a structured response.
  MCP Elicitation [MCP-Elicitation] is a conformant implementation of this
  mechanism and serves as the normative reference implementation for this
  specification.
- **Human-to-Agent (H2A) Communication:** The interaction pattern in which a
  human user delegates actions to an AI Agent that acts as a first-party OAuth
  client. The agent orchestrates protocol flows on behalf of the user.
- **First-Party AI Agent:** An AI agent whose client runtime is built and
  controlled by the same operator as the server-side components and
  authorization server. The agent MAY implement FiPA-specific elicitation
  extensions.
- **Third-Party AI Agent:** An AI agent whose client runtime is a product
  provided by a third party (e.g., Claude, GitHub Copilot). The implementer
  cannot modify client elicitation handling.

# Protocol Overview

This extension operates within the FiPA challenge/response cycle defined in
[FiPA]. A FiPA authorization request that requires additional authentication
receives an Authorization Challenge Response:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123"
}
~~~

The `auth_session` binds all subsequent requests to this authentication
context. FiPA supports multi-step flows via continued challenge/response cycles
on the same `auth_session`.

This extension adds an `elicitations` array to the Authorization Challenge
Response (Section 4). Each entry in `elicitations` carries a Structured
Elicitation descriptor. When the agent runtime supports MCP Elicitation
[MCP-Elicitation], each entry maps directly to a pending `elicitation/create`
call. The agent runtime acting as the FiPA client translates each entry into a
Structured Elicitation request directed at the human user.

Structured Elicitation defines two modes used by this extension:

- **Form mode** — in-band structured data collection. The server provides a
  `requestedSchema` (a restricted JSON Schema subset); the runtime renders a
  native form and returns structured content. Supported `requestedSchema` types
  are: `string` (with `title`, `description`, `minLength`, `maxLength`,
  `pattern`, `format`, `default`, `enum`, `oneOf`), `number`/`integer`,
  `boolean`, and flat arrays with `enum`/`anyOf`.
- **URL mode** — out-of-band interaction. The server provides a URL the
  runtime opens in a browser or webview. The result is submitted externally;
  the response carries only an `action`.

For URL mode, `mode` and `elicitationId` are REQUIRED in the elicitation
params. URL mode responses carry no `content` — the interaction result is
submitted directly from the browser context to the authorization server.

Upon completion of the challenge/response cycle, the authorization server
issues an authorization code or token bound to the `auth_session`.  This token
serves as cryptographic proof that a human user satisfied the required
authentication challenge at a specific point in time, enabling the MCP server
to enforce JIT authorization before executing sensitive operations.

# Elicitation Extension

## The `elicitations` Array

This extension adds an `elicitations` array to the FiPA
`insufficient_authorization` error response. Each entry carries a Structured
Elicitation descriptor. When the agent runtime supports MCP Elicitation
[MCP-Elicitation], each entry maps directly to a pending `elicitation/create`
call.

Fields per entry:

| Field | Required | Description |
|---|---|---|
| `mode` | Yes | `"form"` or `"url"`. Maps to Structured Elicitation `mode`. |
| `message` | Yes | Human-readable prompt. Maps to Structured Elicitation `message`. |
| `requestedSchema` | Form mode only | JSON Schema for the form. Maps 1:1 to Structured Elicitation `requestedSchema`. |
| `url` | URL mode only | Endpoint URL. Maps to Structured Elicitation `url`. |
| `elicitationId` | URL mode only | Unique ID for out-of-band completion tracking. Maps to Structured Elicitation `elicitationId`. |

### The `response` Parameter

When the client submits an Authorization Challenge Request in response to an
`elicitations` entry, the collected credential fields are carried in a `response`
parameter in the request body. The `response` parameter is a JSON object whose
properties correspond to the field names defined in the `requestedSchema` of the
elicitation entry that prompted the request.

For example, if the `requestedSchema` defines a field named `otp`, the
corresponding Authorization Challenge Request carries:

~~~ json
{ "response": { "otp": "847291" } }
~~~

The `response` parameter is a new parameter introduced by this extension and
MUST be included in all intermediate Authorization Challenge Requests that
carry elicitation content.

### Request Content Type for Intermediate Requests

The initial Authorization Challenge Request MUST use
`application/x-www-form-urlencoded` as required by [FiPA]. Intermediate
requests — those that carry an `auth_session` obtained from a prior challenge
response — MAY use `application/json` as the content type.  This is consistent
with [FiPA], which explicitly permits intermediate requests to use a different
format than the initial request.

## Elicitation Sequencing

The `elicitations` array typically contains one entry at a time. Because
subsequent steps depend on prior user selections (e.g., choosing TOTP versus
Passkey changes the next challenge), the authorization server issues each
step's elicitation in a new Authorization Challenge Response after the
preceding credential is submitted.

## Runtime Bindings

The `elicitations` array is a transport-agnostic FiPA extension. The agent
runtime is responsible for translating each entry into its native interaction
mechanism. This section defines bindings for known runtimes.

### MCP Elicitation Binding

When the agent runtime is MCP-based, the runtime (acting as the FiPA client)
maps each `elicitations` entry directly to an `elicitation/create` params
object per [MCP-Elicitation]. The `jsonrpc`, `id`, and `method` fields are MCP
transport framing — the `params` content is identical to the `elicitations`
entry.

The following example shows a form mode elicitation entry from the
Authorization Challenge Response alongside the resulting `elicitation/create`
call:

FiPA `elicitations` entry (from Authorization Challenge Response):

~~~ json
{
  "mode": "form",
  "message": "Additional verification is required. Select your authentication method.",
  "requestedSchema": {
    "type": "object",
    "properties": {
      "authenticator": {
        "type": "string",
        "title": "Authentication Method",
        "oneOf": [
          { "const": "totp", "title": "Authenticator App (TOTP)" },
          { "const": "passkey", "title": "Passkey" }
        ]
      }
    },
    "required": ["authenticator"]
  }
}
~~~

Resulting MCP `elicitation/create` call issued by the MCP server:

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-1",
  "method": "elicitation/create",
  "params": {
    "mode": "form",
    "message": "Additional verification is required. Select your authentication method.",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "authenticator": {
          "type": "string",
          "title": "Authentication Method",
          "oneOf": [
            { "const": "totp", "title": "Authenticator App (TOTP)" },
            { "const": "passkey", "title": "Passkey" }
          ]
        }
      },
      "required": ["authenticator"]
    }
  }
}
~~~

MCP client response:

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-1",
  "result": {
    "action": "accept",
    "content": { "authenticator": "totp" }
  }
}
~~~

#### MCP Elicitation Flow Example

~~~
   Human        AI Agent        MCP Server        Authorization Server
     |               |               |                      |
     |  Prompt       |               |                      |
     |-------------->|               |                      |
     |               |  execute      |                      |
     |               |  MCP tool     |                      |
     |               |-------------->|                      |
     |               |               |  Authorization       |
     |               |               |  Request             |
     |               |               |--------------------->|
     |               |               |                      |
     |               |               |  Authorization       |
     |               |               |  Challenge Error     |
     |               |               |  + Elicitations      |
     |               |               |<---------------------|
     |               |               |                      |
     |               |               |  MCP Elicitation     |
     |               |<--------------|                      |
     |               |               |                      |
     |  Input        |               |                      |
     |-------------->|               |                      |
     |               |  Elicitation  |                      |
     |               |  Response     |                      |
     |               |-------------->|                      |
     |               |               |  Authorization       |
     |               |               |  Challenge Request   |
     |               |               |--------------------->|
     |               |               |                      |
     |               |               |  Authorization       |
     |               |               |  Challenge Error     |
     |               |               |  + Elicitations      |
     |               |               |<---------------------|
     |               |               |                      |
     |               |               |  ... (repeat as      |
     |               |               |  needed)             |
     |               |               |                      |
     |               |               |  Authorization       |
     |               |               |  Code Response       |
     |               |               |<---------------------|
     |               |               |                      |
     |               |               |  Token Request       |
     |               |               |--------------------->|
     |               |               |                      |
     |               |               |  Token Response      |
     |               |               |<---------------------|
     |               |               |                      |
~~~



### HTTP API Binding

TODO: future section

## Clients Without Structured Elicitation Support

For clients that do not support Structured Elicitation, the authorization
server returns a standard OAuth error response per [RFC6749] §5.2. The
`WWW-Authenticate` response header per [RFC6749] §5.2 is one example of how
the error is surfaced at the HTTP level:

~~~ http
WWW-Authenticate: Bearer realm="authorization-server",
                  error="insufficient_scope",
                  auth_session="sess_abc123"
~~~

# Authenticator Selection

The following sections define the FiPA wire format for each authenticator type.
Runtime-specific translation (e.g., MCP Elicitation) is defined in Section 4.3.

When additional authentication is required, the first elicitation presents the
available authenticators. This specification defines elicitation schemas for
two authenticator types: Passkey (WebAuthn) and Authenticator App (TOTP).
Password authentication is outside the scope of this document.

## Authorization Challenge Response

When multiple authenticators are supported, the authorization server returns a
selection prompt:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "form",
      "message": "Additional verification is required. Select your authentication method.",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "authenticator": {
            "type": "string",
            "title": "Authentication Method",
            "oneOf": [
              { "const": "totp", "title": "Authenticator App (TOTP)" },
              { "const": "passkey", "title": "Passkey" }
            ]
          }
        },
        "required": ["authenticator"]
      }
    }
  ]
}
~~~

When only one authenticator is supported, the authorization server directly
requests the required credential without presenting a choice:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "form",
      "message": "Additional verification is required. Enter your one-time passcode.",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "totp_code": {
            "type": "string",
            "title": "One-Time Passcode",
            "description": "Enter the 6-digit code from your authenticator app."
          }
        },
        "required": ["totp_code"]
      }
    }
  ]
}
~~~

## Authorization Challenge Request

~~~ http
POST /as/authorization.oauth2
Content-Type: application/json

{
  "auth_session": "sess_abc123",
  "response": { "authenticator": "totp" }
}
~~~

# TOTP Challenge

TOTP uses form mode: the challenge is a 6-digit numeric code, expressible as a
plain string with `pattern` and length constraints.

## Authorization Challenge Response

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "form",
      "message": "Enter the 6-digit code from your Authenticator App.",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "otp": {
            "type": "string",
            "title": "One-Time Password",
            "minLength": 6,
            "maxLength": 6,
            "pattern": "^[0-9]{6}$"
          }
        },
        "required": ["otp"]
      }
    }
  ]
}
~~~

## Authorization Challenge Request

~~~ http
POST /as/authorization.oauth2
Content-Type: application/json

{
  "auth_session": "sess_abc123",
  "response": { "otp": "847291" }
}
~~~

On success, the authorization server issues an OAuth token with the
authenticated ACR value, per [FiPA].

# Passkey Challenge

In a Third-Party AI Agent deployment, the client runtime is a product the
implementer does not control. It supports standard Structured Elicitation form
mode and URL mode but cannot be extended with custom format handling or WebAuthn
API invocation.
For Third-Party AI Agents, URL mode SHOULD be used to redirect the user to a
browser-based WebAuthn flow. The out-of-band WebAuthn ceremony is outside the
scope of this specification. 

For in-band Passkey support, a First-Party AI Agent deployment is required.
The implementer controls the agent code and MAY implement custom elicitation
handling, including native WebAuthn API invocation using the OS or platform
FIDO2 APIs ([FIDO-CTAP]). The WebAuthn ceremony is performed in-band: no
browser, no redirect.

## WebAuthn Challenge Delivery

This section defines how the authorization server delivers WebAuthn challenge
parameters to the agent through Structured Elicitation form mode.

TODO

## Authorization Challenge Response

TODO

# Security Considerations

TODO Security

# IANA Considerations
This document has no IANA actions.

--- back
