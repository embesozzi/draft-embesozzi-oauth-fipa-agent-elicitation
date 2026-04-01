# OAuth 2.0 Agents Native Authorization via Structured Elicitation

**Document:** `draft-embesozzi-oauth-agent-native-authorization`  
**Status:** Individual Submission — Informational  
**Author:** Martin Besozzi (TwoGenIdentity)  
**Date:** 2026-04-01

## Abstract

This document defines a Structured Elicitation extension to the OAuth 2.0
First-Party Applications (FiPA) specification, establishing a standard
metadata format for FiPA authorization challenge responses. FiPA leaves the
format for advertising available authenticators and their required inputs
undefined, preventing interoperable implementation by AI Agents and other
non-browser clients. This extension adds an `elicitations` array to the FiPA
Authorization Challenge Response, providing a standard metadata format for
authenticator challenges. Model Context Protocol (MCP) Elicitation is the
normative reference binding.

AI Agents executing on behalf of users may encounter operations that require
step-up authentication or explicit user approval — a just-in-time (JIT)
authorization gate. Upon successful completion of the FiPA challenge/response
cycle, the authorization server issues a token that serves as cryptographic
proof of the user's consent, bound to the specific auth session.

This specification covers the just-in-time (JIT) authorization for AI Agents
executing on behalf of users in Human-to-Agent (H2A) flows.

## Documents

| Document | Link |
|----------|------|
| **Draft** | [draft-embesozzi-oauth-agent-native-authorization.html](https://embesozzi.github.io/draft-embesozzi-oauth-agent-native-authorization/draft-embesozzi-oauth-agent-native-authorization.html) |

## Versions

| Date | Link |
|------|------|
| **Latest** | [Editor's Copy](https://embesozzi.github.io/draft-embesozzi-oauth-agent-native-authorization/draft-oauth-agent-native-authorization.html) |

## Contributing

Feedback is welcome via GitHub Issues or Pull Requests.

## Author

Martin Besozzi — [TwoGenIdentity](https://twogenidentity.com) —
embesozzi@twogenidentity.com

