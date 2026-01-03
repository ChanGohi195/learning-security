---
command: chmod
category: permission
tags: [linux, security, permission, file]
---

# chmod

ファイル・ディレクトリの権限を変更。

## 基本

```bash
$ chmod 755 script.sh      # 数値指定
$ chmod u+x script.sh      # 所有者に実行権限を追加
$ chmod go-w config.txt    # グループと他から書き込み権限を削除
$ chmod -R 755 /path/dir   # 再帰的に変更
```

## 権限の読み方

```
-rwxr-xr--
 ↑↑↑↑↑↑↑↑↑
 │└┬┘└┬┘└┬┘
 │ │  │  └── other（その他）: r-- = 4
 │ │  └───── group（グループ）: r-x = 5
 │ └──────── owner（所有者）: rwx = 7
 └────────── ファイル種別
```

## 数値の計算

| 権限 | 2進数 | 10進数 |
|------|-------|--------|
| r (read) | 100 | 4 |
| w (write) | 010 | 2 |
| x (execute) | 001 | 1 |

```
rwx = 4 + 2 + 1 = 7
r-x = 4 + 0 + 1 = 5
r-- = 4 + 0 + 0 = 4
```

## よく使う権限

| 数値 | 権限 | 用途 |
|------|------|------|
| 755 | rwxr-xr-x | 実行ファイル、公開ディレクトリ |
| 644 | rw-r--r-- | 通常ファイル |
| 600 | rw------- | 秘密鍵、パスワードファイル |
| 700 | rwx------ | 個人用ディレクトリ |

## セキュリティ観点

### 絶対に使うべきでない

```bash
$ chmod 777 file    # 全員に全権限 = 危険
```

### SSH秘密鍵

```bash
$ chmod 600 ~/.ssh/id_rsa      # 必須（SSHが権限チェックする）
$ chmod 644 ~/.ssh/id_rsa.pub  # 公開鍵は読まれてOK
$ chmod 700 ~/.ssh             # ディレクトリ
```

権限が緩いと接続拒否される：
```
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 are too open.
```

## 関連コマンド

- [[ls]] - 権限を確認（ls -la）
- [[chown]] - 所有者を変更
