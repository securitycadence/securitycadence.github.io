---
layout: posts
title: "Week 2: Disable LLMNR"
date: 2020-01-20
tags: windows passwords
---

## What?

Link Local Multicast Name Resolution (LLMNR) is a technology built into all modern versions of Windows (From Windows Vista on) and enabled by default. LLMNR is also implemented by systemd-resolved in Linux. It comes into play when you attempt to access a host on your network that is neither in DNS or in your client's hosts file. Windows will send out a packet to all clients on your local subnet asking if anyone has the hostname being requested. The idea behind LLMNR is to allow name resolution on a small network without need of setting up and maintaining a DNS server.

## Why?

Windows trusts any device that responds to an LLMNR request. This is particularly dangerous for SMB requests because once a device responds stating that it is the requested resource, the Windows client will attempt to authenticate by sending a hash of the user's password which an attacker can then take offline and crack. Tools such as [Responder](https://github.com/SpiderLabs/Responder) are purpose built to sit on a network and respond to LLMNR requests. Exploiting LLMNR is often an attacker's first foothold on a victim's domain.

## How?

Disabling LLMNR is best done across the enterprise via GPO.

#### 1. GPO

Probably the easiest and best way of removing the curse LLMNR from your enterprise is to configure a GPO. This setting can be found at **Computer Configuration -> Administrative Templates -> Network -> DNS ClientEnable Turn Off Multicast Name Resolution**.  Simply set this value to **Enabled**. And Ta-Da! One of the lowest hanging fruit in your environment is effectively raised out of an attacker's reach.

#### 2. Command Line

Alternatively you can disable LLMNR at the individual workstation layer via the registry. Issuing the following commands from the command line will do the trick:
`REG ADD  “HKLM\Software\policies\Microsoft\Windows NT\DNSClient”``
`REG ADD  “HKLM\Software\policies\Microsoft\Windows NT\DNSClient” /v ” EnableMulticast” /t REG_DWORD /d “0” /f`


## Gotchas?

The obvious gotcha here is ruining name resolution in an environment that is relying on it. In a modern corporation this seems incredibly unlikely. Still, always be aware of your environment before making changes. If your organization does rely on LLMNR chances are it is in a small, isolated segment where it did not make sense to stand up a DNS server. If that is the case, then don't let not being able to achieve perfection from getting in the way of achieving good. Scope your GPO appropriately to exclude your hosts that rely on LLMNR, and disable it everywhere else.

In a future post we will discuss using LLMNR as a cheap and easy way to detect attackers on your network. For now, check the external links section below for a couple of links for making LLMNR a fantastic canary in your coal mine.

## Additional Thoughts And External Links

I have not once found myself in an environment where LLMNR was required. Further, I have asked on Twitter and during talks at a number of conferences and InfoSec gatherings whether anyone has encountered an environment using LLMNR. Not once has someone responded yes. In the early days of networking LLMNR may have made sense, but considering that it made its debut in 2007 it just seems like a half-baked solution in search of a problem that nobody was having. Frankly, if you're an environment where disabling LLMNR will break something, then I'd argue that your first step should be setting up a simple DNS server. I also think that Microsoft should be ashamed of themselves for continuing to enable it by default.

More information regarding LLMNR:

* [Wikipedia Entry](https://en.wikipedia.org/wiki/Link-Local_Multicast_Name_Resolution)
* [RFC4795](https://tools.ietf.org/html/rfc4795)
* [Tutorial for Capturing LLMNR Packets with Wireshark](https://en.wikiversity.org/wiki/Wireshark/LLMNR)
* [Tutorial for Using Responder to Exploit LLMNR](https://www.4armed.com/blog/llmnr-nbtns-poisoning-using-responder/)
* [Respounder - A Tool for Injecting Fake LLMNR Requests for Detecting Responder](https://github.com/codeexpress/respounder)
* [Bootsy - An automated deployment system for pushing Respounder and Artillery to a Raspberry Pi](https://github.com/IndustryBestPractice/Bootsy)
