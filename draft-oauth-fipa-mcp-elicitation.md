---
title: "OAuth 2.0 Native Authorization based MCP Elicitation"
abbrev: "OAuth 2.0 FiPA MCP Elicitation"
docname: draft-embesozzi-oauth-fipa-mcp-elicitation
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
  RFC8628:
  RFC9700:
  CIBA:
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow — Core 1.0"
    author:
      org: OpenID Foundation
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
  FIDO-CTAP:
    target: https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html
    title: "Client to Authenticator Protocol (CTAP) 2.2"
    author:
      org: FIDO Alliance

...

--- abstract

The OAuth 2.0 First-Party Applications (FiPA) specification defines a
challenge/response protocol enabling API-native authentication without browser
redirects.  FiPA intentionally leaves authenticator metadata out of scope: the
format in which the authorization server describes available authenticators and
the inputs they require is undefined.  This gap prevents interoperable
implementation by AI Agents, CLI tools, and autonomous agents.

This document proposes an extension to FiPA that adopts MCP Elicitation as the
standard metadata language for FiPA challenges.  The scope covers two strong
authenticator types: Authenticator Apps (TOTP) and Passkeys (WebAuthn).
Password authentication is explicitly out of scope.  The same pattern is
extensible to other authenticator types by defining additional requestedSchema
structures.

The extension is analyzed across two deployment types: Third-Party AI Agents
(e.g., Claude, GitHub Copilot), where the MCP client is provided by a third
party and cannot be modified; and First-Party AI Agents, where the implementer
controls the agent code.  Both deployment types share the same FiPA
challenge/response wire format and MCP Elicitation protocol, differing only in
Passkey handling.

--- middle

# 1. Introduction

## 1.1 Applicability

The FiPA specification ([FiPA]) enables first-party clients to authenticate users API-natively. The client submits credentials directly to the authorization server and receives structured challenge responses rather than redirect URLs.

FiPA does not define:

- The format for advertising which authenticators are available
- The input schema each authenticator challenge requires
- How multi-step flows (authenticator selection → credential entry) are sequenced

Without a standard for this metadata, every authorization server invents its own format. AI Agents and agent runtimes cannot generically handle FiPA challenges without server-specific integration.

This extension applies to deployments where:

- The client is an AI Agent or autonomous agent communicating over MCP.
- The authorization server supports the FiPA challenge/response protocol.
- The agent runtime supports MCP Elicitation as defined in [MCP-Elicitation].

## 1.2 Limitations

This document does not define:

- Password authentication. Password credentials MUST NOT be included in the `elicitations` array.
- Capability negotiation between the MCP server and the authorization server. Implementations MUST use application-specific mechanisms until FiPA defines a negotiation mechanism.
- A new first-class MCP Elicitation mode for WebAuthn. The `"format": "webauthn-get"` mechanism defined in Section 8 is a FiPA-specific extension over the existing form mode.
- Changes to the FiPA wire format beyond the `elicitations` array extension.

## 1.3 Deployment Types

This document organizes the extension around two deployment types.

**Third-Party AI Agent**

The MCP client is a third-party product (e.g., Claude, GitHub Copilot in VS Code). The implementer controls the MCP server and authorization server but cannot modify the MCP client. The client supports standard MCP Elicitation form mode and URL mode as defined in [MCP-Elicitation]. No custom elicitation handling can be added.

For Passkeys, the MCP client cannot invoke a WebAuthn API. The only available path is URL mode — an out-of-band browser interaction.

**First-Party AI Agent**

The MCP client is an agent the implementer builds and controls. It can implement custom handling for FiPA-specific elicitation extensions, including invoking the platform WebAuthn API. MCP Elicitation is used as the delivery channel for WebAuthn challenge metadata.

For Passkeys, the agent can perform the WebAuthn ceremony in-band, with challenge parameters delivered through a FiPA extension to the MCP form mode `requestedSchema`. No browser redirect is required.

# 2. Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

## 2.1 Terminology

This document uses terminology defined in [RFC6749], [FiPA], and
[MCP-Elicitation]. The following terms are used as defined:

- **Authorization Server (AS):** As defined in [RFC6749].
- **First-Party AI Agent:** An AI agent whose MCP client is built and
  controlled by the same operator as the MCP server and authorization
  server. The agent MAY implement FiPA-specific elicitation extensions.
- **Third-Party AI Agent:** An AI agent whose MCP client is a product
  provided by a third party (e.g., Claude, GitHub Copilot). The
  implementer cannot modify client elicitation handling.
- **MCP Elicitation:** The mechanism defined in [MCP-Elicitation] by
  which an MCP server requests structured user input mid-flow.
- **Authorization Challenge Response:** An HTTP 400 response from the
  authorization server carrying `"error": "insufficient_authorization"`
  and an `auth_session` binding token, as defined in [FiPA].
- **Authorization Challenge Request:** An HTTP POST from the MCP server
  to the authorization server carrying an `auth_session` and
  credential response, as defined in [FiPA].

# 3. Protocol Overview

This extension operates within the FiPA challenge/response cycle defined in [FiPA]. A FiPA authorization request that requires additional authentication receives an Authorization Challenge Response:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123"
}
~~~

The `auth_session` binds all subsequent requests to this authentication context. FiPA supports multi-step flows via continued challenge/response cycles on the same `auth_session`.

This extension adds an `elicitations` array to the Authorization Challenge Response (Section 4). Each entry in `elicitations` maps directly to a pending `elicitation/create` call per [MCP-Elicitation]. The MCP server acting as the FiPA client translates each entry into an MCP Elicitation request directed at the MCP client.

MCP Elicitation defines two modes used by this extension:

- **Form mode** — in-band structured data collection. The server provides a `requestedSchema` (a restricted JSON Schema subset); the MCP client renders a native form and returns structured content. Supported `requestedSchema` types are: `string` (with `title`, `description`, `minLength`, `maxLength`, `pattern`, `format`, `default`, `enum`, `oneOf`), `number`/`integer`, `boolean`, and flat arrays with `enum`/`anyOf`.
- **URL mode** — out-of-band interaction. The server provides a URL the MCP client opens in a browser or webview. The result is submitted externally; the MCP response carries only an `action`.

For URL mode, `mode` and `elicitationId` are REQUIRED in the `elicitation/create` params. URL mode responses carry no `content` — the interaction result is submitted directly from the browser context to the authorization server.

# 4. Elicitation Extension

## 4.1 The `elicitations` Array

This extension adds an `elicitations` array to the FiPA `insufficient_authorization` error response. Each entry maps directly to a pending `elicitation/create` call.

Fields per entry:

| Field | Required | Description |
|---|---|---|
| `mode` | Yes | `"form"` or `"url"`. Maps to MCP Elicitation `mode`. |
| `message` | Yes | Human-readable prompt. Maps to MCP Elicitation `message`. |
| `requestedSchema` | Form mode only | JSON Schema for the form. Maps 1:1 to MCP Elicitation `requestedSchema`. |
| `url` | URL mode only | Endpoint URL. Maps to MCP Elicitation `url`. |
| `elicitationId` | URL mode only | Unique ID for out-of-band completion tracking. Maps to MCP Elicitation `elicitationId`. |

## 4.2 Elicitation Sequencing

The `elicitations` array typically contains one entry at a time. Because subsequent steps depend on prior user selections (e.g., choosing TOTP versus Passkey changes the next challenge), the authorization server issues each step's elicitation in a new Authorization Challenge Response after the preceding credential is submitted.

## 4.3 Mapping to MCP Wire Format

The MCP server (acting as the FiPA client) maps each `elicitations` entry directly to an `elicitation/create` params object per [MCP-Elicitation].

## 4.4 Non-MCP Fallback

For clients that do not support MCP Elicitation, the authorization server returns a standard OAuth error response per [RFC6749] §5.2. The `WWW-Authenticate` response header per [RFC6749] §5.2 is one example of how the error is surfaced at the HTTP level:

~~~ http
WWW-Authenticate: Bearer realm="authorization-server",
                  error="insufficient_scope",
                  auth_session="sess_abc123"
~~~

# 5. Authenticator Selection

When additional authentication is required, the first elicitation presents the available strong authenticators. Only phishing-resistant (Passkey/WebAuthn) and strong second-factor (TOTP) options are offered. Password authentication MUST NOT be included.

## 5.1 Authorization Challenge Response

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

## 5.2 MCP Elicitation Request

MCP server issues:

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

## 5.3 Authorization Challenge Request

~~~ http
POST /as/authorization.oauth2
Content-Type: application/json

{
  "auth_session": "sess_abc123",
  "response": { "authenticator": "totp" }
}
~~~

# 6. TOTP Challenge

TOTP works identically for both Third-Party and First-Party AI Agents. Form mode is sufficient: the challenge is a 6-digit numeric code, expressible as a plain string with `pattern` and length constraints.

## 6.1 Authorization Challenge Response

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

## 6.2 MCP Elicitation Request

MCP server issues:

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-2",
  "method": "elicitation/create",
  "params": {
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
}
~~~

MCP client response:

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-2",
  "result": {
    "action": "accept",
    "content": { "otp": "847291" }
  }
}
~~~

## 6.3 Authorization Challenge Request

~~~ http
POST /as/authorization.oauth2
Content-Type: application/json

{
  "auth_session": "sess_abc123",
  "response": { "otp": "847291" }
}
~~~

On success, the authorization server issues an OAuth token with the authenticated ACR value, per [FiPA].

# 7. Passkey Challenge — Third-Party AI Agent

In a Third-Party AI Agent, the MCP client is a product the implementer does not control. It supports standard MCP Elicitation form mode and URL mode but cannot be extended with custom format handling or WebAuthn API invocation.

WebAuthn requires a cryptographic ceremony: the authorization server generates a `challenge` nonce and `PublicKeyCredentialRequestOptions`; the client invokes the platform WebAuthn API; the platform authenticator returns a `PublicKeyCredential` assertion. MCP Elicitation form mode cannot express this ceremony within its supported `requestedSchema` primitive types. For Third-Party AI Agents, only URL mode is available for Passkeys.

## 7.1 Authorization Challenge Response

The authorization server includes a URL mode elicitation entry for the Passkey step. The `url` points to an HTTPS endpoint that drives the full WebAuthn ceremony in a browser or webview. The WebAuthn assertion is submitted directly from the browser to the authorization server — it does not transit the MCP channel.

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "insufficient_authorization",
  "auth_session": "sess_abc123",
  "elicitations": [
    {
      "mode": "url",
      "message": "Complete Passkey authentication in the browser window.",
      "url": "https://auth.example.com/webauthn/begin?auth_session=sess_abc123",
      "elicitationId": "el-passkey-ent-001"
    }
  ]
}
~~~

## 7.2 MCP Elicitation Request

MCP server issues:

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-passkey",
  "method": "elicitation/create",
  "params": {
    "mode": "url",
    "message": "Complete Passkey authentication in the browser window.",
    "url": "https://auth.example.com/webauthn/begin?auth_session=sess_abc123",
    "elicitationId": "el-passkey-ent-001"
  }
}
~~~

MCP client response (after browser interaction completes):

~~~ json
{
  "jsonrpc": "2.0",
  "id": "elicit-passkey",
  "result": {
    "action": "accept"
  }
}
~~~

## 7.3 Out-of-Band Completion

Once the browser opens the URL, the WebAuthn ceremony is handled entirely between the browser and the authorization server per [WebAuthn]. This is out of scope for this draft — the MCP server and this extension play no role in it.

## 7.4 Limitations

| Limitation | Impact |
|---|---|
| Requires user browser access | The user MUST be able to open the URL in a browser. CLI clients can present the URL for manual opening, following the device flow pattern per [RFC8628]. |
| Context switch | User leaves the AI Agent interface to complete the ceremony. |
| Out-of-band result | The WebAuthn assertion does not transit the MCP channel. The MCP server MUST poll the authorization server (per [RFC8628] or [CIBA]) until the ceremony completes and a token is issued. |

# 8. Passkey Challenge — First-Party AI Agent

In a First-Party AI Agent, the implementer controls the agent code. The agent can implement custom elicitation handling, including native WebAuthn API invocation using the OS or platform FIDO2 APIs ([FIDO-CTAP]). The WebAuthn ceremony is performed in-band: no browser, no redirect.

## 8.1 WebAuthn Challenge Delivery

This section defines how the authorization server delivers WebAuthn challenge parameters to the agent through MCP Elicitation form mode.

Form mode `requestedSchema` supports string properties with `format` and `x-` (extension) properties. JSON Schema permits `x-` prefixed extension keywords. Although `"format": "webauthn-get"` is not defined in [MCP-Elicitation], a First-Party agent can implement recognition of this extension format as a FiPA-specific capability.

The authorization server encodes all `PublicKeyCredentialRequestOptions` fields ([WebAuthn] §5.5) as `x-webauthn-*` extension properties on the assertion field. The field names mirror the W3C WebAuthn Level 3 specification [WebAuthn].

~~~ json
{
  "type": "string",
  "title": "Passkey Assertion",
  "format": "webauthn-get",
  "x-webauthn-challenge": "<base64url-encoded challenge nonce>",
  "x-webauthn-timeout": 60000,
  "x-webauthn-rpId": "example.com",
  "x-webauthn-allowCredentials": [
    {
      "type": "public-key",
      "id": "<base64url-encoded credential ID>",
      "transports": ["internal"]
    }
  ],
  "x-webauthn-userVerification": "required",
  "x-webauthn-extensions": {}
}
~~~

## 8.2 Authorization Challenge Response

TODO

# 10. Security Considerations

TODO Security

--- back
