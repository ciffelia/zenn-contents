---
title: WSLからpbcopy/pbpasteを使う
emoji: 📋
type: tech
topics:
  - wsl
  - linux
  - windows
published: true
---

```sh
alias pbcopy='iconv -f utf8 -t utf16le | clip.exe'
alias pbpaste='pwsh.exe -Command Get-Clipboard'
```
