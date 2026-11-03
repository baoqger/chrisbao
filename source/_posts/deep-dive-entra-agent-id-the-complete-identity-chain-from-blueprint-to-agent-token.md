---
title: "Deep Dive Entra Agent ID - The Complete Identity Chain from Blueprint to Agent Token"
date: 2026-11-03 09:35:00
tags: Azure, Entra ID, AI Agent, identity
keywords: Entra Agent ID, Agent Identity Blueprint, Microsoft Graph, OAuth client credentials, Foundry
---

## Background

For modern applications, authentication usually follows a pattern where a dedicated, centralized service handles authentication and authorization—that service is the **identity provider (IdP)**—instead of every app implementing those concerns on its own.

**Entra ID** is the IdP for the Azure cloud platform. As I mentioned in my previous articles, you can use it to add authentication and authorization to your Azure solutions. Feel free to revisit those posts if you need a refresher.

In the AI era, people often say agents may work alongside—or even replace—humans in the workplace. Your next “colleague” at the company might be an agent. To support that trend, **Entra ID** extends its existing architecture with **Entra Agent ID**, acting as the IdP for AI agents to manage their identities, authorization, and more. In this article, I will walk through what I learned about the following questions:

- Why is managing **Agent Identity** so important?
- How does **Entra Agent ID** differ from classic **Entra ID** app identity?
- How do you create Entra Agent ID objects and use them for authentication and authorization?

If these topics interest you, bookmark this post and let’s get started. After reading it, you should understand the blueprint-to-agent token flow and where each object fits in Entra.

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/3-Layer_AI_Agent_Identity.png" alt="" >
## Identity Is the Primary Security Perimeter

Before cloud and mobile apps, the **network** was the security boundary—the corporate intranet was treated as trusted. Today, you cannot reliably know who or what is on your network. What can you use as a boundary instead? The answer is **identity as the security perimeter**—at least the primary one. That idea existed long before AI agents showed up.

An **identity** is the set of attributes that define or characterize a person, a device, or a software component. Common categories include:

- **Human identities**: ordinary user accounts.
- **Machine identities**: further split into **device identities** (hardware) and **workload identities** (apps or services).
- **Agent identities**: AI-driven agents—the focus of today’s discussion.

When any identity—user, device, or application—requests access to a resource, that identity becomes the center of verification. The security question is no longer “Is this request coming from inside our corporate network?” but “Who or what is making this request, and should we trust it?” That is why **identity is the security perimeter**: every access decision starts with identity—verify it, then control what that identity is allowed to do.

## From Entra ID to Entra Agent ID

An AI agent is still an application at its core, but autonomy and decision-making change the security model compared to traditional apps. That is why **Agent Identity** is modeled and managed as its own category.

Next, let’s compare traditional applications and AI agents from Entra’s architecture and design perspective.

##### Entra ID’s two-layer model

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102111340.png" alt="" >

**Entra ID** manages application identity with a **two-layer** model: **app registration** and **service principal**:

- **App registration**: You define the app name and settings—scopes, app roles, redirect URLs after sign-in, and so on. If you know OAuth 2.0, this step is essentially registering the OAuth client. Entra ID supports OAuth 2.0 as the authorization server.
- **Service principal**: When a tenant adds the app, Entra creates an identity in that tenant that can take action. In Entra ID, this is the unit to which roles and permissions are assigned.

The app registration is not the runtime object. Registering an app creates a service principal in the tenant; roles and permissions are granted to the **service principal**. Think of the app registration as a **template** defined in a tenant; **instantiation** produces the service principal. Smart design, right? It is very much like the relationship between a **class** and an **object** in object-oriented programming.

An app registration can be published and managed in one Entra ID tenant while the app is consumed by many tenants—multi-tenant apps.

One more important point: in each tenant, the service principal can hold credentials (for example, a client secret) and use them to authenticate and obtain tokens.

###### Entra Agent ID’s three-layer model

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102113718.png" alt="" >

Building on that design, **Entra Agent ID** introduces a **three-layer** model:

- **Agent Identity Blueprint**: For classic apps, the template/instance relationship was implicit. For agents, the name **Blueprint** makes it explicit. Conceptually it matches **app registration**: name and baseline settings, plus credential configuration used to validate agent-related identity flows.
- **Agent Identity Blueprint Principal**: Also a **service principal**—one blueprint principal per tenant. Up to this layer, the model closely mirrors the two-layer app model. The blueprint principal is created automatically; you do not provision it manually.
- **Agent Identity**: This is the new piece. An agent identity is also a **service principal**, created by the blueprint principal. It represents a **single running instance** of an agent—the **work identity** for that agentic application instance.

This architecture keeps **one blueprint principal per tenant** (the service principal for the blueprint) while allowing **many agent identities** in that tenant (one service principal per agent instance).

Unlike the classic pattern where every app instance might hold secrets, **credentials are created and held only on the Agent Identity Blueprint**, so individual agents do not need direct access to secrets—reducing risk. Great, right?

A simple analogy: the **blueprint** is the recipe, the **blueprint principal** is the licensed kitchen allowed to “cook” agents in your tenant, and each **agent identity** is a dish made from that recipe.

## Foundry Agent Identity

You might not have noticed: every agent you develop and deploy on **Foundry** automatically gets an agent identity created in **Entra ID**.

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102133145.png" alt="" >

In the following section, let’s run a hands-on demo: how to create Entra Agent ID objects and use them for authentication and authorization.

## Demo

This demo uses PowerShell and the **Microsoft Graph** API. The steps are:

##### Step 1: Create the blueprint

```powershell
$body = @{
    "@odata.type"         = "Microsoft.Graph.AgentIdentityBlueprint"
    displayName           = "ChrisBao Blueprint"
    "sponsors@odata.bind" = @("https://graph.microsoft.com/v1.0/users/$myUserId")
    "owners@odata.bind"   = @("https://graph.microsoft.com/v1.0/users/$myUserId")
}

$blueprint = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/beta/applications/" `
    -Headers @{ "OData-Version" = "4.0" } `
    -Body ($body | ConvertTo-Json)
```

The key parameter is `Microsoft.Graph.AgentIdentityBlueprint`, which tells Graph you are creating an **Agent Identity Blueprint**.

##### Step 2: Create the blueprint principal

```powershell
$principalBody = @{ appId = $blueprint.appId }
$principal = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/beta/serviceprincipals/graph.agentIdentityBlueprintPrincipal" `
    -Headers @{ "OData-Version" = "4.0" } `
    -Body ($principalBody | ConvertTo-Json)
```

##### Step 3: Create credentials

```powershell
$blueprintApp = (Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/applications?`$filter=appId eq '$($blueprint.appId)'").value[0]
...
$secret = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/beta/applications/$($blueprintApp.id)/addPassword" `
    -Body ($secretBody | ConvertTo-Json)
```

As you can see, the secret is created **under the blueprint**, not on each agent instance.

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102134551.png" alt="" >

##### Step 4: Get a blueprint token

```powershell
$testTokenBody = @{
    client_id     = $blueprint.appId
    scope         = "https://graph.microsoft.com/.default"
    grant_type    = "client_credentials"
    client_secret = $secret.secretText
}

$testResponse = Invoke-RestMethod -Method POST `
    -Uri "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token" `
```

An agent instance has no secret of its own and cannot authenticate by itself. We first use the blueprint’s secret to obtain a token. There is no user sign-in here—we use the OAuth 2.0 **client credentials** flow. If you want to go deeper on that flow, check my earlier articles on the topic.

##### Step 5: Validate token claims

```powershell
$tokenPayload = Get-DecodedJwtToken -Token $blueprintToken | ConvertFrom-Json
if ($tokenPayload.roles -notcontains "AgentIdentity.CreateAsManager") {
    Write-Warning "  [WARN]  Token doesn't have AgentIdentity.CreateAsManager role. Waiting for permissions to propagate..."
    Start-Sleep -Seconds 10
}
```

The blueprint token must include the **`AgentIdentity.CreateAsManager`** role before you can create an agent identity.

##### Step 6: Create an agent identity

```powershell
$agentIdentityBody = @{
    displayName              = "My Agent"
    agentIdentityBlueprintId = $BlueprintAppId
    "sponsors@odata.bind"    = @("https://graph.microsoft.com/v1.0/users/$UserId")
}

$agentIdentity = Invoke-RestMethod -Method POST `
    -Uri "https://graph.microsoft.com/beta/serviceprincipals/Microsoft.Graph.AgentIdentity" `
    -Headers @{
        "Authorization" = "Bearer $blueprintToken"
        "OData-Version"   = "4.0"
        "Content-Type"    = "application/json"
    } `
    -Body ($agentIdentityBody | ConvertTo-Json)
```

With that token, we call Graph to create the **agent identity**.

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102141245.png" alt="" >

##### Step 7: Exchange for an agent token

```powershell
# Exchange Blueprint token for agent token (Agent identity token)
$t2Body = @{
    client_id             = $AgentIdentityAppId
    scope                 = "https://graph.microsoft.com/.default"
    grant_type            = "client_credentials"
    client_assertion_type = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
    client_assertion      = $blueprintToken
}

$t2Response = Invoke-RestMethod -Method POST `
    -Uri "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token" `
    -ContentType "application/x-www-form-urlencoded" `
    -Body $t2Body

$agentToken = $t2Response.access_token
Write-Host "  [OK] Got T2 token (Agent identity)" -ForegroundColor Green
```

The agent needs its **own** token for API calls, so we exchange the blueprint token for an agent identity token using **JWT bearer client assertion**—no agent-held secret required for this step.

##### Step 8: Grant permissions to the agent identity

Even with an agent token, the identity has no effective permissions until app roles are assigned. Next we grant **`User.ReadWrite.All`** so the agent can read and write user data via Graph (adjust permissions to least privilege in production).

```powershell
$permissionMap = @{
    "User.ReadWrite.All" = "741f803b-c850-494e-b5df-cde7c675a1ca"
}

$appRoleId = $permissionMap["User.ReadWrite.All"]

$permissionBody = @{
    principalId = $AgentIdentitySP
    resourceId  = $graphSP
    appRoleId   = $appRoleId
}
Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/$AgentIdentitySP/appRoleAssignments" `
    -Body ($permissionBody | ConvertTo-Json) | Out-Null
Write-Host "  [OK] Added permission: $permission" -ForegroundColor Green
```

<img src="/images/deep-dive-entra-agent-id-the-complete-identity-chain-from-blueprint-to-agent-token/Pasted%20image%2020261102141323.png" alt="" >

##### Step 9: Verify the result

```powershell
$response = Invoke-RestMethod -Method GET `
    -Uri "https://graph.microsoft.com/v1.0/users?`$top=5" `
    -Headers @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type"  = "application/json"
    }
```

The output shows the call succeeded.

```plain 
Display Name User Principal Name                                 ID
------------ -------------------                                 --
Chris Bao    baoqger_gmail.com#EXT#@baoqgergmail.onmicrosoft.com d672a57e-86cc-46e3-b811-8fd26e0b0e50
```

## Sidecar Architecture with Entra Agent Identity

Two common approaches to agent authentication both fall short:

- **Hard-coded secrets in agent code.** Every agent image holds a copy of your app’s `client_secret`. Any leak, log exposure, or committed `.env` can become a tenant-wide incident.
- **Delegated user tokens for everything.** The agent only acts when a human is present, and you lose per-agent auditability—every call can look like the same service principal.

As we saw in the demo, **Entra Agent Identity** handles authentication and authorization without embedding secrets in the agent and without requiring an interactive user for every action. So how do you use this in real applications?

The answer is the **sidecar** pattern: deploy a sidecar next to each agent that encapsulates Entra Agent Identity logic. The agent focuses on its job; the sidecar handles identity and tokens. I will unpack that architecture in a future article—stay tuned.

## Summary

- **Identity** is the primary security perimeter—especially for autonomous agents calling APIs outside a fixed network edge.
- **Entra ID** uses app registration + service principal; **Entra Agent ID** adds blueprint, blueprint principal, and per-instance **agent identity**.
- Credentials stay on the **blueprint**; agents get distinct identities and tokens (including token exchange), which improves auditability and blast-radius control.
- **Foundry** can provision agent identities automatically; the PowerShell demo shows the same objects via Graph when you build custom agent hosts.
