---
title: "systemdでサーバーを定期的に再起動させる"
emoji: "⏰"
type: "tech"
topics:
  - "systemd"
  - "linux"
  - "cron"
published: true
published_at: "2021-09-28 00:23"
---

様々な事情で Linux マシンを定期的に再起動させたいことがある。cron を使うのがいちばんシンプルな方法ではあるが、今回は systemd のタイマー機能を使って実現してみる。

# 手順

1. `/etc/systemd/system/reboot.service` をつくる。

```systemd
[Unit]
Description = Reboot
RefuseManualStart = true
RefuseManualStop = true

[Service]
ExecStart = /sbin/reboot
```

2. `/etc/systemd/system/reboot.timer` をつくる。この例では、日本時間で毎週月曜日の午前3時を指定した。

```systemd
[Unit]
Description = Weekly Reboot Timer

[Timer]
OnCalendar = Mon 3:00 Asia/Tokyo

[Install]
WantedBy = timers.target
```

3. `sudo systemctl enable --now reboot.timer` でタイマーを有効化し、起動する。

4. `systemctl status reboot.timer` でタイマーが起動していることを確認する。

# ポイント

## 1. 再起動時刻は最大で1分遅れることがある

先述した例では省略しているが、`[Timer]` には `AccuracySec` という設定項目があり、デフォルトでは `1m` (1分)になっている。このとき、タイマーは `OnCalendar` で指定した時間から1分以内の適当なタイミングで実行される（`OnCalendar` で指定した時間より前に実行されることはない）。
正確に `3:00:00` に再起動を始めたい場合は、`AccuracySec = 1s` と設定する。

## 2. `reboot.service` に `WantedBy = multi-user.target` を書かない

`reboot.service` を書くとき、ついいつもの癖で

```systemd
[Install]
WantedBy = multi-user.target
```

と設定しそうになってしまうが、絶対にしてはいけない。この記述は、OS 起動時に自動で起動させたいサービスにのみ必要なものである[^1]。

[^1]: https://qiita.com/ledmonster/items/5f2e1633d4124cb978fe

これを書いた状態でうっかり `sudo systemctl enable reboot` とか `sudo systemctl enable reboot.service` を実行してしまうと、サーバーが起動するたびに `/sbin/reboot` が実行されることになってしまい、再起動ループに入って詰むので気をつける。（経験済み）
