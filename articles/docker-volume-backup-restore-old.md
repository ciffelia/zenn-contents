---
title: 'パーミッションを保持したままDockerのボリュームをバックアップ・復元する'
emoji: '💽'
type: 'tech'
topics:
  - 'docker'
  - 'linux'
published: false
---

Docker でボリュームをバックアップするのは意外に難しい。Google で調べてみると、一時コンテナを作成してボリュームをマウントして`docker cp`を使ってデータを取り出すという方法が提示されている。特にハマりがちな罠として、バックアップにパーミッションや所有者などのメタデータが含まれておらず、復元後にコンテナが起動しないという問題がある。

https://twitter.com/

# バックアップ

Docker ボリュームの実体は`/var/lib/docker/volumes/$VOLUME_NAME/_data`にある。tar を使うことで、パーミッションや所有者の情報を含めてバックアップできる。

```sh
VOLUME_NAME=test_volume
BACKUP_DESTINATION=./backup.tar.gz

sudo tar -czf "$BACKUP_DESTINATION" -C "/var/lib/docker/volumes/$VOLUME_NAME" _data
```

# 復元

復元する際は、`--preserve-permissions --numeric-owner`を指定するとメタデータを保持して展開できる。`--numeric-owner`を指定しないと UID や GID が書き換わってしまうことがあるので注意。

```sh
VOLUME_NAME=test_volume
RESTORE_SOURCE=./backup.tar.gz

docker volume create "$VOLUME_NAME"
sudo rm -r "/var/lib/docker/volumes/$VOLUME_NAME/_data"
sudo tar -xzf "$RESTORE_SOURCE" -C "/var/lib/docker/volumes/$VOLUME_NAME" --preserve-permissions --numeric-owner
```

# 追記: パーミッションを保持する必要がない場合

パーミッションを保持する必要がない場合は`docker`コマンドのみでバックアップと復元ができる。

## バックアップ

```sh
VOLUME_NAME=test_volume
BACKUP_DESTINATION=./volume_backup
TEMP_CONTAINER_NAME=volume_backup_temp
TEMP_CONTAINER_IMAGE=hello-world

docker container create --name "$TEMP_CONTAINER_NAME" -v "$VOLUME_NAME:/volume:ro" "$TEMP_CONTAINER_IMAGE"
docker cp "$TEMP_CONTAINER_NAME:/volume" "$BACKUP_DESTINATION"
docker rm "$TEMP_CONTAINER_NAME"
```

## 復元

```sh
VOLUME_NAME=test_volume
RESTORE_SOURCE=./volume_backup
TEMP_CONTAINER_NAME=volume_backup_temp
TEMP_CONTAINER_IMAGE=hello-world

docker volume create "$VOLUME_NAME"
docker container create --name "$TEMP_CONTAINER_NAME" -v "$VOLUME_NAME:/volume" "$TEMP_CONTAINER_IMAGE"
docker cp "$RESTORE_SOURCE" "$TEMP_CONTAINER_NAME:/volume"
docker rm "$TEMP_CONTAINER_NAME"
```
