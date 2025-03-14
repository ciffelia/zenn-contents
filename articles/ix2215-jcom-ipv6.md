---
title: IX2215でJ:COMのIPv6 (DHCPv6-PD)
emoji: 🎛️
type: tech
topics:
  - network
  - nec
  - ix
  - ix2215
  - ipv6
published: true
---

IX2215を買ったので使ってみることにしました。

https://x.com/ciffelia/status/1898278389594911196

私はJ:COM NETを契約しており、J:COMから貸与されているルータでIPv6を使っていました。IX2215でJ:COMのIPv6を使う情報は見つからなかったので、設定をメモしておきます。

## 最低限必要な設定

J:COMのIPv6サービスは、DHCPv6-PDで/56のプレフィックスを配布しているようです。これをIX2215で利用するために最低限必要な設定は次のようになります。別途、ケーブルモデムのルータ機能やWi-Fi AP機能は無効にする必要があります。

```
ip dhcp enable
!
ipv6 dhcp enable
ipv6 access-list permit-all permit ip src any dest any
ipv6 access-list wan-in-dhcpv6 permit udp src fe80::/64 sport eq 547 dest fe80::/64 dport eq 546
ipv6 access-list wan-in-icmpv6 option optimize
ipv6 access-list wan-in-icmpv6 permit icmp type 1 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 2 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 3 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 4 src any dest any
ipv6 access-list wan-in-ra deny icmp type 134 src any dest any
ipv6 access-list dynamic dyn-permit-all access permit-all
!
ip dhcp profile lan0
  dns-server 192.168.0.1
!
ipv6 dhcp client-profile pd
  ia-pd subscriber GigaEthernet2.0 ::/64 eui-64
!
ipv6 dhcp server-profile lan0
  dns-server dhcp
!
interface GigaEthernet0.0
  ip address dhcp receive-default
  ip napt enable
  ip napt hairpinning
  ipv6 enable
  ipv6 dhcp client pd
  ipv6 filter wan-in-dhcpv6 1 in
  ipv6 filter wan-in-icmpv6 2 in
  ipv6 filter wan-in-ra 3 in suppress-logging
  ipv6 filter dyn-permit-all 999 out
  no shutdown
!
interface GigaEthernet2.0
  ip address 192.168.0.1/24
  ip dhcp binding lan0
  ipv6 enable
  ipv6 dhcp server lan0
  ipv6 nd ra enable
  ipv6 nd ra other-config-flag
  proxy-dns ip enable
  no shutdown
!
```

いくつかの設定項目について解説します。

### DHCPv6-PDクライアント

```
ipv6 dhcp client-profile pd
  ia-pd subscriber GigaEthernet2.0 ::/64 eui-64
!
interface GigaEthernet0.0
  ipv6 dhcp client pd
```

GE0でDHCPv6-PDクライアントを起動し、配布されたプレフィックスからGE2にアドレスを割り当てます。もしVLANが必要な場合は、以下のように各サブインタフェースに別のアドレスを指定することになります。

```
ipv6 dhcp client-profile pd
  ia-pd subscriber GigaEthernet2.1 0:0:0:0::/64 eui-64
  ia-pd subscriber GigaEthernet2.2 0:0:0:1::/64 eui-64
!
interface GigaEthernet0.0
  ipv6 dhcp client pd
```

### RA・DHCPv6サーバー

```
ipv6 dhcp enable
!
ipv6 dhcp server-profile lan0
  dns-server dhcp
!
interface GigaEthernet2.0
  ipv6 enable
  ipv6 dhcp server lan0
  ipv6 nd ra enable
  ipv6 nd ra other-config-flag
```

IPv6アドレスをRAで配布します。また、DHCPv6クライアント機能で受け取ったDNSサーバー情報をDHCPv6で配布します。RAのOフラグを有効にすることで、クライアントにDHCPv6でDNSサーバー情報を取得させます。

なお、この設定ではAndroidでIPv6のDNSサーバー自動設定を利用できません。AndroidはRDNSSにのみ対応しておりDHCPv6に対応していないためです。IPv4のDNSサーバーが生きている限り問題になることはないと思います。

IX2215はRDNSSにも対応しているのですが、RDNSSではDNSサーバーのアドレスをコンフィグにハードコードする必要があります。DHCPv6クライアント機能で受け取ったDNSサーバー情報を配布することができません。IPv6ではIX2215自身のIPアドレスが変更される可能性があるため、プロキシDNS機能を利用してもこの問題を回避することはできません。

### アクセスリスト・フィルタ

```
ipv6 access-list permit-all permit ip src any dest any
ipv6 access-list wan-in-dhcpv6 permit udp src fe80::/64 sport eq 547 dest fe80::/64 dport eq 546
ipv6 access-list wan-in-icmpv6 option optimize
ipv6 access-list wan-in-icmpv6 permit icmp type 1 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 2 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 3 src any dest any
ipv6 access-list wan-in-icmpv6 permit icmp type 4 src any dest any
ipv6 access-list wan-in-ra deny icmp type 134 src any dest any
ipv6 access-list dynamic dyn-permit-all access permit-all
!
interface GigaEthernet0.0
  ipv6 filter wan-in-dhcpv6 1 in
  ipv6 filter wan-in-icmpv6 2 in
  ipv6 filter wan-in-ra 3 in suppress-logging
  ipv6 filter dyn-permit-all 999 out
```

IPv6にはNATがないため、NATに相当するファイアウォールの役割を果たすフィルタを設定することが望ましいです。ダイナミックフィルタ機能を利用して、内側から外側に向かうパケットに対する応答パケットのみを通します。

加えて、DHCPv6-PDのパケットとICMPv6のType 1, 2, 3, 4のパケットを通す必要があります。特にICMPv6のType 2はPath MTU Discoveryで利用するため重要です。公式の設定例ではこれらの記述が曖昧で、正しく動作させるのに苦労しました。

RAは使用しないので通す必要はありません。数十秒おきにWAN側からRAが来るので、ブロック時にログ出力しないよう設定しています。

## 必須ではないが入れると良い設定

冒頭の最低限の設定例では省略したものの、入れると良い設定をいくつか紹介します。パスワード、ログ、UFSキャッシュ、HTTP/SSHサーバーなどの一般的な設定は割愛します。

### NAPT・ダイナミックフィルタのセッションタイムアウト

```
ipv6 access-list dynamic timer icmp-timeout 60
ipv6 access-list dynamic timer tcp-fin-timeout 240
ipv6 access-list dynamic timer tcp-idle-timeout 7440
ipv6 access-list dynamic timer tcp-syn-timeout 240
ipv6 access-list dynamic timer udp-idle-timeout 300

interface GigaEthernet0.0
  ip napt translation tcp-timeout 7440
  ip napt translation syn-timeout 240
  ip napt translation finrst-timeout 240
```

IPv4ではNAPT (NAT)、IPv6ではダイナミックフィルタにより、外側からのパケットは内側からのパケットに対する応答のみを通すようになっています。これらのセッションの持続時間がデフォルトでは短すぎるため、十分な長さに設定します。

| IPv4 NAPT タイマー種類 | デフォルト値（秒） | 変更値（秒） |
| --- | --- | --- |
| tcp-timeout | 900 | 7440 |
| syn-timeout | 30 | 240 |
| finrst-timeout | 60, 1 | 240, 1 |
| udp-timeout | 300 | （変更なし） |
| icmp-timeout | 60 | （変更なし） |
| dns-timeout | 60 | （変更なし） |
| gre-timeout | （other-timeoutと同じ） | （変更なし） |
| other-timeout | 60 | （変更なし） |

| IPv6 ダイナミックフィルタ タイマー種類 | デフォルト値（秒） | 変更値（秒） |
| --- | --- | --- |
| tcp-syn-timeout | 30 | 240 |
| tcp-fin-timeout | 5 | 240 |
| tcp-idle-time | 300 | 7440 |
| udp-idle-time | 30 | 300 |
| dns-timeout | 30 | （変更なし） |
| icmp-timeout | 30 | 60 |
| global-timeout | 60 | （変更なし） |

以下の記事を参考にしました。

https://qiita.com/ykatombn/items/51edf7f62aedd8ea1d76#%E3%82%BF%E3%82%A4%E3%83%A0%E3%82%A2%E3%82%A6%E3%83%88%E8%A8%AD%E5%AE%9A

利用できるポート数が制限されているフレッツ光のIPoE環境では、これより短縮すべきとの意見もあります。J:COMでは全てのポートを利用できるため、短くする理由はないでしょう。

### NAPTキャッシュ数制限

```
interface GigaEthernet0.0
  ip napt translation max-entries per-address 4096
```

1台の端末が大量の接続を開くと、NAPTのエントリ数が上限に達し他の端末も接続を開けなくなってしまう可能性があります。これを防ぐため、クライアントあたりの最大エントリ数を制限します。

### 静的フィルタ

```
ip access-list permit-all permit ip src any dest any
ip access-list wan-out-basic option optimize
ip access-list wan-out-basic deny udp src any sport any dest any dport range 137 139
ip access-list wan-out-basic deny tcp src any sport any dest any dport range 137 139
ip access-list wan-out-basic deny udp src any sport any dest any dport eq 445
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 445
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 2049
ip access-list wan-out-basic deny udp src any sport any dest any dport eq 2049
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 1243
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 12345
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 27374
ip access-list wan-out-basic deny tcp src any sport any dest any dport eq 31785
ip access-list wan-out-basic deny udp src any sport any dest any dport eq 31789
ip access-list wan-out-basic deny udp src any sport any dest any dport eq 31791
!
ipv6 access-list wan-out-basic option optimize
ipv6 access-list wan-out-basic deny udp src any sport any dest any dport range 137 139
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport range 137 139
ipv6 access-list wan-out-basic deny udp src any sport any dest any dport eq 445
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 445
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 2049
ipv6 access-list wan-out-basic deny udp src any sport any dest any dport eq 2049
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 1243
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 12345
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 27374
ipv6 access-list wan-out-basic deny tcp src any sport any dest any dport eq 31785
ipv6 access-list wan-out-basic deny udp src any sport any dest any dport eq 31789
ipv6 access-list wan-out-basic deny udp src any sport any dest any dport eq 31791
!
interface GigaEthernet0.0
  ip filter wan-in-basic 1 in
  ip filter permit-all 999 in
  ipv6 filter wan-out-basic 1 out
```

最近の家庭用ルーターのデフォルト設定を参考にして、いくつか外向きパケットにポート制限を設定しました。NetBIOSやSMB・NFSが使用するポート、かつてマルウェアが利用していたポートを制限しているようです。これらの設定が現在においても有効といえるのか不明ですが、念のため設定しておきます。NAPTやダイナミックフィルタを設定してあるので、内向きパケットを制限する必要はありません。

https://www.aterm.jp/function/wx11000t12/appendix/initialvalue_local.html

### 送信元や宛先が不正なパケットを破棄

```
ip access-list wan-in-invalid option optimize
ip access-list wan-in-invalid deny ip src 0.0.0.0/8 dest any
ip access-list wan-in-invalid deny ip src 127.0.0.0/8 dest any
ip access-list wan-in-invalid deny ip src 169.254.0.0/16 dest any
ip access-list wan-in-invalid deny ip src 192.0.2.0/24 dest any
ip access-list wan-in-invalid deny ip src 198.51.100.0/24 dest any
ip access-list wan-in-invalid deny ip src 203.0.113.0/24 dest any
ip access-list wan-out-invalid option optimize
ip access-list wan-out-invalid deny ip src any dest 0.0.0.0/8
ip access-list wan-out-invalid deny ip src any dest 127.0.0.0/8
ip access-list wan-out-invalid deny ip src any dest 169.254.0.0/16
ip access-list wan-out-invalid deny ip src any dest 192.0.2.0/24
ip access-list wan-out-invalid deny ip src any dest 198.51.100.0/24
ip access-list wan-out-invalid deny ip src any dest 203.0.113.0/24
!
ipv6 access-list wan-in-invalid option optimize
ipv6 access-list wan-in-invalid deny ip src 2001:db8::/32 dest any
ipv6 access-list wan-out-invalid option optimize
ipv6 access-list wan-out-invalid deny ip src any dest ::/96
ipv6 access-list wan-out-invalid deny ip src any dest ::ffff:0:0/96
ipv6 access-list wan-out-invalid deny ip src any dest 2001:db8::/32
ipv6 access-list wan-out-invalid deny ip src any dest fec0::/10
!
interface GigaEthernet0.0
  ip filter wan-in-invalid 2 in
  ip filter wan-out-invalid 1 out
  ip filter permit-all 999 out
  ipv6 filter wan-in-invalid 2 in
  ipv6 filter wan-out-invalid 2 out
```

送信元や宛先が不正なパケットを破棄します。プライベートIPアドレスやループバックアドレス、ドキュメント用IPアドレスなどが該当します。このようなパケットはISPが破棄するはずですが、念の為設定しておきます。

また、本来はDHCPv6-PDで配布されたプレフィックスのアドレスを送信元とする外側からのパケットは破棄すべきです。残念ながらIX2215にはそれを実現する手段がないようです。

## まとめ

IX2215でJ:COMのIPv6を利用するための設定を紹介しました。私はこの設定にポートVLAN・タグVLANを追加したり、RAの間隔やUFSキャッシュのパラメータを調整して使っています。この記事が参考になれば幸いです。

## 参考

https://jpn.nec.com/univerge/ix/Manual/index.html

https://nyanshiba.com/blog/nec-ix/

https://kusoneko.blogspot.com/2021/11/bought-nec-ix2215-for-home-broadband-router.html

https://takachan.jra.net/blog/archives/1216

https://www.ymstmsys.jp/blog/2023/11/03/

https://www.janog.gr.jp/meeting/janog49/wp-content/uploads/2021/12/JANOG49-ixpipv6-20220126_01.pdf

https://www.nic.ad.jp/ja/materials/iw/2017/proceedings/s03/s3-fujisaki.pdf

https://www.v6pc.jp/jp/wg/coexistenceWG/v6hgw-swg.phtml
