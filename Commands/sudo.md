---
command: sudo
category: privilege
tags: [linux, security, privilege-escalation, audit]
---

# sudo

一時的にroot権限（または他ユーザー権限）で実行。

## 基本

```bash
$ sudo apt update              # rootとして実行
$ sudo -u postgres psql        # postgresユーザーとして実行
$ sudo -i                      # rootシェルに切り替え
$ sudo -l                      # 自分が実行可能なコマンドを表示
```

## セキュリティ観点

### なぜrootログインではなくsudoか

| 観点 | rootログイン | sudo |
|------|-------------|------|
| 権限の範囲 | 常に全権限 | 必要なときだけ昇格 |
| 監査 | 「root」としか記録されない | 「userがsudoした」と記録 |
| ミスの影響 | 常に致命的 | 通常操作は安全 |

### 監査ログ

```bash
# /var/log/auth.log に記録される
Jan 2 12:00:00 host sudo: user : TTY=pts/0 ; PWD=/home/user ; USER=root ; COMMAND=/usr/bin/apt update
```

### 最小権限の原則

```
必要な権限を、必要なときだけ、必要な範囲で
```

## 設定ファイル

```bash
$ sudo visudo    # /etc/sudoers を安全に編集
```

```
# 特定コマンドのみ許可
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

## 関連コマンド

- [[su]] - ユーザー切り替え（非推奨）
- [[id]] - 現在の権限確認
