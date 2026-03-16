# OAuth 2.0 First-Party Native Authorization for AI Agents via Structured Elicitation

**Document:** `draft-embesozzi-oauth-fipa-agent-elicitation`
**Status:** Individual Submission — Informational
**Author:** Martin Besozzi (TwoGenIdentity)
**Date:** 2026-03-09

## Abstract

This document defines a Structured Elicitation extension to the OAuth 2.0
First-Party Applications (FiPA) specification, establishing a standard
metadata format for FiPA authorization challenge responses. FiPA intentionally
leaves authenticator metadata out of scope: the format in which the
authorization server describes available authenticators and the inputs they
require is undefined. This gap prevents interoperable implementation by AI
Agents, CLI tools, and other non-browser clients.

Model Context Protocol (MCP) Elicitation serves as the normative reference
implementation of the Structured Elicitation mechanism. The extension
addresses Human-to-Agent (H2A) communication patterns in which a human user
interacts with an AI Agent acting as the FiPA client on their behalf. The
scope covers two strong authenticator types: Authenticator Apps (TOTP) and
Passkeys (WebAuthn). Password authentication is explicitly out of scope. The
same pattern is extensible to other authenticator types by defining additional
`requestedSchema` structures.

The extension is specified across two deployment types: Third-Party AI Agents
(e.g., Claude, GitHub Copilot), where the client runtime is provided by a
third party and cannot be modified; and First-Party AI Agents, where the
implementer controls the agent code. Both deployment types share the same
FiPA challenge/response wire format and Structured Elicitation protocol,
differing only in Passkey handling.

## Documents

| Document | Link |
|----------|------|
| **Draft** | [draft-oauth-fipa-agent-elicitation.html](https://embesozzi.github.io/draft-embesozzi-oauth-fipa-mcp-elicitation/draft-oauth-fipa-agent-elicitation.html) |

## Versions

| Date | Link |
|------|------|
| **Latest** | [Editor's Copy](https://embesozzi.github.io/draft-embesozzi-oauth-fipa-mcp-elicitation/draft-oauth-fipa-agent-elicitation.html) |

## Contributing

Feedback is welcome via GitHub Issues or Pull Requests.

## Author

Martin Besozzi — [TwoGenIdentity](https://twogenidentity.com) —
embesozzi@twogenidentity.com

---

*This document is an independent submission Internet-Draft for community discussion. It is not an IETF or OpenID Foundation standards-track document.*

