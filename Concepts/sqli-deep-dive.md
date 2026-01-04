---
topic: SQL Injection Deep Dive
category: web-security
tags: [security, web, sqli, owasp, deep-dive]
created: 2026-01-03
---

# SQLインジェクション深掘り

OWASP Top 10の概要を超えた、攻撃者視点での理解。

## 根本原因

**データ（ユーザー入力）とコード（SQL文）の混在。**

```python
# 脆弱なコード
query = "SELECT * FROM users WHERE name = '" + username + "'"
```

攻撃者は「データとして送ったもの」を「コードとして実行させる」。

## 攻撃パターン1: ログインバイパス

### 正常なログイン

```
入力: username=admin, password=secret123

SQL: SELECT * FROM users WHERE name = 'admin' AND pass = 'secret123'

結果: パスワードが正しければログイン成功
```

### 攻撃: パスワードチェックの無効化

```
入力: username=admin'--, password=anything

SQL: SELECT * FROM users WHERE name = 'admin'--' AND pass = 'anything'
                                             ↑
                                    ここ以降はコメント（無視）

実質: SELECT * FROM users WHERE name = 'admin'

結果: パスワードなしでadminとしてログイン成功
```

### 攻撃: 任意のユーザーでログイン

```
入力: username=' OR '1'='1'--, password=anything

SQL: SELECT * FROM users WHERE name = '' OR '1'='1'--...

結果: '1'='1'は常に真 → 全ユーザーが返る → 最初のユーザー（通常admin）でログイン
```

## キーとなる文字の役割

| 文字 | 役割 |
|------|------|
| `'` | 文字列を閉じて「SQLの世界」に出る |
| `--` | 残りのSQLをコメント化（パスワードチェック等を無効化） |
| `OR '1'='1'` | 常に真の条件を挿入 |

### 自分の理解

> 最初「`'`は構文エラー用」と思ったが、正確には「文字列を早めに閉じて、攻撃者がSQLコードを書ける状態にする」ため。エラーを起こすのではなく、SQLの文脈に「脱出」するのが目的。

## 攻撃パターン2: UNION攻撃（データ窃取）

ログインバイパスは「認証突破」。より深刻なのは**データベースの中身を盗む**こと。

### シナリオ: 商品検索機能

```
正常: 検索ワード「apple」

SQL: SELECT name, price FROM products WHERE name LIKE '%apple%'
```

### 攻撃: ユーザーテーブルの内容を表示

```
入力: ' UNION SELECT username, password FROM users--

SQL: SELECT name, price FROM products WHERE name LIKE '%'
     UNION SELECT username, password FROM users--'

結果: 商品リストに混じって、全ユーザーのID・パスワードが表示される
```

### UNIONの制約

1. **カラム数が一致する必要がある**
2. **データ型が互換性を持つ必要がある**

攻撃者はカラム数を知らないので、試行錯誤で特定:

```sql
' UNION SELECT NULL--              -- 1カラム？エラー
' UNION SELECT NULL, NULL--        -- 2カラム？エラー
' UNION SELECT NULL, NULL, NULL--  -- 3カラム？成功！
```

### 自分の思考

> 「UNIONが成功する条件は？」と問われ、「データ種類ごとの分離」「権限範囲の重なり」と答えた。セキュリティ的には正しい発想だが、技術的制約は「カラム数とデータ型の一致」だった。設計思想と実装制約は別レイヤー。

## 攻撃フロー（全体像）

```
1. SQLi可能か確認
   → ' を入れてエラー観察

2. カラム数を特定
   → UNION SELECT NULL, NULL... で試行

3. 表示されるカラムを特定
   → NULLを1,2,3に置き換え、画面に出る位置を探す

4. データベース情報を取得
   → テーブル名、カラム名を抽出

5. 目的のデータを抽出
   → users, passwords等
```

### 検知ポイント

> ステップ2-3で試行回数が必要 → アクティビティレベルが上がる → ログ監視で検知可能

## 防御の多層構造

### 1. 根本対策: パラメータ化クエリ

```python
# 安全なコード
query = "SELECT * FROM users WHERE name = ?"
cursor.execute(query, (username,))
```

データとコードが**構造的に分離**される。何を入れても絶対にコードにならない。

### 2. 多層防御: WAF

既知の攻撃パターンをブロック:
- `' OR '1'='1`
- `UNION SELECT`
- `--`（コメント）

### 3. 攻撃者のバイパス手法

| 元のペイロード | バイパス例 |
|---------------|-----------|
| `UNION SELECT` | `UnIoN SeLeCt`（大文字小文字混合） |
| `UNION SELECT` | `UNION/**/SELECT`（コメント挿入） |
| `' OR '1'='1` | `' OR '2'='2`（パターン変更） |

**いたちごっこ。だからWAFは補助、パラメータ化が根本。**

### 4. 監視と検知

| 兆候 | 意味 |
|------|------|
| 同一IPから大量のリクエスト | 試行錯誤中 |
| SQLエラーの急増 | 構文を探っている |
| `UNION`, `SELECT`, `--` を含むリクエスト | SQLi試行 |

## まとめ

| レベル | 内容 |
|--------|------|
| 原因 | データとコードの混在 |
| 攻撃 | ログインバイパス、UNION攻撃 |
| フロー | 検出 → カラム特定 → データ抽出 |
| 防御 | パラメータ化（根本）+ WAF（多層）+ 監視（検知） |

## 関連

- [[owasp-top-10]] - 全体像
- [[xss-deep-dive]] - 次の深掘り対象
- [[authentication]] - 認証の脆弱性
