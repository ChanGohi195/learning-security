# Security Jargon Index

セキュリティ学習で出てきた専門用語の索引。カテゴリ別に整理し、関連概念へのリンクを含む。

---

## Web Security

| 用語 | 説明 | 関連 |
|------|------|------|
| SQLi (SQL Injection) | SQLクエリにコード注入 | parameterized query |
| XSS (Cross-Site Scripting) | 悪意あるスクリプト注入。Stored/Reflected/DOM-based | CSP, HttpOnly |
| CSRF (Cross-Site Request Forgery) | 偽造リクエスト送信 | SameSite Cookie |
| SSRF (Server-Side Request Forgery) | サーバーから内部リクエスト強制 | メタデータサービス |
| IDOR (Insecure Direct Object Reference) | オブジェクト参照の認可不備 | 認可 vs 認証 |
| CSP (Content Security Policy) | スクリプト実行元制限 | XSS対策 |
| OWASP | Webセキュリティ標準化団体 | Top 10 |
| WAF (Web Application Firewall) | Webアプリ用ファイアウォール | シグネチャ検知 |

---

## 認証・認可

| 用語 | 説明 | 関連 |
|------|------|------|
| Authentication（認証） | 「誰か」を確認 | 認可との違い |
| Authorization（認可） | 「何ができるか」を確認 | RBAC |
| Session | サーバー側で状態保持 | Cookie, SessionID |
| JWT (JSON Web Token) | 署名付きトークン。Header.Payload.Signature | alg:none攻撃 |
| OAuth | 認可委譲プロトコル | アクセストークン |
| MFA (Multi-Factor Authentication) | 多要素認証 | TOTP |
| Cookie属性 | HttpOnly, Secure, SameSite | XSS/CSRF対策 |

---

## 暗号

| 用語 | 説明 | 関連 |
|------|------|------|
| 対称暗号 | 同一鍵で暗号/復号（AES） | 鍵配送問題 |
| 非対称暗号 | 公開鍵/秘密鍵（RSA） | デジタル署名 |
| ハイブリッド暗号 | 対称+非対称の組み合わせ | TLS |
| Hash | 一方向変換（SHA-256） | パスワード保存 |
| Salt | Hashに追加するランダム値 | レインボーテーブル対策 |
| base64 | エンコーディング（暗号ではない） | 可読性変換 |

---

## Linux Kernel Security

| 用語 | 説明 | 関連 |
|------|------|------|
| Namespace | リソースの可視性分離（PID, Mount, Network等） | コンテナ |
| cgroups | リソース使用量制限（CPU, メモリ） | DoS対策 |
| Capabilities | 特権操作の細分化 | 最小権限 |
| seccomp | システムコールフィルタ | ホワイトリスト |
| SELinux / AppArmor | 強制アクセス制御（MAC） | ポリシーベース |
| ASLR | アドレス空間配置のランダム化 | メモリ攻撃対策 |
| eBPF | カーネル内プログラム実行 | 監視、ネットワーク |
| netfilter / iptables | カーネルレベルパケットフィルタ | ファイアウォール |
| Kernel Module | カーネル機能拡張 | rootkit |
| /proc, /sys | カーネル情報の仮想ファイルシステム | 情報漏洩リスク |

---

## コンテナ

| 用語 | 説明 | 関連 |
|------|------|------|
| Container | 隔離されたプロセス実行環境 | VM vs Container |
| Image | コンテナのテンプレート | レイヤー構造 |
| Registry | イメージ保管場所 | Docker Hub |
| Docker | コンテナランタイム | daemon, containerd, runc |
| Rootless Docker | root権限なしでDocker実行 | User Namespace |
| Docker Content Trust | イメージ署名検証 | Notary |
| Cosign | OCI準拠のイメージ署名 | Sigstore |
| Supply Chain Attack | 依存関係への攻撃 | SolarWinds |

---

## Kubernetes

| 用語 | 説明 | 関連 |
|------|------|------|
| Pod | 最小デプロイ単位（1+コンテナ） | Node |
| Node | Podを実行するマシン | Worker |
| Control Plane | クラスタ管理層 | API Server, etcd |
| API Server | 全操作の入口 | 認証・認可 |
| etcd | 設定・状態保存DB | 暗号化必須 |
| RBAC | Role-Based Access Control | Role, ClusterRole |
| ServiceAccount | Pod用の認証手段 | Token自動マウント |
| Pod Security Standards | Privileged/Baseline/Restricted | Admission Control |
| Network Policy | Pod間通信制御 | マイクロセグメンテーション |
| Admission Control | 操作の最終検証 | Webhook |

---

## クラウドセキュリティ

| 用語 | 説明 | 関連 |
|------|------|------|
| Shared Responsibility Model | クラウド/顧客の責任分界 | マンション比喩 |
| IAM | Identity and Access Management | 最小権限 |
| Metadata Service | 169.254.169.254 | SSRF経由の攻撃 |
| IMDSv2 | メタデータサービスv2（トークン必須） | AWS |
| Firecracker | microVM | AWS Lambda |
| gVisor | ユーザー空間カーネル | GCP |
| Confidential Computing | ハードウェア暗号化 | TEE |

---

## 攻撃者・脅威

| 用語 | 説明 | 関連 |
|------|------|------|
| TeamTNT | Docker/K8s標的のマイニンググループ | ドイツ拠点 |
| Kinsing | コンテナ環境自動スキャン | マイニング |
| WatchDog | K8sクラスタ標的 | 中国拠点 |
| Cryptojacking | 計算資源でマイニング | 検知困難 |
| Lateral Movement | 横展開 | 初期侵入後 |
| Persistence | 永続化 | バックドア |

---

## 設計原則

| 用語 | 説明 | 関連 |
|------|------|------|
| Defense in Depth | 多層防御 | 1つ突破されても次で止める |
| Least Privilege | 最小権限の原則 | 必要最小限のみ |
| Zero Trust | 信頼しない前提 | Assume Breach |
| Separation | 分離の原則 | 脆弱性の根本原因 |
| Blast Radius | 被害範囲 | 局所化 |

---

## 学習中の名言

> 「分けるべきものを分けていないことに起因する脆弱性」

> 「鍵なしで入れて、机に金庫の鍵が金庫番号ごとに整理されている」
> （Kubernetesのデフォルト設定への比喩）

> 「Linux偉大すぎる。カーネルのデザインがクラウドサービスの根底にある」

---

## 更新履歴

- 2026-01-06: 初版作成（Phase 3完了時点）
  - Web Security, 認証・認可, 暗号
  - Linux Kernel Security
  - コンテナ, Kubernetes
  - クラウドセキュリティ
  - 攻撃者・脅威
  - 設計原則

---

#security #jargon #index #reference
