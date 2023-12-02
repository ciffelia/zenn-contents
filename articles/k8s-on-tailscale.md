---
title: 'Tailscaleネットワーク上でKubernetesクラスタを構築する'
emoji: '👻'
type: 'tech'
topics:
  - 'kubernetes'
  - 'tailscale'
published: false
---

Tailscale ネットワーク上のマシンで Kubernetes クラスタを構築するにあたって、注意する点を解説します。

NOTE: 公式記事は Pod を Tailscale ネットワークに乗せる方法
https://tailscale.com/kb/1185/kubernetes/

# ポイント 1. `systemd-resolved`を無効化する

```sh
sudo rm /etc/resolv.conf
sudo cp /run/systemd/resolve/resolv.conf /etc/resolv.conf

# ヘッダとts.netのsearchを消す
sudo vi /etc/resolv.conf

sudo systemctl disable systemd-resolved.service
sudo systemctl stop systemd-resolved.service
sudo reboot
```

# ポイント 1. ノード間で通信する際に使う IP アドレスを設定する

```sh
# 他のノードからこのノードにアクセスするときのIPを、kubeletの引数で指定する
# 本来は kubeadm join --config ./join-config.yml を使って指定するのが望ましいが、面倒なのでこれで済ませる
# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#non-public-ip-used-for-containers
echo "KUBELET_EXTRA_ARGS='--node-ip=$(tailscale ip -4),$(tailscale ip -6)'" | sudo tee /etc/default/kubelet
```

# ポイント 2. `kubeadm init`

```sh
sudo kubeadm init \
  --control-plane-endpoint=xxx.yyy.ts.net:6443 \
  --apiserver-advertise-address=$(tailscale ip -4) \
  --pod-network-cidr=10.244.0.0/16
```

# ポイント 3. `flannel`

```sh
mkdir flannel
cd flannel
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# kube-flannel の args に --iface=tailscale0 を追加
vi ./kube-flannel.yml
kubectl apply -f ./kube-flannel.yml
```
