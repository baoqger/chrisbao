---
title: Project
date: 2021-06-21 15:43:07
---

[cPacketSniffer](https://github.com/baoqger/cPacketSniffer): is a Linux network packets sniffer written in C language based on `libpcap`:

- Analyze network packets at low layers of TCP/IP stack, including Ethernet, ARP, ICMP, IP(IPv4), TCP, UDP, etc. 
- Detect network security attacks: ARP spoofing detection, Ping flood detection
- Analyze and track network traffics: TCP session tracking and traffic analysis, TFTP session tracking and traffic analysis

in detail, please check [here](https://github.com/baoqger/cPacketSniffer).

[Entra Agent ID Sidecar Lab](https://github.com/baoqger/entra-agent-id-sidecar-lab): hands-on labs for securing AI agents with Microsoft Entra **Agent ID**—a sample LLM agent, the official **auth sidecar** (token acquisition and exchange), and a JWT-validating downstream API, covering **autonomous (app-only)** and **on-behalf-of (OBO)** patterns:

- For **agent and app developers**: call tools and APIs without putting blueprint `client_secret`, MSAL, or token caching in agent code
- For **identity and platform engineers**: a runnable Docker Compose reference for the [Entra Agent ID auth sidecar](https://learn.microsoft.com/en-us/entra/msidweb/agent-id-sdk/overview), with a sample resource API that validates Agent Identity tokens
- **Lab1** — PowerShell scripts provision Entra blueprint, Agent Identity, and Weather API; Compose runs the auth sidecar, LangChain/Ollama agent, and weather API (autonomous / app-only tokens)
- **Lab2** — same stack pattern with MSAL browser sign-in and OBO token exchange through the sidecar; separate `LabN/scripts/` and `.env` per lab (do not reuse Lab1 app IDs in Lab2)

in detail, please check [here](https://github.com/baoqger/entra-agent-id-sidecar-lab).
