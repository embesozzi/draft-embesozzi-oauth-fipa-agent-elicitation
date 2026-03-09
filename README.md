# OAuth 2.0 Native Authorization via MCP Elicitation

**Document:** `draft-embesozzi-oauth-fipa-mcp-elicitation`  
**Status:** Individual Submission — Informational  
**Author:** Martin Besozzi (TwoGenIdentity)  
**Date:** 2026-03-09  

## Abstract

The OAuth 2.0 First-Party Applications (FiPA) specification defines a
challenge/response protocol enabling API-native authentication without
browser redirects. FiPA intentionally leaves authenticator metadata out
of scope. This gap prevents interoperable implementation by AI Agents,
CLI tools, and autonomous agents.

This document proposes an extension that adopts MCP Elicitation as the
standard metadata language for FiPA authenticator challenges, covering
TOTP and Passkey (WebAuthn) authenticator types across Third-Party and
First-Party AI Agent deployments.

## Document Structure

| Section | Topic |
|---|---|
| §1 Introduction | Applicability, Limitations, Deployment Types |
| §2 Conventions and Definitions | RFC 2119 keywords, Terminology |
| §3 Protocol Overview | FiPA challenge/response + MCP Elicitation modes |
| §4 Elicitation Extension | `elicitations` array, sequencing, wire format, non-MCP fallback |
| §5 Authenticator Selection | Method selection elicitation |
| §6 TOTP Challenge | Form mode TOTP flow |
| §7 Passkey — Third-Party Agent | URL mode (out-of-band), limitations |
| §8 Passkey — First-Party Agent | In-band WebAuthn via `format: webauthn-get` |
| §10 Security Considerations | OTP exposure, challenge binding, phishing |
| §11 References | Normative and Informative |

## Related Work

- [OAuth 2.0 First-Party Applications (FiPA)](https://datatracker.ietf.org/doc/draft-ietf-oauth-first-party-apps/)
- [MCP Elicitation Specification](https://spec.modelcontextprotocol.io)
- [RFC 6749 — OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749)
- [W3C Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn/)

## Contributing

Feedback is welcome via GitHub Issues or Pull Requests.

## Author

Martin Besozzi — [TwoGenIdentity](https://twogenidentity.com) —
embesozzi@twogenidentity.com

---

*This document is an independent submission Internet-Draft for community discussion. It is not an IETF or OpenID Foundation standards-track document.*

