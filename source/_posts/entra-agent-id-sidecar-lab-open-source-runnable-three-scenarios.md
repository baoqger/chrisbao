---
title: "Entra Agent ID Sidecar Lab - Open Source, Runnable, Three Scenarios"
date: 2026-11-17 09:30:00
tags: Azure, Entra ID, AI Agent, Sidecar, OAuth
keywords: Entra Agent ID Sidecar Lab, Docker Compose, Jupyter, Autonomous App Flow, OBO, microsoft-identity-web
---

## Background

My earlier article on Entra Agent ID received encouraging feedback. This week I spent some time polishing that work into a project and released it on GitHub so you can clone it, run it, and see how things behave end to end.

As I mentioned in that previous post, Entra Agent ID can feel abstract until you wire up real objects, permissions, and tokens. After reading this article, you should know what the **Entra Agent ID Sidecar Lab** repo contains, how the labs are organized, and why it might save you setup time if you want to experiment on your own.

<img src="/images/entra-agent-id-sidecar-lab-open-source-runnable-three-scenarios/Agent_Sidecar_Architecture_Overview.png" alt="" >
## Project overview

The open-source project is called **Entra Agent ID Sidecar Lab**. Its main goal is to give you a **runnable, end-to-end baseline** you can learn from and extend—not just slides or architecture diagrams.

Repository: [https://github.com/baoqger/entra-agent-id-sidecar-lab](https://github.com/baoqger/entra-agent-id-sidecar-lab)

`Entra Agent ID` currently supports **three** authentication and authorization patterns:

- **Autonomous App Flow** — the agent runs on its own, with no user in the loop.
- **On Behalf Of (OBO) Flow** — delegated access on behalf of a user.
- **Agent User Flow** — autonomous execution, but acting as a dedicated simulated user.

The repo is structured around those three scenarios: **one Lab per flow**. Lab 1—the autonomous case—is complete today; I plan to add the other two labs next.

Each lab has two parts: `scripts` and `sidecar`.

<img src="/images/entra-agent-id-sidecar-lab-open-source-runnable-three-scenarios/Pasted%20image%2020261114154648.png" alt="" >

### `scripts`

The `scripts` folder holds **Python** automation. To make the material easier to follow, I used **Jupyter notebooks** instead of the PowerShell scripts you often see in official samples. These notebooks create and configure Entra objects and permissions—the blueprint, identities, grants, and related setup you need **before** the sidecar stack can run. Please run them first.

<img src="/images/entra-agent-id-sidecar-lab-open-source-runnable-three-scenarios/Pasted%20image%2020261114155809.png" alt="" >

### `sidecar`

Under `sidecar`, **Docker Compose** runs a **four-container** setup that talks to each other on a shared network.

<img src="/images/entra-agent-id-sidecar-lab-open-source-runnable-three-scenarios/Pasted%20image%2020261114155017.png" alt="" >

- **`llm-agent`** — a small but real agent built with LangChain
- **`ollama`** — a local model runtime that backs the agent
- **`weather-api`** — a downstream API used as a tool in the agent
- **`sidecar`** — Microsoft’s open-source auth sidecar image (`mcr.microsoft.com/entra-sdk/auth-sidecar`); you don’t need to build this part yourself

If you’ve already read [[Entra Agent ID + Sidecar - A Practical Guide to Building Secure AI Agents]], this layout should look familiar—the lab turns that pattern into something you can start with one command after the scripts finish.

## Why this repo is useful

<img src="/images/entra-agent-id-sidecar-lab-open-source-runnable-three-scenarios/Pasted%20image%2020261114160357.png" alt="" >

I hope the project helps in a few concrete ways:

- **Entra Agent ID** — the topic is still young, and there are few repos you can run immediately. Rolling your own tenant setup from scratch takes time; this repo is a **complete starting point**.
- **OAuth 2.0 on Entra ID** — Entra ID is a well-known identity provider (IdP) for users and applications. The lab exercises real registration, consent, and token flows. You can line up the notebooks and containers with the spec and dig into authorization details hands-on.
- **AI agent security** — you can also use the stack as a baseline for offensive and defensive experiments around agent identity and privilege. That space is wide open—I’m sure others will find angles I haven’t thought of yet, and I’d love to hear about them.

Next step: pick Lab 1, run the notebooks, then bring up Compose. Great way to connect theory to a running agent, right?