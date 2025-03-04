---
layout: page
title: About
permalink: /about/
---

Hi, I'm Michael Maltsev, also known as m417z. I'm interested in reverse engineering and vulnerability research of any kind, from the low level world of operating systems to the high level world of front-end and browsers.

## Some of the things I did

### Research

* Research of the SMBGhost Windows kernel SMB bug ([CVE-2020-0796](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-0796)) and discovery of the SMBleed bug ([CVE-2020-1206](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-1206)). Writeups and code:
	* Exploiting SMBGhost for LPE: [Writeup](/archive/zecops/exploiting%20smbghost%20for%20a%20local%20privilege%20escalation), [POC Source Code](https://github.com/ZecOps/CVE-2020-0796-LPE-POC).
	* Exploiting SMBleed: [Writeup](/archive/zecops/01%20-%20chaining%20smbleed%20with%20smbghost), [POC Source Code](https://github.com/ZecOps/CVE-2020-1206-POC).
	* Combining SMBGhost and SMBleed for RCE: [Part 1](/archive/zecops/02%20-%20unauthenticated%20memory%20read%20-%20preparing%20the%20ground%20for%20an%20rce), [Part 2](/archive/zecops/03%20-%20from%20remote%20read%20(smbleed)%20to%20rce), [POC Source Code](https://github.com/ZecOps/CVE-2020-0796-RCE-POC).
* Webcam interception and protection in kernel mode in Windows: [Paper](https://www.virusbulletin.com/virusbulletin/2018/09/through-looking-glass-webcam-interception-and-protection-kernel-mode/), [VB2019 presentation](https://www.virusbulletin.com/conference/vb2019/abstracts/webcam-interception-and-protection-kernel-mode-windows-partner-presentation) ([slides](https://www.virusbulletin.com/uploads/pdf/conference_slides/2019/VB2019-Maltsev.pdf)).

### Publications

* [Paged Out!](https://pagedout.institute/) #2: C as a portable assembly - Porting 32-bit assembly code to 64-bit ([PDF](https://pagedout.institute/download/PagedOut_002_beta2.pdf), page 8).
* [Paged Out!](https://pagedout.institute/) #3: winapiexec - Run WinAPI functions from the command line ([PDF](https://pagedout.institute/download/PagedOut_003_beta1.pdf), page 25).

### Software

* [Windhawk](https://windhawk.net/) - The customization marketplace for Windows programs.
* [7+ Taskbar Tweaker](https://tweaker.ramensoftware.com/) - A Windows taskbar customization tool.
* [Unchecky](https://unchecky.com/) - A tool for Windows that automatically unchecks unrelated offers to keep potentially unwanted programs out of the computer. Was acquired by Reason Cybersecurity.
* [Ramen Software](https://ramensoftware.com/) - More software for Windows.

### Other projects

* [Winbindex - The Windows Binaries Index](https://winbindex.m417z.com/) - An index of Windows binaries, including download links for executables such as exe, dll and sys files.
* [NtDoc](https://ntdoc.m417z.com/) - Native API online documentation, based on the System Informer (formerly Process Hacker) [phnt headers](https://github.com/winsiderss/systeminformer/tree/master/phnt).
* [BitSniff](https://m417z.com/bitsniff/) - A tool for detecting Bitcoin-related communications in encrypted traffic, developed with [Niko Kudriastev](https://79jke.github.io/) during the [Bitcoin emBassy Hackathon](https://www.meetup.com/bitcoin-il/events/264327474/). [Technical details](https://79jke.github.io/BitSniff/), [presentation](https://www.youtube.com/watch?v=9S8xsDq3PTU).
* Contributor of the [MinHook](https://github.com/m417z/minhook) hooking library for Windows.
* [Microsoft Patch Tuesday Countdown](https://patch-tuesday.m417z.com/).
