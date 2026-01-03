---
command: id
category: user-identity
tags: [linux, security, user, group]
---

# id

ユーザーID、グループID、所属グループを表示。

## 基本

```bash
$ id
uid=1000(user) gid=1000(user) groups=1000(user),27(sudo),999(docker)
```

## オプション

```bash
$ id -u          # UID のみ
$ id -g          # プライマリGID のみ
$ id -G          # 全グループID
$ id -nG         # 全グループ名
$ id username    # 特定ユーザーの情報
```

## セキュリティ観点

| グループ | 意味 | リスク |
|---------|------|--------|
| sudo | 管理者権限を使える | 高（意図的） |
| docker | Dockerを操作できる | 高（実質root） |
| adm | ログを読める | 中 |
| www-data | Webサーバー用 | 低 |

### dockerグループの危険性

```bash
# dockerグループのユーザーが実行可能
docker run -v /:/host -it ubuntu bash
# → ホスト全体にアクセス可能 = 実質root
```

## 関連コマンド

- [[whoami]] - ユーザー名のみ
- [[groups]] - グループ一覧
