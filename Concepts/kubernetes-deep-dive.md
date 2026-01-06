---
topic: Kubernetes Security Deep Dive
category: cloud-security
tags: [security, kubernetes, k8s, container, orchestration, rbac, network-policy, deep-dive]
created: 2026-01-06
---

# Kubernetes セキュリティ深掘り

コンテナオーケストレーションプラットフォームの防御と攻撃を理解する。

## Kubernetesとは

### Docker vs Kubernetes

```
Docker = 1部屋の管理
  コンテナを起動、停止、ネットワーク接続

Kubernetes = マンション全体の管理
  複数のコンテナを複数のサーバーで動かす
  死んだら自動で再起動
  負荷に応じて増減
  どこに配置するか決定
```

**Dockerの上に立つオーケストレーター。**

### 構造

```
┌─────────────────────────────────────────────────┐
│ Control Plane（脳）                              │
│  ┌──────────────┐                               │
│  │ API Server   │ ← 全操作の入口                │
│  ├──────────────┤                               │
│  │ etcd         │ ← 全設定・状態を保存          │
│  ├──────────────┤                               │
│  │ Scheduler    │ ← どこに配置するか決定        │
│  ├──────────────┤                               │
│  │ Controller   │ ← 望ましい状態を維持          │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘
             │
             ↓ 指示
┌─────────────────────────────────────────────────┐
│ Node（手足）                                     │
│  ┌──────────────┐                               │
│  │ kubelet      │ ← Control Planeと通信         │
│  ├──────────────┤                               │
│  │ kube-proxy   │ ← ネットワーク設定            │
│  ├──────────────┤                               │
│  │ Pod          │ ← 実際のコンテナ              │
│  │  └─Container │                               │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘
```

## Control Planeの重要性

### etcdが全て

```
etcd = クラスター全体の設定・状態・秘密情報

内容:
  - どのPodがどこにあるか
  - 各Podの設定
  - Secrets（パスワード、APIキー）
  - 誰にどの権限があるか
```

**etcdを取られる = クラスター全体を掌握される。**

### 攻撃者の視点

```
標的の優先順位:

1位: etcd
   → 全Secretsを取得
   → 全ノードに任意のコンテナをデプロイ

2位: API Server
   → etcdへのゲートウェイ
   → 全操作の通り道

3位: 個別のPod
   → 横展開の起点
   → 権限昇格の足がかり
```

**防御は必然的にControl Plane保護が最優先。**

## API Serverの3層防御

```
リクエスト
    ↓
┌─────────────────────────────────────┐
│ 1. Authentication（認証）            │
│    「お前は誰か？」                  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. Authorization（認可）             │
│    「お前は何ができるか？」          │
│    → RBAC                           │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 3. Admission Control                │
│    「そのリクエストは許可できるか？」│
│    → Pod Security Standards         │
└─────────────────────────────────────┘
    ↓
etcd
```

### Authentication（認証）

| 方式 | 説明 |
|------|------|
| ServiceAccount | Pod用の認証（Tokenベース） |
| X.509証明書 | 人間・管理ツール用 |
| OIDC | 外部IdP連携 |

**匿名アクセスは原則無効化。**

### Authorization（認可）= RBAC

```yaml
# Role: 名前空間内の権限定義
kind: Role
metadata:
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]

# RoleBinding: ユーザー/SAとRoleを紐づけ
kind: RoleBinding
subjects:
  - kind: ServiceAccount
    name: my-app
roleRef:
  kind: Role
  name: pod-reader
```

### スコープの違い

| 種類 | スコープ | 用途 |
|------|----------|------|
| Role / RoleBinding | 単一Namespace内 | アプリケーション権限 |
| ClusterRole / ClusterRoleBinding | Cluster全体 | 管理者、インフラ権限 |

### 自分の気づき

> 鍵なしで入れて、机に金庫の鍵が金庫番号ごとに整理されておいてある。
> デフォルトのServiceAccountは全Podが同じTokenを持つから、1つ侵入すればパターン共通化。

### 危険な設定例

```yaml
# やってはいけない
subjects:
  - kind: Group
    name: system:authenticated  # 認証済み全員
roleRef:
  kind: ClusterRole
  name: cluster-admin  # 全権限
```

**認証されていれば誰でもCluster全体を操作可能 = 事実上認証なし。**

## Admission Control

### Pod Security Standards

| レベル | 制約 |
|--------|------|
| Privileged | 無制限（何でも可能） |
| Baseline | 明らかに危険な設定を禁止 |
| Restricted | 最小権限、ベストプラクティス強制 |

### Restrictedの制約

```yaml
# 禁止事項:
  - hostPath マウント（ホストのディレクトリを見せない）
  - privileged: true（root権限を与えない）
  - hostNetwork: true（ホストのネットワークを共有しない）
  - runAsNonRoot: false（rootユーザーで動かさない）
```

### なぜ全てRestrictedにできないか

```
システムコンポーネント（kube-proxy、CNI等）:
  ホストのネットワーク設定変更が必要
  → hostNetwork: true が必須

ログ収集（Fluentd等）:
  全コンテナのログを読む必要
  → hostPath マウントが必須
```

**現実には多層構造が必要。**

| 対象 | レベル |
|------|--------|
| システムコンポーネント | Privileged（厳重に監視） |
| 信頼されたミドルウェア | Baseline |
| アプリケーション | Restricted |

## ServiceAccount

### デフォルトSAの危険性

```yaml
# 何も指定しないと
spec:
  serviceAccountName: default  # 自動設定
  automountServiceAccountToken: true  # 自動マウント
```

```bash
# コンテナ内から
cat /var/run/secrets/kubernetes.io/serviceaccount/token
# → API Serverにアクセス可能なToken
```

**全Podが同じTokenを持つ = 1つ侵害されれば攻撃パターンが全Podに適用可能。**

### 推奨設定

```yaml
# 1. 専用SAを作成
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa

---
# 2. Podに指定
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false  # 不要なら無効化
```

### RBAC連携

```yaml
# 3. 最小権限をRoleで定義
kind: Role
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]  # 読み取り専用

---
# 4. SAとRoleを紐づけ
kind: RoleBinding
subjects:
  - kind: ServiceAccount
    name: my-app-sa
roleRef:
  kind: Role
  name: configmap-reader
```

**ServiceAccount単位でBlast Radiusを限定。**

## Network Policy

### デフォルトの危険性

```
Podは全てbridge的なネットワークに接続
  → デフォルトで全Pod間通信可能
```

```
攻撃者 → WebサーバーのPodに侵入
       → 直接DBのPodに攻撃可能
```

**横展開経路が無制限。**

### Network Policyで分離

```yaml
# DBは特定Podからのみアクセス許可
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: backend  # backendからのみ許可
      ports:
        - protocol: TCP
          port: 5432
```

### マイクロセグメンテーション

```
従来（全通信可能）:
  web ←→ api ←→ db
  web ←→ db  # 直接アクセス可能

Network Policy適用後:
  web → api → db
  web → db  # 拒否
```

**侵害されたPodからの被害を局所化。**

### 自分の理解

> ゼロトラストの具体化。
> 「同じクラスターにいる = 信頼」ではなく、明示的な許可のみ。

## Secrets管理

### base64の誤解

```yaml
# Secret定義
apiVersion: v1
kind: Secret
data:
  password: cGFzc3dvcmQxMjM=  # base64エンコード
```

```bash
echo "cGFzc3dvcmQxMjM=" | base64 -d
# → password123

# 暗号化ではない、単なるエンコーディング
```

**etcdにそのまま保存される = etcd侵害でSecrets全流出。**

### etcd暗号化（Encryption at Rest）

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <32-byte-key>
```

**etcdのディスクが盗まれてもSecretsは読めない。**

### 外部Secret管理（推奨）

```
クラスター内:
  SecretはKubernetesに保存しない

外部システム:
  - HashiCorp Vault
  - AWS Secrets Manager
  - Azure Key Vault
  - GCP Secret Manager

アクセス時:
  Podが起動時に外部から取得
```

| 方式 | etcd侵害時の被害 |
|------|------------------|
| k8s Secrets（暗号化なし） | 全Secrets流出 |
| k8s Secrets（etcd暗号化） | 鍵も盗まれたら流出 |
| 外部Secret管理 | Kubernetes侵害では流出しない |

### RBAC制限

```yaml
# Secretsへのアクセスを厳密に制限
kind: Role
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["my-app-secret"]  # 特定Secretのみ
```

## 攻撃シナリオ: 5段階の侵入

### Stage 1: 初期侵入

```
攻撃対象:
  脆弱なWebアプリケーション（SQLi、XSS、RCE）

結果:
  1つのPod内でコード実行可能
```

### Stage 2: 情報収集

```bash
# コンテナ内から
cat /var/run/secrets/kubernetes.io/serviceaccount/token
# → ServiceAccount Token取得

env | grep KUBERNETES
# → KUBERNETES_SERVICE_HOST、KUBERNETES_SERVICE_PORT
# → API Serverの場所がわかる
```

**PodはデフォルトでAPI Serverの場所を知っている。**

### Stage 3: 権限確認

```bash
# Tokenを使ってAPI Serverに問い合わせ
kubectl auth can-i --list

# 出力例（危険な設定の場合）:
Resources   Verbs
*.*         *      # 全リソースに全操作可能
```

**この時点でクラスター全体を掌握可能。**

### Stage 4: 横展開 or 権限昇格

```bash
# パターンA: 横展開
kubectl get pods --all-namespaces
# → 他のPodを発見
# → 同じ攻撃パターンを適用（defaultSA共通利用の場合）

# パターンB: 権限昇格
kubectl get secrets --all-namespaces
# → DB認証情報、外部APIキーを取得

# パターンC: 永続化
kubectl create -f malicious-pod.yaml
# → 自分のバックドアPodをデプロイ
```

### Stage 5: 目的達成

```
- クリプトマイニング（計算資源を盗む）
- データ窃取（Secretsから認証情報）
- ランサムウェア（etcdを暗号化）
- 踏み台（他システムへの攻撃）
```

## 現実の攻撃と攻撃者

### なぜクリプトマイニングか

```
攻撃者の動機:

1. 計算資源が潤沢
   クラウドのCPUは強力
   複数のクラスターを侵害すれば大規模なマイニングファーム

2. 検知されにくい
   CPU使用率が高いだけ
   「負荷が高いだけ」と見逃されやすい
   データ窃取やランサムウェアより静か

3. 即座に換金可能
   盗んだデータは売却が必要
   マイニングは仮想通貨に直接変換
```

**クラウド = 無料の計算資源として狙われる。**

### 攻撃者グループ（判明している事例）

| グループ | 特徴 |
|----------|------|
| TeamTNT | Docker/Kubernetes標的のマイニング特化（ドイツ拠点とされる） |
| Kinsing | 設定ミスのコンテナ環境を自動スキャン |
| WatchDog | Kubernetesクラスター標的（中国拠点とされる） |

### 実際のインシデント

#### Tesla (2018)

```
脆弱性:
  Kubernetesダッシュボードを認証なしで公開

攻撃:
  1. ダッシュボードから全Podを発見
  2. AWS認証情報を含むPodを特定
  3. S3バケットへアクセス
  4. マイニングPodをデプロイ
```

**認証なし = 鍵なしで金庫を公開。**

#### Azure (2021)

```
脆弱性:
  MLインフラのKubernetesクラスターが標的

攻撃:
  権限昇格 → cluster-admin取得
  → 全ノードにマイニングコンテナをデプロイ
```

#### 2022年事例

```
組み合わせ攻撃:
  Dashboard無認証公開
    +
  全Podに cluster-admin 権限
    +
  Secretsに他システムのAPIキー

結果:
  クラスター侵害 → 外部システムへの横展開
```

**単一の防御では不十分、多層防御が必須。**

## Defense in Depth（多層防御）

```
┌─────────────────────────────────────────┐
│ Control Plane保護                        │
│  API Server: 認証 → 認可(RBAC) → Admission│
│  etcd: 暗号化、アクセス制限              │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Workload保護                            │
│  ServiceAccount: 専用SA、最小権限        │
│  Pod Security Standards: Restricted推奨  │
│  Network Policy: マイクロセグメンテーション│
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ データ保護                              │
│  Secrets: 外部管理、RBAC制限            │
│  automountServiceAccountToken: false    │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 検知・対応                              │
│  監査ログ（API Server全操作）           │
│  異常検知（CPU異常、不審なAPI呼び出し） │
│  定期的な権限監査                       │
└─────────────────────────────────────────┘
```

### 攻撃Stage別の対策マッピング

| Stage | 攻撃 | 対策 |
|-------|------|------|
| 1. 初期侵入 | アプリ脆弱性 | アプリケーションセキュリティ、Pod Security Standards |
| 2. 情報収集 | SA Token取得 | automountServiceAccountToken: false |
| 3. 権限確認 | API呼び出し | 監査ログ、異常検知 |
| 4. 横展開 | 他Pod攻撃 | Network Policy、専用SA |
| 5. 目的達成 | マイニング | リソース監視、cgroups制限 |

**どの層が破られても次の層で防ぐ。**

## Linux Kernel Securityとの対応

| Kubernetes | Linux Kernel | 目的 |
|------------|--------------|------|
| RBAC | Capabilities | 必要な権限のみ付与 |
| Admission Control | seccomp / AppArmor | 危険な操作を禁止 |
| Namespace（k8s） | Namespace（Linux） | リソース分離 |
| Network Policy | iptables / netfilter | 通信制御 |
| cgroups制限 | cgroups | リソース制限 |

### 自分の理解

> KubernetesはLinuxカーネルの防御機能を抽象化して宣言的に管理できるようにした。
> YAMLで定義するだけで、裏側で複雑なカーネル設定が適用される。
> これがクラウドネイティブの本質。

## まとめ: 設定ミスが最大の脅威

```
技術的に可能な防御:
  - RBAC
  - Network Policy
  - Pod Security Standards
  - Secrets暗号化

現実の脆弱性:
  - デフォルト設定のまま運用
  - 「動けばいい」で権限を広げる
  - 複雑さゆえに理解不足

結果:
  攻撃者は技術的脆弱性ではなく
  設定ミスを突く
```

### 防御の原則

1. **デフォルトを信用しない**
   - default ServiceAccount無効化
   - automountServiceAccountToken: false

2. **最小権限**
   - 専用ServiceAccount
   - Role/RoleBindingで明示的に許可

3. **分離**
   - Network Policyでマイクロセグメンテーション
   - Namespaceで論理分離

4. **外部化**
   - Secrets管理は外部システム

5. **監視**
   - 監査ログ
   - 異常検知

**鍵なしで入れて、机に金庫の鍵が整理されている状態をデフォルトとしない。**

## 関連

- [[linux-kernel-deep-dive]] - Kubernetesの基盤技術
- [[docker-deep-dive]] - コンテナランタイム
- [[container-deep-dive]] - コンテナセキュリティ全般
- [[authentication-deep-dive]] - 認証の深掘り
