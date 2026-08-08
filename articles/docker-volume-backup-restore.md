---
title: 'パーミッションを保持したままDockerのボリュームをバックアップ・復元する'
emoji: '💽'
type: 'tech'
topics:
  - 'docker'
  - 'linux'
published: true
---

多分これが一番早いと思います

# バックアップ

Dockerボリュームの実体は`/var/lib/docker/volumes/$VOLUME_NAME/_data`にあります。tarを使うことで、パーミッションや所有者といったメタデータを含めてバックアップできます。

```sh
VOLUME_NAME=test_volume
BACKUP_DESTINATION=./backup.tar.gz

sudo tar -caf "$BACKUP_DESTINATION" -C "/var/lib/docker/volumes/$VOLUME_NAME" _data
```

# 復元

復元する際は、`--preserve-permissions --numeric-owner`を指定するとメタデータを保持して展開できます。`--numeric-owner`を指定しないとUIDやGIDが書き換わってしまうことがあるので注意が必要です[^1]。

```sh
VOLUME_NAME=test_volume
RESTORE_SOURCE=./backup.tar.gz

docker volume create "$VOLUME_NAME"
sudo rm -r "/var/lib/docker/volumes/$VOLUME_NAME/_data"
sudo tar -xf "$RESTORE_SOURCE" -C "/var/lib/docker/volumes/$VOLUME_NAME" --preserve-permissions --numeric-owner
```

[^1]: https://jigoku1119.hatenablog.jp/entry/2019/11/08/170050
