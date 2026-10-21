---
title: "Transaction Token - Fine-Grained OAuth Authorization in the AI Agent Era"
date: 2026-10-21 15:00:00
tags: OAuth, Transaction Token, AI Agent, security
keywords: OAuth 2.0, transaction token, txn-token, Zero Trust, RFC 8693, agentic AI
---


## Background

In this article, I will continue the thread from my earlier post on the OAuth 2.0 **Token Exchange** authorization flow and dive into a more challenging piece of the puzzle: **Transaction Token**. We will look at why it exists, what problems it solves, and why those problems matter even more in the AI Agent era.

This post mixes big-picture security context with protocol details, so buckle up—it is a bit of a ride. After reading it, you should understand how txn-tokens fit into Zero Trust and multi-hop service calls.

<img src="/images/transaction-token-fine-grained-oauth-authorization-in-the-ai-agent-era/Transaction_Tokens_for_AI_Agents.png" alt="" >

## The OAuth 2.0 Transaction Token Proposal

**Transaction Token** (txn-token for short) is still an **Internet Draft**. It comes from the **OAuth working group**, not a single author, and is already at **draft version 11**, which tells you how actively it is being reviewed and revised. It is not an RFC yet, but it has drawn a lot of attention in the community.

So what is the proposal trying to say? Let’s walk through it step by step.

## Why Do We Need Transaction Tokens?

Modern systems are usually made of many services and apps. When a request enters your network from the Internet, it often fans out across several internal services. At that point, you have a few common ways to validate tokens.

##### Castle and moat

In this pattern, you validate the token at the network edge—typically the API gateway. Once that check passes, internal services call each other **without** repeating authentication or authorization.

<img src="/images/transaction-token-fine-grained-oauth-authorization-in-the-ai-agent-era/Pasted%20image%2020261021105117.png" alt="" >

That is the classic **castle-and-moat** model: the corporate network is the castle, and the perimeter is the moat. Outsiders cannot get in; but once you cross the boundary—say, after login or VPN access—users and devices are often treated as fully trusted and can move freely inside the castle.

Cloud and SaaS broke that model. The replacement many teams adopt is **Zero Trust**: **never trust, always verify**. Whether the caller is inside or outside the corporate network, every access request should be authenticated and authorized.

That is why teams moved toward the approach below.

##### Token Exchange

<img src="/images/transaction-token-fine-grained-oauth-authorization-in-the-ai-agent-era/Pasted%20image%2020261021110633.png" alt="" >

Service-to-service traffic still needs identity and permissions. A common shortcut is to **reuse the user’s access token**. But OAuth 2.0 was designed around an interactive flow while the user is present, granting the client a relatively static set of scopes up front. Passing the user token deep into the call graph greatly increases the risk of **over-privileged leakage**.

As I mentioned in the previous article, you can use the **Token Exchange** grant (RFC 8693): exchange the user token for one with a **narrower, more appropriate scope**, then pass that token to downstream services.

<img src="/images/transaction-token-fine-grained-oauth-authorization-in-the-ai-agent-era/Pasted%20image%2020261021111553.png" alt="" >

Even Token Exchange, however, leaves a meaningful **attack surface**:

- **Spurious invocation**: a compromised internal service calls downstream APIs when there is no real user-driven request.
- **Identity impersonation**: the caller crafts the request and pretends to be another user.
- **Parameter tampering**: a compromised middle tier changes fields in the JSON body (for example, order quantity).
- **Token theft and replay**: an attacker steals a bearer token from logs or memory and replays it until it expires.

**Transaction Token** is meant to remove—or at least shrink—that attack surface.

## How Transaction Tokens Work

<img src="/images/transaction-token-fine-grained-oauth-authorization-in-the-ai-agent-era/Pasted%20image%2020261021113406.png" alt="" >

As the name suggests, a txn-token scopes authorization to a single **transaction**. “Transaction” here is **not** only a database transaction or a payment. It is a **logical unit of work** along one end-to-end business path. Because authorization is bound to that unit, txn-tokens are **short-lived**—often just a few minutes. You shrink the attack surface in **time** and in **scope**.

Next, let’s unpack the flow using a stock-trading example.

##### External call

The client sends a normal user access token to the API gateway:

```plain
POST /trade HTTP/1.1
Authorization: Bearer <oauth-access-token>
Content-Type: application/json

{ "action": "BUY", "ticker": "MSFT", "quantity": "100" }
```

##### Exchanging for a transaction token

The API gateway calls a **Transaction Token Server (TTS)**. You can use the same interfaces and conventions as **Token Exchange**, because txn-token issuance is fundamentally another form of token exchange. In OAuth terms, the **Authorization Server** can act as the TTS.

```plain
POST /txn-token-service/token_endpoint HTTP/1.1
Host: txn-token-service.trust-domain.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:txn_token
&audience=trust-domain.example
&scope=trade.stocks
&subject_token=eyJhbGciOiJFUzI1NiIsImtpZC...
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&request_context={"req_ip":"69.151.72.123","authn":"urn:ietf:rfc:6749"}
&request_details={"action":"BUY","ticker":"MSFT","quantity":"100"}
```

The exchange carries **context** and **details** via `request_context` and `request_details`. Notice that `audience` is no longer a single downstream API—it represents the **trust boundary** for this transaction.

##### What is inside the transaction token?

The txn-token is a signed JWT. The TTS derives claims from the **subject token** (the user access token) and from the request, including `request_details` and `request_context`, and adds further attributes as policy allows.

A typical txn-token **header** looks like this:

```json
{
  "typ": "txntoken+jwt",
  "alg": "RS256",
  "kid": "tts-2026-q2"
}
```

The type is `txntoken+jwt`. `kid` identifies the public key downstream services use to verify the signature.

The **payload** might look like this:

```json
{
  "iat": 1686536226,
  "aud": "trust-domain.example",
  "exp": 1686536586,
  "txn": "97053963-771d-49cc-a4e3-20aad399c312",
  "sub": "d084sdrt234fsaw34tr23t",
  "req_wl": "apigateway.trust-domain.example",
  "scope": "trade.stocks",
  "rctx": {
    "req_ip": "69.151.72.123",
    "authn": "urn:ietf:rfc:6749"
  },
  "tctx": {
    "action": "BUY",
    "ticker": "MSFT",
    "quantity": "100",
    "customer_type": { "geo": "US", "level": "VIP" }
  }
}
```

Key fields:

- **`sub`**: user identifier, taken from the subject token.
- **`aud`**: the trust boundary; the token is only meaningful inside that boundary.
- **`exp`**: short lifetime—on the order of minutes.
- **`txn`**: unique id for this transaction.
- **`req_wl`**: who initiated the exchange (often the gateway).
- **`scope`**: same idea as in ordinary access tokens.
- **`rctx`**: request context (for example, client IP).
- **`tctx`**: **transaction context**. Important: the TTS should **not** blindly copy the request body into `tctx`; that would be easy to abuse. The TTS decides what may be bound into `tctx`.

The TTS signs the JWT with its private key and returns it as `access_token` in the Token Exchange response—exactly as RFC 8693 defines.

```plain
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "token_type": "N_A",
  "issued_token_type": "urn:ietf:params:oauth:token-type:txn_token",
  "access_token": "eyJ0eXAiOiJ0eG50b2tlbitqd3QiLCJhbGciOiJSUzI1NiIsImtpZCI6InR0cy0yMDI2LXEyIn0..."
}
```

##### Propagation and verification

Downstream calls should send the txn-token in the **`Txn-Token`** HTTP header, because **`Authorization`** is reserved for ordinary access tokens:

```plain
POST /orders HTTP/1.1
Host: order-service.trust-domain.example
Txn-Token: eyJ0eXAiOiJ0eG50b2tlbitqd3QiLC...
Content-Type: application/json

{"action": "BUY", "ticker": "MSFT", "quantity": 100}
```

The downstream service typically verifies:

- **Signature**: validate with the TTS public key so the payload cannot be altered.
- **`aud`**: matches this deployment’s trust boundary (usually configuration).
- **`exp`**: token is still valid; lifetimes are short by design.
- **`tctx`**: this is the txn-token superpower. A compromised service might change the body from “buy 100 shares” to “buy 1000 shares,” but **`tctx` is inside the signed JWT**—downstream code should trust `tctx`, not the mutable body. Smart design, right?

## What Transaction Tokens Buy You

Now that we have walked through the flow, let’s map txn-tokens back to the four attack patterns:

- **Spurious invocation**: only the TTS can mint a valid txn-token. A compromised service cannot forge one; downstream verification fails fast.
- **User impersonation**: `sub` is part of the signed JWT and cannot be rewritten in transit.
- **Parameter tampering**: business parameters live in **`tctx`**. Verifiers read authoritative data from the token, not from attacker-controlled JSON.
- **Token theft and replay**: txn-tokens are tied to a **single audience** (one trust boundary), carry **minimal scope**, expire quickly, and include a unique **`txn`** id so you can enforce **one-time use** with a cache. Even if theft happens, the window is tiny.

## Agentic AI

Rolling out transaction tokens is real engineering work—it can feel heavy for classic three-tier apps. For **agentic AI** systems, though, where tool calls, delegation, and multi-hop workflows are the norm and behavior is harder to predict, tightening the authorization attack surface is often worth the investment.
