---
title: "OAuth 2.0 Token Exchange - Laying the Foundation for AI Agent Authorization"
date: 2026-10-10 12:00:00
tags: OAuth, Token Exchange, AI Agent, security
keywords: RFC 8693, OAuth 2.0, delegation, impersonation, AI agent authorization
---


## Background

In a previous article, I walked through the three most common OAuth 2.0 grant flows: `Authorization Code Flow`, `PKCE Flow`, and `Device Flow`. We looked at which scenarios and application types each one fits, what problems they solve, and how they differ in implementation.

In this post, I will add another important flow: `Token Exchange`. After reading it, you should have a clearer picture of how token exchange works—and why that understanding pays off when you start thinking about **AI Agent** authorization. Let's get started.

<img src="/images/oauth-2-0-token-exchange-laying-the-foundation-for-ai-agent-authorization/Securing_APIs_and_AI_Agents.png" alt="" >
## What Is Token Exchange?

The `Token Exchange (RFC 8693)` flow is straightforward at a high level: when a client already holds a token, it asks the authorization server (Authorization Server) to issue a **new** token that is better suited to another service or another security context. You exchange an old token for a new one—hence **Token Exchange**.

## Why Do We Need Token Exchange?

Modern systems are usually made of many services and applications. `Token Exchange` is especially valuable for service-to-service communication and cross-boundary scenarios. Suppose an upstream API receives the user's token and then needs to call a downstream API. Instead of forwarding the original user token everywhere, the upstream API can exchange it for a **new** token that is meant only for the downstream API. That pattern helps in several ways:

- **Matching audience**: A user token is often issued for one specific API. The receiver should align with the token's `aud` (audience). Otherwise you are accepting a token that was not minted for you—a classic **trust boundary** problem.
- **Appropriate scope**: The original user token may carry permissions intended for the first API. If every downstream API keeps using that same token, they often end up with more authority than they need—the **least privilege** problem.
- **Explicit actor**: User tokens usually do not carry delegation metadata. Downstream services often need to know who delegated access (delegator) and who is actually performing the action (actor).
- **Controlled token forwarding**: If every hop only forwards the original user token—for example, browser → API gateway → service A → service B → service C—then any compromise of one service exposes a token that still works elsewhere. That is the **blast radius** problem.

So the concerns `Token Exchange` addresses line up closely with how we think about network security:

- **Least privilege**: Every component gets only the access it needs.
- **Trust boundary**: A token issued for Service A should not be automatically trusted by Service B.
- **Context**: Do not only ask “Is this token valid?”—also ask “Valid **for whom**? **For what**?”
- **Defense in depth**: Even if one token leaks, limit where it can be used.
- **Blast radius reduction**: Keep token compromise as local as possible instead of spreading across the whole system.
- **Auditability and attribution**: Know who the user is and which service is acting.
- **Explicit delegation**: Make delegation visible; do not let one party silently inherit another's rights.

Unlike other OAuth flows, `Token Exchange` centers on a concept that matters a lot for AI agents: **delegation**. That is why I see it as groundwork for agent authorization.

## Impersonation vs Delegation

The `Token Exchange` specification spends considerable effort distinguishing **impersonation** from **delegation**. Let's walk through a concrete example.

Assume this setup:

- User: `Alice`
- Frontend service: `orders-web`
- Backend service: `inventory-api`
- Authorization server: `auth.example.com`

Alice uses the frontend. `orders-web` holds Alice's token but needs a **new** token scoped for `inventory-api`. `Token Exchange` covers two patterns: **impersonation** and **delegation**.

#### Impersonation

In the **impersonation** case, `orders-web` requests a token that lets it **act as Alice** when calling `inventory-api`:

```plain
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXJzLXdlYjpzZWNyZXQ=

grant_type=urn:ietf:params:oauth:grant-type:token-exchange&
subject_token=user-access-token&
subject_token_type=urn:ietf:params:oauth:token-type:access_token&
audience=inventory-api
```

`subject_token` is the subject's token—in our story, Alice's original user token. That identity shows up in the new token's `sub` claim. `audience` is the intended recipient API, here `inventory-api`.

Key claims in the new token:

```json
{
  "sub": "alice",
  "aud": "inventory-api",
  "scope": "inventory.read"
}
```

From `inventory-api`'s perspective, the token is **about Alice**. It has no signal that `orders-web` is the caller. That is why this pattern is called **impersonation**.

#### Delegation

In the **delegation** case, `orders-web` sends a slightly richer request:

```plain
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXJzLXdlYjpzZWNyZXQ=

grant_type=urn:ietf:params:oauth:grant-type:token-exchange&
subject_token=user-access-token&
subject_token_type=urn:ietf:params:oauth:token-type:access_token&
actor_token=orders-web-service-token&
actor_token_type=urn:ietf:params:oauth:token-type:access_token&
audience=inventory-api
```

Besides `subject_token`, the client also supplies `actor_token`—the token for the acting party, `orders-web`. That relationship appears in the `act` claim:

```json
{
  "sub": "alice",
  "aud": "inventory-api",
  "scope": "inventory.read",
  "act": {
    "sub": "orders-web"
  }
}
```

Now `inventory-api` can see: the request concerns Alice, and **orders-web** is the actor performing the call on her behalf. That is **delegation**.

Great—once delegation is clear, the next question is natural: what does this mean for **AI Agent** authorization?

## AI Agent Scenarios

Compared with traditional apps, AI agents stand out because of **autonomy**: an agent can decide its own next steps. One client application may host **multiple** agents, each able to act on its own. For AI agents, an explicit delegation chain is not a nice-to-have—it is often required. That is a big reason many agent authorization designs lean on **Token Exchange**.

For example, **AWS AgentCore** and **Azure Foundry Agent** both support **On-Behalf-Of Token Exchange**-style authorization. I will dig into how those implementations work in upcoming posts—stay tuned!
