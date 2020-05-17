---
title: "Covenant BlockDll (Injector 2.0)"
categories:
  - Blog
tags:
  - C#
  - Covenant
  - Red Teaming
---

Using Microsoft mitigation policies to protect malware

> This is built upon the work of [https://blog.xpnsec.com/protecting-your-malware/]([https://blog.xpnsec.com/protecting-your-malware/). 

A common feature of endpoint security products is their ability to load their own DLLs into processes in order to analyse and report on suspicious behaviour. This ability completely circumnavigates any protections a malicious actor (or your friendly neighbourhood PenTester) may have put in place to obfuscate their malicious payloads from signature-based detection; as it will directly hook into API calls to analyse any suspicious functions that are called.

Fortunately (depending on your perspective), Microsoft actually provides, and uses, a method of preventing this from happening via their mitigation policies. If we take a look at the latest Microsoft Edge process, we can see this in action.

![Edge]({{site.url}}/assets/posts/blockdll/Edge.png)

Here we can see that the Microsoft Edge program is running with ‘Store Only’ permissions, meaning that only Windows Store signatures are allowed to interact with the process.
Based upon @_xpn_’s research, we can replicate this within our own tooling by utilising the ‘CreateProcess’ Windows API in conjunction with a STARTUPINFOX struct containing the mitigation policy of our choice. 

A bit of GoogleFu and we are presented with the following options for applicable mitigation options under the ‘UpdateProcThreadAttribute’ API function.

![MitigationOptions]({{site.url}}/assets/posts/blockdll/MitigationOptions.png)

Ideally, we want to use **CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON** as this should only allow Microsoft binaries to inject into our process.

In order to accomplish this in C# we can create the relevant flag attribute.

![Flags]({{site.url}}/assets/posts/blockdll/Flags.png)

And subsequently call it when using the ‘UpdateProcThreadAttribute’ API function.

![UpdateAtribute]({{site.url}}/assets/posts/blockdll/UpdateAtribute.png)


A full C# POC of this process was kindly shared by [@_Rastamouse](https://gist.github.com/rasta-mouse/af009f49229c856dc26e3a243db185ec). This has the added advantage of also spoofing PPIDs if needed.

Building off of this it didn’t take much to re-write the teams original shellcode injector to utilise this new method.

![Covenant1]({{site.url}}/assets/posts/blockdll/Covenant1.png)

![Permisions2]({{site.url}}/assets/posts/blockdll/permisions2.png)

There are some drawbacks to this technique. Because you are loading specific API calls in order to implement the mitigation, a few endpoint vendors are now specifically flagging this capability, regardless of whether malicious activity follows.

![VT1]({{site.url}}/assets/posts/blockdll/VT1.png)

Despite the test executable having no imbedded payload it still got flagged by a few AV products (though this result should be taken with a pinch of salt due to AV companies traditionally turning up their detections of VirusTotal).

Additionally, some AV have been shown to specifically utilise DLLs that are signed by Microsoft in order to circumnavigate this.
