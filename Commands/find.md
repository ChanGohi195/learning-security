---
command: find
category: file-search
tags: [linux, security, audit, suid]
---

# find

ファイルを検索。セキュリティ監査で頻用。

## 基本

```bash
$ find /path -name "*.txt"           # 名前で検索
$ find /path -type f                 # ファイルのみ
$ find /path -type d                 # ディレクトリのみ
$ find /path -mtime -7               # 7日以内に変更
$ find /path -size +100M             # 100MB以上
```

## セキュリティ監査用

### SUID/SGID ファイル検索

```bash
$ find / -perm -4000 2>/dev/null     # SUID
$ find / -perm -2000 2>/dev/null     # SGID
$ find / -perm -6000 2>/dev/null     # 両方
```

**なぜ重要か**: SUID ファイルは所有者の権限で実行される。rootが所有するSUIDファイルの脆弱性 = 権限昇格。

### 誰でも書けるファイル

```bash
$ find / -perm -o+w -type f 2>/dev/null
$ find /home -name "*.sh" -perm -o+w    # スクリプト限定
```

**なぜ危険か**: 実行されるスクリプトが書き換え可能 = コード注入。

### 所有者不明ファイル

```bash
$ find / -nouser 2>/dev/null
$ find / -nogroup 2>/dev/null
```

**なぜ重要か**: 削除されたユーザーの痕跡、または不正ファイル。

### 最近変更されたファイル

```bash
$ find / -mtime -1 -type f 2>/dev/null   # 24時間以内
$ find /etc -mtime -1                     # 設定ファイルの変更
```

**インシデント調査時に有用**。

### 隠しファイル

```bash
$ find /tmp -name ".*" -type f
$ find /var/tmp -name ".*" -type f
```

## 関連コマンド

- [[ls]] - ファイル一覧
- [[chmod]] - 権限変更
- [[grep]] - ファイル内容検索
