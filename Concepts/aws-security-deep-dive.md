---
topic: AWS Security Deep Dive
category: cloud-security
tags: [security, aws, iam, s3, cloud, deep-dive]
created: 2026-01-06
---

# AWS セキュリティ深掘り

クラウドセキュリティの根幹となるAWSセキュリティを理解する。

## AWSセキュリティの3大攻撃面

```
IAM
  └─ 権限昇格、認証情報漏洩

S3
  └─ 設定ミス、データ漏洩

EC2 Metadata Service
  └─ SSRF経由の認証情報窃取
```

この3つが攻撃者の主要標的。
すべて「設計の複雑さ」が原因で脆弱性が生まれる。

## IAM（Identity and Access Management）

### 構成要素

```
User
  └─ 人間、またはアプリケーション（長期認証情報）

Role
  └─ 一時的に引き受ける権限（短期認証情報）

Policy
  └─ 権限定義（JSONドキュメント）
```

### KubernetesRBACとの対応

| AWS IAM | Kubernetes RBAC | 説明 |
|---------|-----------------|------|
| User | User | 人間またはアプリケーション |
| Role | ServiceAccount | 一時的に引き受ける権限 |
| Policy | Role/ClusterRole | 何ができるかの定義 |
| Policy attach | RoleBinding | 誰がどの権限を持つか |

**共通するパターン: 権限の分離と最小化。**

### AssumeRoleの設計

```
永続的Access Key（従来）:
  AccessKeyId: AKIAIOSFODNN7EXAMPLE
  SecretAccessKey: wJalrXUtnFEMI/...
  → 漏洩すると永久に悪用可能

AssumeRole（一時認証情報）:
  AccessKeyId: ASIAIOSFODNN7EXAMPLE
  SecretAccessKey: wJalrXUtnFEMI/...
  SessionToken: FwoGZXIvYXdzEBYaD...
  Expiration: 2026-01-06T06:00:00Z  ← 1時間後に失効
  → 漏洩しても被害がスケールしづらい
```

#### 自分の洞察

> コントロールを奪われても被害がスケールしづらい。
> 1時間後には自動的に無効化される。
> これは設計の天才的な部分。

### PassRole攻撃

**シナリオ: 弱い権限から強い権限へエスカレーション**

#### 攻撃フロー

```
1. 攻撃者: 弱い権限のUserを取得
   └─ Lambda作成権限
   └─ PassRole権限（iam:PassRole）

2. 攻撃者: Lambda関数を作成
   └─ Admin Roleを紐付け（PassRoleを使用）

3. 攻撃者: Lambda関数を実行
   └─ Lambdaが実行時にAdmin Roleを引き受ける
   └─ Admin権限を取得

4. 攻撃者: S3全削除、アカウント乗っ取り等
```

#### なぜ成功するか

```json
{
  "Effect": "Allow",
  "Action": [
    "lambda:CreateFunction",
    "iam:PassRole"
  ],
  "Resource": "*"  ← これが危険
}
```

**PassRoleの意味: 「他のサービスにこのRoleを渡す権限」**

Lambdaに強力なRoleを渡せてしまう = 自分がそのRoleを引き受けるのと同義。

#### 防御策

##### 1. PassRoleのResource制限

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/LambdaBasicRole"
}
```

**"*"は絶対に使わない。**

##### 2. Role側の信頼ポリシー制限

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "aws:SourceAccount": "123456789012"
      }
    }
  }]
}
```

**「誰が」「どの条件で」引き受けられるかを制限。**

##### 3. Permission Boundary

```
User/Roleに付与できる権限の上限を設定
  → PassRoleで渡せるRoleもこの上限を超えられない
```

**権限の天井を設ける。**

#### 自分の気づき

> PassRoleは「代理人に権限を委任する権限」。
> 委任先が侵害されると元の権限以上のことができてしまう。
> 最小権限の原則が破られる瞬間。

## S3（Simple Storage Service）

### アクセス制御の層

```
1. Bucket Policy（バケット単位）
     ↓
2. ACL（オブジェクト単位、レガシー）
     ↓
3. Block Public Access（強制ブロック）← これが最優先
```

**複数の設定が重なり、優先順位が複雑。**

### 漏洩が頻発する理由

```
設定項目が多い
  → 複雑さがミスを誘発

一時的な設定の戻し忘れ
  → 「テスト用に公開」→「戻すの忘れた」

設定の優先順位の混乱
  → 「Bucket Policyで許可したけどACLで拒否」
  → 「どっちが効いてる?」
```

#### 自分の洞察

> 設定項目が多いから逆にミスが誘発される。
> 柔軟性と安全性のトレードオフ。
> 結局は「デフォルト拒否」が最強。

### 実際の漏洩事例

| 年 | 事件 | 被害 |
|----|------|------|
| 2017 | 米国防総省契約企業 | 18億件の個人情報 |
| 2019 | Capital One | 1億人の顧客データ |
| 2020 | コロナ関連データ | 複数の医療データ |

**共通点: S3の設定ミス。**

### Block Public Access（対策の切り札）

```
アカウント全体で有効化:
  ☑ Block all public access
    ├─ Block public access to buckets and objects granted through new access control lists (ACLs)
    ├─ Block public access to buckets and objects granted through any access control lists (ACLs)
    ├─ Block public access to buckets and objects granted through new public bucket or access point policies
    └─ Block public and cross-account access to buckets and objects through any public bucket or access point policies
```

**個別設定より優先される仕組み。**

「公開したいバケットがあるなら明示的に例外設定」というアプローチ。

#### 効果

```
従来:
  Bucket Policy + ACL + IAM → 複雑な組み合わせ
  → どこかで穴が開く

Block Public Access:
  すべてを上書きして拒否
  → 例外は明示的に設定
  → デフォルト拒否の実現
```

**複雑さを力技で解決。**

## EC2 Metadata Service（IMDS）

### 仕組み

```
EC2インスタンス内から:

curl http://169.254.169.254/latest/meta-data/
  → インスタンス情報を取得

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
  → そのインスタンスに紐付いたIAM Roleの一時認証情報を取得
```

**インスタンス内からのみアクセス可能。**
**外部からは到達不可能。**

### SSRF攻撃パターン

```
1. Webアプリに脆弱性（SSRF）
     ↓
2. 攻撃者: http://169.254.169.254/latest/meta-data/iam/security-credentials/
     ↓
3. アプリがそのURLにアクセス
     ↓
4. メタデータサービスが応答
     ↓
5. 一時認証情報（AccessKeyId, SecretAccessKey, Token）取得
     ↓
6. 攻撃者: その認証情報でAWS APIを呼び出し
     ↓
7. S3からデータ抽出、EC2起動、等
```

**SSRF = 外部から内部へのトンネル。**

### Capital One事件（2019）

#### 攻撃フロー

```
1. WAFの設定ミス（SSRF可能）
     ↓
2. メタデータサービスにアクセス
     ↓
3. IAM Role認証情報を取得
     ↓
4. S3から1億人分のデータ抽出
```

#### 失敗した4層

```
ネットワーク層:
  WAF設定ミス → SSRF可能

認証層:
  IMDSv1（トークンなし）→ 単純なHTTP GETで認証情報取得可能

権限層:
  IAM Role過剰権限 → S3への読み取り権限が広すぎ

リソース層:
  S3暗号化/鍵分離なし → データが平文で保存
```

#### 自分の分析

> 4層が全部失敗: 認証、権限、ネットワーク、リソース。
> Defense in Depthが機能していなかった。
> どれか1つでも止められていれば被害は防げた。

### IMDSv2（緩和策）

#### 違い

```
IMDSv1:
  GET http://169.254.169.254/latest/meta-data/...
  → すぐ返す

IMDSv2:
  1. PUT http://169.254.169.254/latest/api/token
       Header: X-aws-ec2-metadata-token-ttl-seconds: 21600
       → トークン取得

  2. GET http://169.254.169.254/latest/meta-data/...
       Header: X-aws-ec2-metadata-token: トークン
       → トークン付きでなければ拒否
```

#### SSRF対策としての効果

```
多くのSSRF:
  URLのみ指定可能
  ヘッダーは指定不可
  → PUT + ヘッダーが難しい
  → 攻撃成功率低下
```

**ただし完全防御ではない。**

#### IMDSv2でも突破されるケース

```
1. フルHTTP制御可能なSSRF
     → PUT + ヘッダー送信可能

2. RCE（コード実行）
     → curlを直接実行できる

3. プロキシ経由のヘッダーインジェクション
     → X-Forwarded-For等を悪用
```

#### 結論

> 緩和策であり完全防御ではない。
> 「SSRF対策」と「IMDS保護」は両方必要。

## CloudTrail（監視）

### 役割

```
AWSのAPIコールを全記録:
  - 誰が（UserAgent, IPアドレス, User/Role）
  - いつ（Timestamp）
  - 何をしたか（API名、パラメータ）
  - 結果（成功/失敗、エラーメッセージ）
```

**AWS版の監査ログ。**

### 攻撃者の回避パターン

#### 1. StopLogging（露骨）

```bash
aws cloudtrail stop-logging --name my-trail
```

**検知されやすい。**
**StopLogging自体がCloudTrailに記録される（停止前）。**

#### 2. ログ削除（S3バケットから）

```bash
aws s3 rm s3://cloudtrail-logs-bucket/ --recursive
```

**S3の操作もCloudTrailに記録される（別のログとして）。**
**ただし、S3バケットが侵害されていれば可能。**

#### 3. 記録されないAPIを使う

```
CloudTrail = Control Plane（管理操作）のみ記録

Data Plane（データアクセス）:
  - S3オブジェクトの読み書き
  - DynamoDBのデータ読み書き
  → デフォルトでは記録されない（有効化が必要）
```

**攻撃者: データを抜いても記録されない。**

#### 4. 別リージョンで活動

```
CloudTrailは有効化したリージョンのみ記録
  → 未有効なリージョンでEC2起動、等
  → 記録されない
```

**全リージョン有効化が必須。**

### 防御策

```
1. 全リージョン有効化
     ↓
2. ログバケット保護
     - 削除不可（Bucket Policy）
     - 別アカウントに保存
     - MFA Delete有効化
     ↓
3. StopLoggingアラート
     - CloudWatch Alarm
     - SNS通知
     ↓
4. Data Planeログ有効化
     - S3データイベント
     - DynamoDBデータイベント
```

**監視を監視する。**

## AWS Security全体像

```
予防:
├─ IAM最小権限、PassRole制限
├─ S3 Block Public Access
├─ IMDSv2強制
├─ Security Group最小化
└─ 不要なサービス無効化

検知:
├─ CloudTrail（API記録）
├─ GuardDuty（脅威検知AI）
├─ Config（設定変更監視）
├─ VPC Flow Logs（ネットワーク）
└─ CloudWatch Logs（アプリケーション）

対応:
├─ 認証情報の即時無効化
├─ フォレンジック（CloudTrail分析）
├─ インシデント対応プレイブック
└─ 事後レビュー
```

**Defense in Depth: どれか1つで止める。**

## 設計原則との接続

### Shared Responsibility Model

```
AWS:
  - 物理セキュリティ
  - ネットワークインフラ
  - ハイパーバイザー

ユーザー:
  - IAM設定
  - S3設定
  - アプリケーション脆弱性
```

**マンション比喩: 建物はAWS、部屋の鍵は自分。**

### Defense in Depth

```
ネットワーク → 認証 → 認可 → リソース → 監視
  ↓            ↓       ↓       ↓         ↓
SG          IMDS     IAM     暗号化   CloudTrail
```

**どこか1つで止める。全部失敗すると漏洩（Capital One）。**

### Least Privilege

```
必要最小限の権限のみ:
  - PassRoleのResource制限
  - S3 Bucket PolicyのPrincipal制限
  - IAM RoleのCondition制限
```

**"*"は原則禁止。**

### Assume Breach

```
侵入は防げない前提:
  - CloudTrail監視
  - GuardDuty異常検知
  - 一時認証情報（被害のスケール防止）
```

**検知と対応の速度が勝負。**

## まとめ: 自分の洞察

### コントロールを奪われても被害がスケールしづらい

AssumeRoleの一時性: 1時間で失効するため、漏洩しても永続的な被害にならない。

### 設定項目が多いから逆にミスが誘発される

S3の複雑さ: 柔軟性と安全性のトレードオフ。Block Public Accessで力技解決。

### 4層が全部失敗

Capital One分析: ネットワーク、認証、権限、リソースがすべて破られた。
Defense in Depthの重要性。

### IMDSv2は緩和策、完全防御ではない

SSRF対策とIMDS保護は別物。両方必要。

## 関連

- [[container-deep-dive]] - コンテナセキュリティ全般
- [[docker-deep-dive]] - Dockerセキュリティ
- [[kubernetes-deep-dive]] - Kubernetesセキュリティ
- [[authentication-deep-dive]] - 認証の深掘り
