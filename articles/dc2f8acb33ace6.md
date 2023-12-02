---
title: 'Docker Compose v2が正式リリースされたのでインストールしてみる'
emoji: '🐳'
type: 'tech'
topics:
  - 'docker'
  - 'linux'
  - 'container'
published: true
published_at: '2021-09-28 22:37'
---

先程Docker Compose v2.0.0がリリースされた。[公式ブログ](https://www.docker.com/blog/)等では言及されていないが、おそらくCompose v2の正式リリース版ではないかと思われる。

https://github.com/docker/compose/releases/tag/v2.0.0

v1は単独のバイナリ（`docker-compose`）として配布されていたが、v2はCLI pluginとして配布されているため、`$HOME/.docker/cli-plugins`にインストールする必要がある。

```sh
mkdir -p $HOME/.docker/cli-plugins
curl -Lf -o $HOME/.docker/cli-plugins/docker-compose "https://github.com/docker/compose/releases/download/v2.1.1/docker-compose-linux-x86_64"
chmod +x $HOME/.docker/cli-plugins/docker-compose
```

```sh
$ docker compose version
Docker Compose version v2.1.1
```
