---
title: WSLからxdg-openでWindowsの既定ブラウザを開けるようにする
emoji: 🌐
type: tech
topics:
  - wsl
  - linux
  - windows
published: true
---

```sh
mkdir -p ~/.local/share/applications
cat << EOS > ~/.local/share/applications/file-protocol-handler.desktop
[Desktop Entry]
Type=Application
Version=1.0
Name=File Protocol Handler
NoDisplay=true
Exec=rundll32.exe url.dll,FileProtocolHandler
EOS

xdg-settings set default-web-browser file-protocol-handler.desktop
```

# 参考

https://diary.shu-cream.net/WSL2%E3%81%AE%E4%B8%AD%E3%81%8B%E3%82%89Windows%E5%81%B4%E3%81%AE%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%82%92%E9%96%8B%E3%81%8F
