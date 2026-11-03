---
title: "Lightweight Fine-Grained Authorization for AI Agents - OAuth Rich Authorization Requests"
date: 2026-11-03 09:31:00
tags: OAuth, AI Agent, RAR, authorization
keywords: Rich Authorization Requests, RFC 9396, authorization_details, fine-grained authorization, scope
---


## Background

In a previous article, I introduced **Transaction Token** and walked through how it supports fine-grained authorization. If you read that post, you probably felt the same thing I did: the approach is powerful, but it comes with real engineering cost.

In this article, I will share a lighter-weight OAuth extension—**Rich Authorization Requests (RFC 9396)**—that can also express **fine-grained** permissions. For **AI Agents**, that capability is especially valuable. After reading this post, you should understand why classic `scope` strings fall short, and how RAR adds structured authorization details without jumping straight to a full Transaction Token rollout.

<img src="/images/lightweight-fine-grained-authorization-for-ai-agents-oauth-rich-authorization-requests/Fine-Grained_OAuth_for_AI.png" alt="" >
## The `scope` Parameter

In the OAuth 2.0 authorization flow, the user delegates access to an application within a bounded set of permissions. That boundary is carried in the authorization request as the **`scope`** parameter, for example:

```plain
GET /authorize?
   response_type=code
   &client_id=s6BhdRkqt3
   &state=af0ifjsldkj
   &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
   &scope=openid%20profile%20email
```

Because this request is usually sent with **`GET`**, `scope` has to be a single string. That limits how much meaning and semantics you can pack into it.

Community practice often looks like this:

```plain
users:read users:write reports:read
```

The pattern is **`resource:action`**. When you need permissions on multiple resources, you separate them with spaces. That works well for requests like “let me read user profile data,” but it struggles with richer intent—for example, “let me transfer **45** to **Merchant A**.”

For AI Agents, **fine-grained authorization** is not a nice-to-have. Agents make autonomous decisions. If the consent screen cannot show those details, the user is effectively approving a blind spot—and that is dangerous. This gap between classic OAuth authorization and what agents need is one of the most common pain points teams hit when they wire agents into real APIs.

As we mentioned above, the **Transaction Token** path can record authorization scope inside the transaction context field (**`tctx`**). Smart design—but as I said at the start, the implementation cost is higher. In this post, let’s look at a **lighter** approach that is already on the standards track.

## Rich Authorization Requests

**Rich Authorization Requests (RAR)** is an OAuth 2.0 extension standardized as **RFC 9396**. The idea is simple: add a new parameter, **`authorization_details`**, that describes authorization scope in a structured way.

To cover multiple resources or use cases, `authorization_details` is a **JSON array**. Each object describes one authorization scenario—for example:

```json
[
   {
      "type": "account_information",
      "actions": [
         "list_accounts",
         "read_balances",
         "read_transactions"
      ],
      "locations": [
         "https://example.com/accounts"
      ]
   },
   {
      "type": "payment_initiation",
      "actions": [
         "initiate",
         "status",
         "cancel"
      ],
      "locations": [
         "https://example.com/payments"
      ],
      "instructedAmount": {
         "currency": "EUR",
         "amount": "123.50"
      },
      "creditorName": "Merchant A",
      "creditorAccount": {
         "iban": "DE02100100109307118603"
      },
      "remittanceInformationUnstructured": "Ref Number Merchant"
   }
]
```

The **`type`** field identifies the resource or scenario. Your authorization server must register and support each type you use. Other fields depend on your business requirements.

When you send the authorization request, you URL-encode that JSON and attach it to the query string:

```plain
GET /authorize?
   response_type=code
   &client_id=s6BhdRkqt3
   &state=af0ifjsldkj
   &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
   &authorization_details=%5B%7B%22type%22%3A%22account%5Finfo
```

From that angle, RAR is a remarkably straightforward way to move from coarse scope strings toward **lightweight, fine-grained authorization**—especially when you are prototyping agent workflows or comparing options before committing to a heavier Transaction Token architecture.

## Summary

- Classic OAuth **`scope`** is a space-delimited string; it is easy to use but hard to express precise, transaction-level consent.
- **AI Agents** amplify that problem because users must understand *what the agent is about to do*, not just which API family it may call.
- **Rich Authorization Requests (RFC 9396)** add **`authorization_details`** so clients can describe structured permissions—including amounts, payees, and actions—in the authorization request itself.
- RAR does not replace every Transaction Token use case, but it is a **lighter** standards-based step toward fine-grained authorization. Great starting point, right?
