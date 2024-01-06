---
title: 'TailscaleでSSHしてルートパーティションの暗号化を解除する'
emoji: '🔐'
type: 'tech'
topics:
  - 'luks'
  - 'tailscale'
  - 'linux'
  - 'ssh'
published: true
---

LUKSによるルートパーティションの暗号化を、Tailscale + SSHでリモートから解除する方法を紹介します。Tailscaleと軽量SSH実装のDropbearをinitramfsに入れることで実現します。

ルートパーティションの暗号化は完了しており、マシンに物理的にアクセスすれば暗号化を解除できる状態になっているものとします。

# 1. initramfsで使用するIPアドレスの設定

initramfsからインターネットに接続するために、IPアドレス等の設定を行います。DHCPを使用する場合はこの項目はスキップできます。

`/etc/default/grub`を編集し、`GRUB_CMDLINE_LINUX_DEFAULT`に`ip=`を追加します。

```
GRUB_CMDLINE_LINUX="ip=<client-ip>:<server-ip>:<gw-ip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>:<ntp0-ip>"
```

※不要な項目は省略できます。`<server-ip>`はNFSサーバに接続する場合に指定するもので、今回は不要です。

例えば、次のように設定します。

```
GRUB_CMDLINE_LINUX="ip=192.168.0.10::192.168.0.1:255.255.255.0:my-hostname-initramfs:eth0:none"
```

`GRUB_CMDLINE_LINUX`で指定できるパラメータの一覧は次のページにまとめられています。

https://www.kernel.org/doc/html/v6.6/admin-guide/kernel-parameters.html

`ip=`で指定できる項目は次のページにまとめられています。

https://www.kernel.org/doc/html/v6.6/admin-guide/nfs/nfsroot.html

# 2. dropbear-initramfsの設定

軽量SSH実装のDropbearをinitramfsに入れます。

```sh
sudo apt-get update
sudo apt-get install dropbear-initramfs
```

SSHの公開鍵を登録します。

```sh
echo 'ssh-ed25519 ...' | sudo tee -a /etc/dropbear/initramfs/authorized_keys
sudo chmod 600 /etc/dropbear/initramfs/authorized_keys
```

# 3. tailscale-initramfsの設定

Tailscaleをinitramfsに入れます。今回は次の実装を利用します。

https://github.com/darkrain42/tailscale-initramfs

```sh
# Add the repository
sudo mkdir -p --mode=0755 /usr/local/share/keyrings
curl -fsSL https://darkrain42.github.io/tailscale-initramfs/keyring.asc | sudo tee /usr/local/share/keyrings/tailscale-initramfs-keyring.asc >/dev/null
echo 'deb [signed-by=/usr/local/share/keyrings/tailscale-initramfs-keyring.asc] https://darkrain42.github.io/tailscale-initramfs/repo stable main' | sudo tee /etc/apt/sources.list.d/tailscale-initramfs.list >/dev/null

# Install tailscale-initramfs
sudo apt-get update
sudo apt-get install tailscale-initramfs
```

Tailscaleの[admin console](https://login.tailscale.com/admin/settings/keys)から、デバイスを追加するためのauth keyを生成します。reusableとephemeralを有効にし、必要に応じてタグを設定してください。

生成したauth keyは`/etc/tailscale/initramfs/config`に追加します。

```sh
sudo vi /etc/tailscale/initramfs/config
```

:::message
Auth keyは平文でinitramfsに保存されます。この鍵を使用するとTailnetにデバイスを追加できるため、必ず適切なACLを設定してください。
:::

# 4. initramfsの再構築

initramfsを再構築します。

```sh
update-initramfs -c -k all
update-grub
```

# 5. リブート

再起動して動作を確認します。

```sh
sudo reboot
```

SSHでinitramfsに接続します。ホスト名がわからない場合はTailscaleのadmin consoleから確認できます。

```sh
ssh root@my-hostname-initramfs
```

`cryptroot-unlock`コマンドでルートパーティションの暗号化を解除します。成功するとSSH接続は切断され、Linuxが起動します。

```sh
cryptroot-unlock
```
