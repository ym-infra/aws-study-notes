# AWS Control Towerを使ってみた

## 目的

Control Towerを実際に有効化して、次のことを確かめる。

- ランディングゾーン4.0で何が変わったか
- コントロール（SCP / Config）がどう適用されるか
- Configが無効なリージョンを追加するとどうなるか
- Security Hubで検出結果を確認できるか

## Control Towerとは

マルチアカウント環境を「ルール付き」で楽に管理するための仕組み。

- AWSでは、環境や用途ごとにアカウントを分けるマルチアカウント運用が基本
- アカウントが増えると設定の抜けやバラつきが出やすく、手作業での管理は大変
- Control Towerを使うと、ルールを決めたマルチアカウント環境をまとめて整えられる

---

## 1. 有効化してみる

ランディングゾーンが4.0になって、セットアップ内容がいろいろ変わっていた。

- 参考：[Control Tower ランディングゾーン v4.0 の変更点（DevelopersIO）](https://dev.classmethod.jp/articles/202511-cloudgirl-lt-control-tower-landing-zone-version-4/)

### 1-1. セットアップ方法の選択

どちらを選んでも適切に設計されたマルチアカウント環境になり、違うのは初期セットアップの手順だけ。

| Description | 説明 |
|---|---|
| You can use your existing organization | 既存の組織を利用できる |
| You can enable AWS-managed controls | AWS管理のコントロールを有効にできる |
| You can modify all optional service integrations after setup of AWS Control Tower | セットアップ後に、すべてのオプションのサービス統合を変更できる |
| You can modify all optional integrations during setup of AWS Control Tower | セットアップ中に、すべてのオプションの統合を変更できる |
| You can create a new organization | 新しい組織を作成できる |

今回は新しいフルセットアップで進めた。もう一方は「コントロールだけ使いたい（Configの有効化などは不要）」場合向け。

<!-- 画像: セットアップ方法の選択画面 -->

### 1-2. ホームリージョン

- **追加リージョン**：統制を有効にするリージョン。Control Towerの管理下になる
- **拒否リージョン**：指定したリージョンでのAWS操作を明示的に拒否する

### 1-3. 自動アカウント登録

AWS Organizations配下に新しく追加されたアカウントを、自動でControl Towerの管理下（ガードレールの適用対象）にする仕組み。

### 1-4. 組織単位（OU）の作成

Control Towerが管理するための最低限のOU構造が作成される。デフォルトの設計では、Security OUに「ログアーカイブ」と「監査」のアカウントを入れる。

- 参考：[OUの設定（AWS公式ドキュメント）](https://docs.aws.amazon.com/ja_jp/controltower/latest/userguide/configure-ous.html)

### 1-5. サービス統合の設定

OUに既存のOUを選ぶと、次の設定で既存のアカウントを選べた。

> **疑問**：Control Towerの作成時に2つのAWSアカウントが作られ、そこではConfigとCloudTrailを使う。既に有効化していると困るのでは？

### 1-6. Configの有効化

有効化・無効化を選べる。既存で有効になっているなら、ここで無効を選べばよさそう（要確認）。

### 1-7. CloudTrailのログ保管

Configと同じく有効化・無効化を選べる（要確認）。

### 1-8. IAM Identity Center

4.0から「利用しない」を選べるようになった。以前は利用が必須で、ユーザー・グループ・許可セットが自動で作られていたため、不要なリソースができてしまうケースがあった。

- 参考：[Control Towerのアカウントアクセス設定の柔軟化（DevelopersIO）](https://dev.classmethod.jp/articles/control-tower-account-access-configuration-flexibility/)

### 1-9. バックアップ

今回は保留。

**結果**：有効化の完了まで約30分かかった。

---

## 2. ランディングゾーン4.0の変更点

2025年11月頃に新バージョンがリリースされた。「コントロールだけ使う（Configは使わない）」構成もできるようになった。

| 項目 | 従来 | 4.0 |
|---|---|---|
| Config | 自動で設定（必須） | 選択可能 |
| CloudTrail | 自動で設定（必須） | 選択可能 |
| Security OU | 必須 | 選択可能 |
| ログアーカイブ・監査アカウント | 必須 | 選択可能 |
| Identity Centerのユーザー | 必須 | 選択可能 |
| Backup | 選択可能 | 選択可能 |

- 参考：[Key changes in landing zone 4.0（AWS公式ドキュメント）](https://docs.aws.amazon.com/controltower/latest/userguide/key-changes-lz-v4.html)

---

## 3. コントロールについて

最低限守ってほしいルールを、OU単位でまとめて効かせる仕組み。主に2種類ある。

| 種類 | 実体 | 動き |
|---|---|---|
| 予防的コントロール | SCP | 違反自体をさせない（禁止する） |
| 発見的コントロール | Config | 違反は許すが検知して通知する |

### SCP（Service Control Policy）とは

組織に所属するアカウントの権限の上限を決めるポリシー。例えば次のように使う。

```
SCP-Region
  └ 特定リージョン以外を禁止

SCP-Security
  └ CloudTrail / Config の停止を禁止

SCP-IAM
  └ 権限昇格系のAPIを禁止
```

### 必須と選択

- **必須**：すべてのOUに適用され、外せないコントロール（Security OUは一部異なる）
- **選択**：OU単位で必要なものを選んで有効化できるコントロール

コントロールの一覧は公式ドキュメントを参照。

- 参考：[コントロールリファレンス（AWS公式ドキュメント）](https://docs.aws.amazon.com/ja_jp/controltower/latest/controlreference/controls-reference.html)

---

## 4. コントロールを有効化してみる

1. 管理アカウントのControl Towerで「Control Catalog」を開き、使いたいコントロールを探す
2. コントロールを有効にすると、選択したOUにSCPが適用される
3. コントロールのアーティファクト、またはOrganizationsでOUに適用されているSCPから、反映されたポリシーを確認できる

<!-- 画像: Control CatalogとOrganizations上のSCP -->

**気づき**：SCPからは「どのOUに適用されているか」は確認できるが、一覧でまとめて見られないのが少し不便。

---

## 5. 拒否リージョンの設定

### GRREGIONDENY（ランディングゾーン全体に適用）

ホームリージョンと追加リージョンには適用されない。今回は `ap-northeast-1` と `us-east-1` 以外を拒否する設定になっている。

<details>
<summary>適用されたSCP（クリックで展開）</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-northeast-1",
            "us-east-1"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:*:iam::*:role/AWSControlTowerExecution"
          ]
        }
      },
      "Resource": "*",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "access-analyzer:*",
        "account:*",
        "acm:*",
        "activate:*",
        "artifact:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "aws-portal:*",
        "billing:*",
        "billingconductor:*",
        "budgets:*",
        "ce:*",
        "chatbot:*",
        "chime:*",
        "cloudfront:*",
        "cloudtrail:LookupEvents",
        "compute-optimizer:*",
        "config:*",
        "consoleapp:*",
        "consolidatedbilling:*",
        "cur:*",
        "datapipeline:GetAccountLimits",
        "devicefarm:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways",
        "ec2:DescribeVpnGateways",
        "ecr-public:*",
        "fms:*",
        "freetier:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "invoicing:*",
        "iq:*",
        "kms:*",
        "license-manager:ListReceivedLicenses",
        "lightsail:Get*",
        "mobileanalytics:*",
        "networkmanager:*",
        "notifications-contacts:*",
        "notifications:*",
        "organizations:*",
        "payments:*",
        "pricing:*",
        "quicksight:DescribeAccountSubscription",
        "resource-explorer-2:*",
        "route53-recovery-cluster:*",
        "route53-recovery-control-config:*",
        "route53-recovery-readiness:*",
        "route53:*",
        "route53domains:*",
        "s3:CreateMultiRegionAccessPoint",
        "s3:DeleteMultiRegionAccessPoint",
        "s3:DescribeMultiRegionAccessPointOperation",
        "s3:GetAccountPublicAccessBlock",
        "s3:GetBucketLocation",
        "s3:GetBucketPolicyStatus",
        "s3:GetBucketPublicAccessBlock",
        "s3:GetMultiRegionAccessPoint",
        "s3:GetMultiRegionAccessPointPolicy",
        "s3:GetMultiRegionAccessPointPolicyStatus",
        "s3:GetStorageLensConfiguration",
        "s3:GetStorageLensDashboard",
        "s3:ListAllMyBuckets",
        "s3:ListMultiRegionAccessPoints",
        "s3:ListStorageLensConfigurations",
        "s3:PutAccountPublicAccessBlock",
        "s3:PutMultiRegionAccessPointPolicy",
        "savingsplans:*",
        "shield:*",
        "sso:*",
        "sts:*",
        "support:*",
        "supportapp:*",
        "supportplans:*",
        "sustainability:*",
        "tag:GetResources",
        "tax:*",
        "trustedadvisor:*",
        "vendor-insights:ListEntitledSecurityProfiles",
        "waf-regional:*",
        "waf:*",
        "wafv2:*"
      ],
      "Sid": "GRREGIONDENY"
    }
  ]
}
```

</details>

**結果**：拒否したシドニーリージョンでは、APIの取得がエラーになった。

<!-- 画像: シドニーリージョンでのエラー画面 -->

### CT.MULTISERVICE.PV.1（OU単位で適用）

`[CT.MULTISERVICE.PV.1] Deny access to AWS based on the requested AWS Region for an organizational unit`

設定の流れ：

1. 対象のコントロールを選ぶ
2. 適用するOUを選ぶ
3. 許可するリージョンを選ぶ
4. サービスアクションを入力する（＝拒否されたくないアクションを指定する）
5. コントロールから除外するIAMを指定する
6. 内容を確認して有効化する

基本的な動きはGRREGIONDENYと同じで、適用範囲をOU単位にできるのが違い。

---

## 6. Configが無効なリージョンを追加するとどうなるか

予防的コントロール `[AWS-GR_CONFIG_ENABLED] Enable AWS Config in all available regions` は「すべてのリージョンでConfigを有効にすること」を求める。

では、Configを有効にしていないリージョン（シドニー）をランディングゾーンに追加するとどうなるか試した。

**結果**：「混合ガバナンス」の状態になり、エラーになった。シドニーを外すとエラーは解消した。

- 参考：[混合ガバナンス（AWS公式ドキュメント）](https://docs.aws.amazon.com/ja_jp/controltower/latest/userguide/mixed-governance.html)

**気づき**：Control Towerは混合ガバナンスの状態ではコントロールを有効にできない。つまり、設定が不一致のままだとコントロールは効いていないことになる。

---

## 7. ConfigとSecurity Hubの連携

Security HubがConfigの検出結果を取り込み、EventBridgeのルールでそのイベントを拾って通知する。

```json
{
  "detail-type": ["Security Hub Findings - Imported"],
  "source": ["aws.securityhub"],
  "detail": {
    "findings": {
      "ProductName": ["Config"],
      "RecordState": ["ACTIVE"],
      "Workflow": {
        "Status": ["NEW"]
      },
      "UserDefinedFields": {
        "Urgency": ["URGENT", "NOT URGENT"]
      }
    }
  }
}
```

ターゲットを通知用のSecurityアカウントにして、検出結果を受け取れた。

<!-- 画像: 通知の受信結果 -->

---

## 8. コントロールに引っかかったルールを辿る

Security Hubで検出された `AWSControlTower_AWS-GR_RESTRICTED_SSH` を辿ってみた。

1. **Security Hub**：検出結果として表示される
2. **Config**：Control Towerが自動で作成したルールとして存在する
3. **Control Tower**：有効になっているコントロールとして確認できる

<!-- 画像: Security Hub / Config / Control Tower それぞれの画面 -->

**気づき**：画面ごとに名前が一致していないのが厄介。それでも、Control Towerで有効にしたコントロールの検出結果がSecurity Hubで確認できることは確かめられた。

---

## まとめ

- ランディングゾーン4.0では、Config・CloudTrail・Identity Centerなどが任意になり、構成の自由度が上がった
- 予防的コントロールはSCP、発見的コントロールはConfigで実現されている
- Configが無効なリージョンを含めると混合ガバナンスになり、コントロールが効かなくなる
- 検出結果はSecurity Hubに集約でき、EventBridgeで通知まで組める

## 今後やりたいこと

- [ ] 既存でConfig / CloudTrailが有効な環境での導入手順を確認する
- [ ] バックアップ設定を試す
- [ ] SCPをCloudFormation（StackSets）で配布してみる → [aws-iac-study](https://github.com/ym-infra/aws-iac-study)
