---
layout: page
title: About
permalink: /about/
---

Hi, I'm Michael Maltsev, also known as m417z. I'm interested in reverse engineering and vulnerability research of any kind, from the low level world of operating systems to the high level world of front-end and browsers.

Currently a security researcher at ZecOps.

## Some of the things I did

### Research

* Research of the SMBGhost Windows kernel SMB bug ([CVE-2020-0796](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-0796)) and discovery of the SMBleed bug ([CVE-2020-1206](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-1206)). Writeups and code:
	* Exploiting SMBGhost for LPE: [Writeup](https://blog.zecops.com/vulnerabilities/exploiting-smbghost-cve-2020-0796-for-a-local-privilege-escalation-writeup-and-poc/), [POC Source Code](https://github.com/ZecOps/CVE-2020-0796-LPE-POC).
	* Exploiting SMBleed: [Writeup](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-chaining-smbleed-cve-2020-1206-with-smbghost/), [POC Source Code](https://github.com/ZecOps/CVE-2020-1206-POC).
	* Combining SMBGhost and SMBleed for RCE: [Part 1](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-ii-unauthenticated-memory-read-preparing-the-ground-for-an-rce/), [Part 2](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-iii-from-remote-read-smbleed-to-rce/), [POC Source Code](https://github.com/ZecOps/CVE-2020-0796-RCE-POC).
* Webcam interception and protection in kernel mode in Windows: [Paper](https://www.virusbulletin.com/virusbulletin/2018/09/through-looking-glass-webcam-interception-and-protection-kernel-mode/), [VB2019 presentation](https://www.virusbulletin.com/conference/vb2019/abstracts/webcam-interception-and-protection-kernel-mode-windows-partner-presentation) ([slides](https://www.virusbulletin.com/uploads/pdf/conference_slides/2019/VB2019-Maltsev.pdf)).

### Publications

* [Paged Out!](https://pagedout.institute/) #2: C as a portable assembly - Porting 32-bit assembly code to 64-bit ([PDF](https://pagedout.institute/download/PagedOut_002_beta2.pdf), page 8).

### Software

* [Unchecky](https://unchecky.com/) - A tool for Windows that automatically unchecks unrelated offers to keep potentially unwanted programs out of the computer. Was acquired by Reason Cybersecurity.
* [7+ Taskbar Tweaker](https://tweaker.rammichael.com/) - A Windows taskbar customization tool.

### Other projects

* [Winbindex - The Windows Binaries Index](https://winbindex.m417z.com/) - An index of Windows binaries, including download links for executables such as exe, dll and sys files.
* [BitSniff](https://m417z.com/bitsniff/) - A tool for detecting Bitcoin-related communications in encrypted traffic, developed with [Niko Kudriastev](https://79jke.github.io/) during the [Bitcoin emBassy Hackathon](https://www.meetup.com/bitcoin-il/events/264327474/). [Technical details](https://79jke.github.io/BitSniff/), [presentation](https://www.youtube.com/watch?v=9S8xsDq3PTU).
* Contributor of the [MinHook](https://github.com/m417z/minhook) hooking library for Windows.
* [Microsoft Patch Tuesday Countdown](https://m417z.com/ms-patch-tuesday/).
