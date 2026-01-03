---
command: whoami
category: user-identity
tags: [linux, security, user]
---

# whoami

現在のユーザー名を表示。

## 基本

```bash
$ whoami
user
```

## セキュリティ観点

- スクリプト実行前に「誰として動いているか」を確認
- rootで動いていないかのチェックに使用
- 権限エラー時の原因切り分け

## 関連コマンド

- [[id]] - より詳細なユーザー情報
- [[sudo]] - 権限昇格
