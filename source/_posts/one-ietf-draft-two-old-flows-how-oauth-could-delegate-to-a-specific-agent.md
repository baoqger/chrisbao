---
title: "One IETF Draft, Two Old Flows - How OAuth Could Delegate to a Specific Agent"
date: 2026-10-10 13:00:00
tags: OAuth, AI Agent, IETF, security
keywords: OAuth 2.0, AI agent authorization, Token Exchange, Authorization Code Flow, on-behalf-of delegation
---

<img src="/images/one-ietf-draft-two-old-flows-how-oauth-could-delegate-to-a-specific-agent/Why_AI_Agents_Need_OAuth%201.png" alt="" >
## Why AI Agents Need a New OAuth Extension

OAuth 2.0 was designed in an era where the main security question was straightforward: how can a user authorize an application to access a protected resource? That model still works well for traditional web apps and APIs, but AI agents introduce a new kind of actor into the system. An AI agent is not just a browser, not just a backend service, and not merely a normal client application. It is a decision-making component that can interpret user goals, choose actions, and call tools or APIs with a degree of autonomy. That shift matters for authorization.

To understand why a new OAuth extension is needed for AI agents, it helps to start with the two existing patterns: the standard Authorization Code Flow and OAuth Token Exchange. Both solve important problems. But neither fully captures what it means for a specific AI agent to act on behalf of a user.

### 1. What the Standard Authorization Code Flow Solves

The standard Authorization Code Flow for a server-side web app is built around a simple trust model.

A user is redirected to the authorization server, signs in, and grants consent. The authorization server then returns an authorization code to the application. The backend redeems that code using its `client_id` and `client_secret`, and receives tokens. The basic meaning of this flow is clear:

- the user approved access
- the application proved its identity as the registered client
- the authorization server issued tokens based on that combination

This is a strong design for classic web applications because it binds user approval to a trusted confidential client. It also protects the token exchange step by requiring the backend to authenticate itself with a `client_secret` or similar credential.

However, this model only really understands two important parties:

- the `user`
- the `client application`

That was enough when the client application was assumed to be the only meaningful actor. But in an AI system, that assumption starts to break down.

### 2. Why the Standard Authorization Code Flow Is Not Enough for AI Agents

<img src="/images/one-ietf-draft-two-old-flows-how-oauth-could-delegate-to-a-specific-agent/Pasted%20image%2020260609211837.png" alt="" >

Inside an AI application, there may be multiple internal agents with very different behaviors and risk profiles. One agent may summarize emails. Another may send messages. Another may create financial reports or call external systems. From a security point of view, those are not all the same thing.

But standard Authorization Code Flow has no native way to express that distinction.

When the user consents in the classic flow, the consent screen can say:

- this application wants access
- these scopes are requested

What it cannot clearly say is:

- this specific agent inside the app wants to act for you
- this agent may read your calendar but not send email
- this agent is the one that will perform the next step autonomously

In other words, the standard flow gives consent to the client, not to the internal actor. That creates a visibility gap. The authorization server knows which app received consent, but it does not know which agent inside that app is actually going to act. The resource server later sees a token, but it may only know the user and the client, not the specific agent making decisions.

That gap becomes important because AI agents are not passive code paths. They can choose actions dynamically. They can chain tools together. They can operate across multiple hops. If the protocol cannot identify the acting agent, then user delegation becomes too coarse. The user is effectively authorizing the whole application, even if only one specific agent should be trusted for a given task.

Fixing that gap might require richer delegation in the token, not just a better consent screen — which is why Token Exchange is often discussed alongside agents. 
### 3. What OAuth Token Exchange Solves

<img src="/images/one-ietf-draft-two-old-flows-how-oauth-could-delegate-to-a-specific-agent/Token_Exchange_Security_Overview.png" alt="" >

OAuth Token Exchange, described in RFC 8693, addresses a similar problem in micro service architectures.

It starts from the idea that one token is often the wrong token for the next hop. A frontend service may receive a user token, but a downstream API may need a different token with a different audience, narrower scope, or explicit delegation context. Instead of blindly forwarding the original token, the service can exchange it for a new one better suited for the downstream service.

This is valuable for several reasons:

- it reduces scope
- it enforces correct audience boundaries
- it improves trust separation between services
- it can preserve delegation semantics using actor-related claims such as `act`

This is already much closer to the AI-agent problem than classic Authorization Code Flow is. Token Exchange understands that the entity presenting a token and the subject represented by that token may not be the same party. It gives us a more precise model for multi-hop systems.

That is why Token Exchange feels like an important building block for agent security.

### 4. Why Token Exchange Still Is Not Enough for AI Agents

Even though Token Exchange is closer to what AI agents need, it still does not fully solve the problem.

The biggest reason is that Token Exchange is mainly a back-channel mechanism. It assumes the calling service already has some token or authority and now wants to transform it into a different token for another context. That works well in service-to-service architectures. But AI-agent delegation often needs something more: explicit user consent in the front channel.

An AI agent may decide that it now needs permission to access a new resource or perform a higher-risk action. At that moment, the system should not simply assume that the client can translate existing authority into agent authority. The user may need to be asked directly:

- do you allow this specific agent to act for you?
- do you allow it for these scopes?
- do you allow it in this context?

Plain Token Exchange does not define that consent step. It helps you transform authority, but it does not define how the user approves a particular internal agent in the first place.

There is also another limitation: Token Exchange can model delegation, but it does not make the AI agent a first-class identity in the end-to-end flow. In many AI systems, the real security question is not just "does the application have permission?" but "which exact agent is exercising that permission right now?" If that identity is not explicitly captured and verified, then the protocol cannot strongly bind user approval to the acting agent.

### 5. The Core Problem: AI Agents Create a New Security Boundary

Traditional OAuth assumes the client application is the relevant unit of trust. In an AI system, that is no longer always true.

The client is still important, but now there is a second boundary inside the client:

- the `client` is the application boundary
- the `agent` is the decision-making boundary

That distinction matters because the agent is often the component that interprets intent and chooses actions. It is the one deciding whether to query a calendar, send a message, call a database, or invoke another tool. If the security system only sees the outer client, it misses the actual actor driving the behavior.

This creates several risks:

- the user may consent to an application without understanding which agent will act
- one agent may misuse permissions intended for another agent
- the authorization server may be unable to verify that the same agent approved in the consent step is the one redeeming the delegation
- the resource server may be unable to audit which specific agent performed the action

In short, AI agents introduce a new layer of autonomy, and that autonomy needs to become visible to the authorization system.

### 6. What the New OAuth Extension Adds

Although OAuth has not yet published a formal RFC for AI-agent authorization, the OAuth working group and individual developers have already contributed a number of approaches. This article walks through one such proposal as an example; other drafts and designs are worth exploring separately.

That proposal is the draft OAuth 2.0 Extension: On-Behalf-Of User Authorization for AI Agents. https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html

<img src="/images/one-ietf-draft-two-old-flows-how-oauth-could-delegate-to-a-specific-agent/Delegated_AI_for_Agent_Security.png" alt="" >

The new OAuth extension for AI agents addresses this by combining the strengths of the two older models.

From Authorization Code Flow, it keeps the front-channel consent model. The user is still redirected to the authorization server and can still approve access interactively.

From Token Exchange, it borrows the delegation mindset. The final token should not merely represent "the app acting for the user." It should represent "this specific agent acting for the user through this client."

To make that work, the extension adds new protocol elements such as:

- a `requested_actor` parameter in the authorization request
- an `actor_token` in the token request
- an `act` claim in the resulting access token

These additions matter because they create a full chain of authorization meaning:

1. The client tells the authorization server which agent is requesting delegation.
2. The user consents not only to the app and scope, but also to that specific agent.
3. The agent proves its identity later using an `actor_token`.
4. The authorization server verifies that the actor redeeming the flow is the same actor the user approved.
5. The resulting access token records both the user and the agent.

That closes the gap that exists in both older flows.

### 7. Why This Is Better Than Reusing the Old Flows As-Is

Without the new extension, a system would have to improvise.

If it used only Authorization Code Flow, it could authenticate the app and capture user consent, but it would not clearly identify the internal agent. The user's approval would be too broad.

If it used only Token Exchange, it could model delegation better, but it would miss the front-channel consent experience needed for user-approved agent behavior. The protocol would know how to transform authority, but not how to get the user's explicit approval for a specific agent.

The new extension is needed because AI-agent security requires both:

- explicit user consent
- explicit actor identity

Standard Authorization Code Flow gives you the first, but not the second. Token Exchange gives you pieces of the second, but not the first in the right form. The extension combines them into a model that fits agent systems.

### 8. The Bigger Security Meaning

At a deeper level, this is not just a protocol convenience. It is a shift in security thinking.

Older OAuth patterns mostly answer the question: can this application access this resource for this user?

Agent-based systems force us to ask a more precise question: which autonomous actor inside the application is acting for the user, and did the user explicitly approve that delegation?

That is a more demanding requirement, but it matches the reality of AI systems. AI agents are not just UI features. They are operational actors. They need explicit identity, explicit delegation, and explicit auditability.

### Conclusion

The new OAuth extension for AI agents is needed because AI changes the unit of trust.

Standard Authorization Code Flow assumes the client application is the meaningful actor. Token Exchange assumes delegated authority can be transformed across service boundaries. But AI agents sit between those two models: they need fresh user consent like an interactive client flow, and they need explicit delegation semantics like a multi-hop service flow.

That is why a new extension makes sense. It allows OAuth to move from a world where "an app gets a token for a user" to a world where "a specific agent is delegated limited authority to act for a user, and every party in the system can see and verify that fact."
