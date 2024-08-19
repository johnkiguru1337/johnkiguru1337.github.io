---
title: "Windows HandBook"
date: 2024-07-31
draft: false
summary: "This is a windows cheatsheet handbook"
tags: ["windows", "cheatsheet"]
---

## Windows

In most `corporate environments`, you will find extensive usage of Windows Microsoft Operating System. It is barely inevitable to miss this operating system during a security assessment. Most `Unix-Oses` are used for server operations while `Windows` is used for day to day operations.

### Helpful Points
For Domain-level Security, a server, be it smb or ftp acts as a member of a `windows domain`. Each domain has at least one `domain controller` usually windows NT server providing password authentication. The `DC` Provides the `workgroup` with a definitive password server. The `DCs` keep track of users and passwords in their own `NTDS.dit` and `Security Authentication Module` (SAM) and authenticate each user when they log in for the first time and wish to access another machine's share.

All the secrets in an Active Directory Environment are stored in `NTDS.dit` file, We can use it to carry out a `DCSync` attack.

If you found a `bi-directional parent-child trust`, one way you would abuse it, is the using `Golden-ticket attack`.

### Tricks
- At times Tools will fail in a windows reverse shell or remote session. In this case you can try get another shell and try the tool again. What can I say at times Windows is `Weird!`
- Running `Windows Powershell` exist as a default rule in most modern EDRs. It is almost inevitable for an EDR to miss you when you execute `powershell`. Instead windows `cmd` is a little bit safer and stealthier since, EDR won't really know whether the process is malicious not until it inspects the commands being issued.