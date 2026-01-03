---
command: tail
category: log
tags: [linux, security, log, monitoring]
---

# tail

ファイルの末尾を表示。ログ監視に必須。

## 基本

```bash
$ tail file.txt              # 末尾10行
$ tail -n 50 file.txt        # 末尾50行
$ tail -f /var/log/syslog    # リアルタイム監視
$ tail -F /var/log/syslog    # ローテーション対応
```

## セキュリティ用途

### リアルタイム監視

```bash
# 認証ログを監視
$ tail -f /var/log/auth.log

# 複数ログを同時監視
$ tail -f /var/log/auth.log /var/log/syslog

# フィルタリングしながら監視
$ tail -f /var/log/auth.log | grep "Failed"
```

### 主要ログファイル

| ファイル | 内容 |
|---------|------|
| /var/log/auth.log | 認証（sudo、SSH等） |
| /var/log/syslog | システム全般 |
| /var/log/kern.log | カーネル |
| /var/log/apache2/access.log | Apache アクセス |
| /var/log/nginx/access.log | Nginx アクセス |

### インシデント対応時

```bash
# 直近のログを確認
$ tail -n 100 /var/log/auth.log

# 特定時刻以降を監視
$ tail -f /var/log/auth.log | grep --line-buffered "Failed"
```

## -f vs -F

| オプション | 動作 |
|-----------|------|
| -f | ファイルを追跡（ローテーションで止まる） |
| -F | ファイル名を追跡（ローテーション後も継続） |

長時間監視には `-F` を使う。

## 関連コマンド

- [[head]] - ファイル先頭
- [[grep]] - 検索
- [[less]] - ページング表示
