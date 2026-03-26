---
title: "Brut-force attacks protection with fail2ban"
date: 2024-06-20T19:24:03+01:00
description: "Use fail2ban to block nginx botsearch requests"
---

## Introduction

In this post I will show you how to use fail2ban to block requests from bots searching for vulnerabilities in your nginx server.

## Install fail2ban

First you need to install fail2ban, on debian based systems you can do this with:

```
apt install fail2ban
```

## Create a jail

Create a custom jail file in `/etc/fail2ban/jail.d/nginx-botsearch.local` with the following content:

```
[nginx-botsearch]
enabled   = true
port      = http,https
filter    = nginx-botsearch
failregex = ^.+ <HOST> \- \S+ \[\] \"(GET|POST|HEAD) \/.* \S+\" 404 .+$
            ^.+ <HOST> \- \S+ \[\] \"(GET|POST|HEAD) \/.* \S+\" 5\d{2} .+$
            ^ \[error\] \d+#\d+: \*\d+ (\S+ )?\"\S+\" (failed|is not found) \(2\: No such file or directory\), client\: <HOST>\, server\: \S*\, request: \"(GET|POST|HEAD) \/<block> \S+\"\, .*?$
            ^.+ <HOST> \- \S+ \[\] \"(GET|POST|HEAD) \/.* \S+\" 5\d{2} .+$
            ^ \[error\] \d+#\d+: \*\d+ (\S+ )?\"\S+\" (failed|is not found) \(2\: No such file or directory\), client\: <HOST>\, server\: \S*\, request: \"(GET|POST|HEAD) \/<block> \S+\"\, .*?$
```

This file enables the nginx-botsearch filter and overried its failregex to match any 404 and 5xx status codes.

Reload fail2ban with:

```
systemctl reload fail2ban
```

