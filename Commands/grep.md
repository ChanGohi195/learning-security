---
command: grep
category: search
tags: [linux, security, log, audit]
---

# grep

ファイル内容を検索。ログ分析で必須。

## 基本

```bash
$ grep "pattern" file.txt
$ grep -r "pattern" /path/          # 再帰検索
$ grep -i "pattern" file.txt        # 大文字小文字無視
$ grep -v "pattern" file.txt        # 不一致行
$ grep -n "pattern" file.txt        # 行番号表示
$ grep -c "pattern" file.txt        # マッチ数
```

## セキュリティ用途

### 認証失敗の検索

```bash
$ grep "Failed password" /var/log/auth.log
$ grep "authentication failure" /var/log/auth.log
```

### sudo使用の追跡

```bash
$ grep "sudo" /var/log/auth.log
$ grep "COMMAND=" /var/log/auth.log
```

### SSH接続の確認

```bash
$ grep "Accepted" /var/log/auth.log      # 成功
$ grep "Failed" /var/log/auth.log        # 失敗
$ grep "Invalid user" /var/log/auth.log  # 存在しないユーザー
```

### 特定IPの活動

```bash
$ grep "192.168.1.100" /var/log/auth.log
$ grep "192.168.1.100" /var/log/nginx/access.log
```

### 設定ファイルの確認

```bash
$ grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"
# コメントと空行を除外して有効な設定のみ表示
```

## パイプとの組み合わせ

```bash
$ cat /var/log/auth.log | grep "Failed" | wc -l
# 失敗回数をカウント

$ grep "Failed" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
# 失敗したIPをカウントしてランキング
```

## 関連コマンド

- [[find]] - ファイル検索
- [[tail]] - ログ監視
- [[awk]] - テキスト処理
