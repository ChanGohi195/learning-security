---
command: ps
category: process
tags: [linux, security, process]
---

# ps

実行中プロセスを表示。

## 基本

```bash
$ ps              # 現在のシェルのプロセス
$ ps aux          # 全プロセス（BSD形式）
$ ps -ef          # 全プロセス（System V形式）
$ ps aux | grep nginx
```

## 出力の読み方

```
USER   PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
root     1  0.0  0.1 169584 13236 ?    Ss   Jan01   0:05 /sbin/init
www-data 1234 0.1 1.2 123456 12345 ?   S    10:00   0:30 nginx: worker
```

| 列 | 意味 |
|----|------|
| USER | 実行ユーザー |
| PID | プロセスID |
| %CPU | CPU使用率 |
| %MEM | メモリ使用率 |
| STAT | 状態 |
| COMMAND | コマンド |

## セキュリティ観点

### 確認すべきこと

```bash
# rootで動いているプロセス
$ ps aux | grep "^root"

# 不審なプロセス名
$ ps aux | grep -E "(nc|ncat|netcat|/tmp/)"

# 高CPU/メモリのプロセス（マイニング等）
$ ps aux --sort=-%cpu | head
```

### 不審なパターン

- `/tmp/` や `/dev/shm/` から実行されている
- 意味不明な名前（ランダム文字列）
- rootで動いているが不要なプロセス

## 関連コマンド

- [[top]] / [[htop]] - リアルタイム監視
- [[ss]] - ネットワーク接続
- [[lsof]] - 開いているファイル
