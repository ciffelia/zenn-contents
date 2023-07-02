---
title: '【PostgreSQL】「database "foo" has a collation version mismatch」への対処法'
emoji: '🕌'
type: 'tech'
topics:
  - 'postgresql'
  - 'database'
  - 'rdb'
published: false
---

PostgreSQL を使っていると、以下のようなエラーが出ることがあります。この記事ではこのエラーの原因と対処法について解説します。

```
2023-07-02 06:27:49.880 UTC [338] WARNING:  database "foo" has a collation version mismatch
2023-07-02 06:27:49.880 UTC [338] DETAIL:  The database was created using collation version 2.31, but the operating system provides version 2.36.
2023-07-02 06:27:49.880 UTC [338] HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE foo REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
```

# 対処法

次のクエリを実行すると解決します。

:::message
念のため、実行前にデータベースのバックアップを取得してください。
:::

```sql
-- foo はデータベース名
REINDEX DATABASE foo;
ALTER DATABASE foo REFRESH COLLATION VERSION;
```

:::message alert
`REINDEX`にかかる時間はインデックスの大きさによって異なります。`REINDEX`の実行中は、そのインデックスの元となるテーブルへの書き込みと、そのインデックスを使用する読み込みがブロックされます。稼働中のデータベースで実行する場合はユーザーに与える影響を考慮してください。
:::

# 原因

PostgreSQL は、データベースの照合順序 (collation)を計算するために libc または ICU を使用します。データベースを作成した時点での libc または ICU のバージョンと、現在のバージョンが異なる場合、インデックスが破損する危険性があるためこのような警告が表示されます。
私のケースでは、OS のアップデートで libc のバージョンが v2.31 から v2.36 に変わったためこのような警告が表示されていたようです。

これを解決するには、`REINDEX`でインデックスを再構築し、`ALTER DATABASE foo REFRESH COLLATION VERSION`でデータベースに記録されている照合順序のバージョン（libc や ICU のバージョン）を更新する必要があります。
注意点として、`ALTER DATABASE foo REFRESH COLLATION VERSION`は実際に照合順序が更新されているかどうかをチェックしません。`REINDEX`を実行しなければ、単に警告が消えるだけで問題を解決したことにはなりません。

なお、`ALTER DATABASE foo REFRESH COLLATION VERSION`を実行してから`REINDEX`を実行しても恐らく問題はありません。

# 参考

https://dba.stackexchange.com/q/324649
https://forum.greenbone.net/t/the-database-was-created-using-collation-version-2-35-but-the-operating-system-provides-version-2-36/13562
https://www.postgresql.org/docs/15/sql-altercollation.html#SQL-ALTERCOLLATION-NOTES
https://www.postgresql.org/docs/15/sql-reindex.html
