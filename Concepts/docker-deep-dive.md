---
topic: Docker Deep Dive
category: cloud-security
tags: [security, docker, container, rootless, network, deep-dive]
created: 2026-01-04
---

# Docker 深掘り

Dockerのアーキテクチャとセキュリティを理解する。

## Dockerの構成要素

```
Docker CLI（docker run, build...）
       ↓ API
Docker Daemon (dockerd) ← 常駐プロセス
       ↓
containerd ← コンテナランタイム
       ↓
runc ← 実際にコンテナを起動
       ↓
Linux Kernel（namespace, cgroups, seccomp）
```

## Docker Daemonの問題

### 従来の設計

```
Docker Daemon = root権限で動作
```

dockerコマンドを実行できる = rootと同等の権限

```bash
# dockerグループに所属していれば
docker run -v /etc:/host-etc ubuntu cat /host-etc/shadow
# → ホストの/etc/shadowが読める
```

### 自分の気づき

> マンションの各部屋に全館のブレーカー操作盤を置いていた、みたいなアーキテクチャ。
> 1部屋（コンテナ）が侵害されると全館（ホスト）が危険。
> やばい設計が長年続いていた。

### なぜrootが必要だったか

```
コンテナの分離に必要なLinux機能:
  - namespace（プロセス、ネットワーク等を分離）
  - cgroups（リソース制限）
  - ネットワーク設定（仮想ブリッジ作成）

これらの操作 → 従来はroot権限が必須
```

**「分離するためにroot」という矛盾。**

## Rootless Docker

### 解決策

```
User Namespace = 「自分だけの仮想root空間」
       ↓
その空間内では「root」として振る舞える
       ↓
実際のシステムには影響しない
```

### 構造の違い

```
従来:
┌─────────────────────────────┐
│ ホスト（実際のroot）         │
│   └─ Docker Daemon (root)   │
│        └─ コンテナ           │
└─────────────────────────────┘
  → Daemonが侵害されるとホスト全体が危険

Rootless:
┌─────────────────────────────┐
│ ホスト                       │
│   └─ User Namespace          │
│        └─ Docker Daemon      │ ← この空間内でのみroot
│             └─ コンテナ      │
└─────────────────────────────┘
  → 侵害されてもUser Namespace内に閉じ込められる
```

### 自分の理解

> 権限の範囲を制限することで必要な権限のみ与えることができるようになった。
> 非常にスマートな問題解決。
> 最小権限の原則がカーネルレベルで実現された。

### 比較

| 項目 | 従来 (root) | Rootless |
|------|-------------|----------|
| Daemon権限 | root | 一般ユーザー |
| 侵害時の影響 | ホスト全体 | そのユーザーのみ |
| 機能 | フル | 一部制限あり |
| 設定難易度 | 低 | やや高 |

## Dockerネットワーク

### 3つのモード

#### bridge（デフォルト）

```
ホスト
  eth0 (192.168.1.100)
    │
  docker0 (172.17.0.1) ← 仮想ブリッジ
    ├── コンテナA (172.17.0.2)
    └── コンテナB (172.17.0.3)
```

- コンテナ同士は通信可能
- 外部へはNAT経由
- 分離されつつ、必要な通信は可能

#### host

```
ホスト
  eth0 (192.168.1.100)
    │
  コンテナ（ホストと同じネットワーク）
```

- ネットワーク分離なし
- ホストのネットワークをそのまま使う
- **高速だが危険**

#### none

```
コンテナ（ネットワークなし）
  → localhostのみ
```

- 完全に隔離
- 外部通信不可
- 最も安全、用途限定

### セキュリティ比較

| モード | 分離度 | リスク | 用途 |
|--------|--------|--------|------|
| bridge | 中 | 中 | 一般的な用途 |
| host | なし | 高 | パフォーマンス重視 |
| none | 完全 | 低 | バッチ処理、計算のみ |

### --net=host の危険性

```
bridge:
  コンテナ侵入 → コンテナ内のみ
  ホストへは別途脱獄が必要

host:
  コンテナ侵入 → ホストのネットワークに直接アクセス
  他サービス（DB等）への攻撃が容易
```

**自分の脆弱性がホストへの侵入口になる。**

### カスタムネットワーク（推奨）

```yaml
services:
  web:
    networks:
      - frontend
  api:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend  # webから直接アクセス不可

networks:
  frontend:
  backend:
```

```
web ←→ api ←→ db
web ─×→ db（直接アクセス不可）
```

**ネットワークレベルでBlast Radiusを制限。**

## Dockerセキュリティ設定まとめ

| 項目 | デフォルト | 推奨 |
|------|-----------|------|
| Daemon | root | Rootless |
| ユーザー | root | 非root (USER) |
| ネットワーク | bridge | カスタム分離 |
| マウント | 任意 | 必要最小限、読み取り専用 |
| 権限 | 全capabilities | 必要なもののみ |

## 歴史的教訓

> 「動けばいい」から始まり、セキュリティは後付け。
> これはIT全般に共通する負債。

Dockerも例外ではなく、root前提の設計が長年続いた。
Rootless Dockerの登場により、ようやく最小権限の原則が実現。

## 関連

- [[container-deep-dive]] - コンテナセキュリティ全般
- [[sqli-deep-dive]] - SQLインジェクション
- [[authentication-deep-dive]] - 認証
- [[kubernetes-security]] - 次のトピック
