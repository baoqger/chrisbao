---
title: fabio-source-code-study
date: 2021-01-29 22:09:56
tags:
---

Fabio 官方文档上的说明:

Fabio is an HTTP and TCP reverse proxy that configures itself with data from Consul.

Traditional load balancers and reverse proxies need to be configured with a config file. The configuration contains the hostnames and paths the proxy is forwarding to upstream services. This process can be automated with tools like consul-template that generate config files and trigger a reload.

Fabio works differently since it updates its routing table directly from the data stored in Consul as soon as there is a change and without restart or reloading.

When you register a service in Consul all you need to add is a tag that announces the paths the upstream service accepts, e.g. urlprefix-/user or urlprefix-/order and fabio will do the rest.


主要依赖两个部分:
watchBackend()，它会维护一个routeTable;
startServers(), 它会launch proxy server;

