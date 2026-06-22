# aws-nfw-policy-switch-poc

AWS Network Firewall の Firewall Policy を、Systems Manager Patch Manager のメンテナンス時間に合わせて一時的に切り替える PoC です。

通常運用では厳格な Firewall Policy を使用し、パッチ適用の時間帯だけ CDN やパッケージリポジトリへの通信を許可したパッチ運用用 Firewall Policy に切り替えます。

<br>

## 概要

このリポジトリでは、Amazon EventBridge Scheduler から AWS Lambda を起動し、AWS Network Firewall に関連付ける Firewall Policy を自動で切り替える構成を CloudFormation で作成します。

対象は、既存の AWS Network Firewall と、事前に作成済みの 2 つの Firewall Policy です。

| 種別 | 役割 |
| --- | --- |
| 通常運用ポリシー | 日常運用向けの厳格な通信制御 |
| パッチ運用ポリシー | Patch Manager 実行中に必要な宛先を一時的に許可 |

<br>

## リポジトリの目的

この PoC の目的は、セキュリティを維持しながら、OS パッケージ更新に必要な一時的な外部通信を運用時間内だけ許可することです。

パッケージ更新時は、OS やミドルウェアのリポジトリ、CDN、証明書失効確認先、ベンダー配布サイトなど、複数の外部宛先へ通信が発生します。

特に Cloudflare などの CDN 配下で配布されるリポジトリや、更新元 URL がリダイレクトされるケースでは、宛先ドメインや IP アドレスを固定的に洗い出し続ける運用が難しくなります。

そのため、この構成では Firewall Policy の中身を都度編集するのではなく、通常運用用とパッチ運用用の 2 つの Firewall Policy を事前に用意し、メンテナンス時間だけ関連付けを切り替えます。

<br>

## アーキテクチャ

このリポジトリの CloudFormation テンプレートは、既存の AWS Network Firewall と 2 つの Firewall Policy を対象に、関連付けの切り替え処理だけを作成します。

```mermaid
flowchart LR
    SSM[Systems Manager<br>Maintenance Window] --> Scheduler[EventBridge Scheduler]
    Scheduler -->|{"targetPolicy":"patch"}| Lambda[Lambda<br>policy switcher]
    Scheduler -->|{"targetPolicy":"normal"}| Lambda
    Lambda -->|AssociateFirewallPolicy| NFW[AWS Network Firewall]
    NFW --> Normal[通常運用 Firewall Policy]
    NFW --> Patch[パッチ運用 Firewall Policy]
    NFW --> CDN[CDN / Package Repository]
```

構成図を作成する場合は、VPC、Subnet、EC2、NAT Gateway などの既存環境を詳細に描き込むよりも、EventBridge Scheduler、Lambda、AWS Network Firewall、2 つの Firewall Policy の関係を中心に整理します。

<br>

## 作成される主な AWS リソース

| リソース | 用途 |
| --- | --- |
| AWS Lambda | Firewall Policy の切り替え処理 |
| IAM Role for Lambda | Network Firewall 操作と CloudWatch Logs 出力 |
| Amazon EventBridge Scheduler | メンテナンス前後の Lambda 起動 |
| IAM Role for Scheduler | Lambda Invoke 用の実行ロール |
| CloudWatch Logs Log Group | Lambda 実行ログ |

<br>

## Lambda 仕様

| 項目 | 値 |
| --- | --- |
| Lambda 関数名 | `nfw-policy-switcher` |
| Runtime | `python3.12` |
| Handler | `index.lambda_handler` |
| Timeout | `60` 秒 |
| Payload | `targetPolicy` に `patch` または `normal` を指定 |

Lambda 関数名は、CloudFormation パラメータ `LambdaFunctionName` で変更できます。

<br>

## 手動で用意するもの

このテンプレートでは、AWS Network Firewall 本体や Firewall Policy は作成しません。

以下は事前に作成済みであることを前提にしています。

| 項目 | 説明 |
| --- | --- |
| AWS Network Firewall | Firewall Policy の切り替え対象 |
| 通常運用 Firewall Policy | 通常時に関連付ける厳格なポリシー |
| パッチ運用 Firewall Policy | パッチ適用時に関連付ける一時緩和ポリシー |
| Patch Manager / Maintenance Window | パッチ適用処理の実行基盤 |

<br>

## 命名例

構成図やパラメータ例では、以下のような名前にすると役割が分かりやすくなります。

| 対象 | 命名例 | 用途 |
| --- | --- | --- |
| 通常運用 Firewall Policy | `NormalOperation-Policy` | 通常時に関連付ける厳格な Firewall Policy |
| パッチ運用 Firewall Policy | `PatchOperation-Policy` | Maintenance Window 中だけ関連付ける Firewall Policy |
| Lambda | `nfw-policy-switcher` | Firewall Policy の関連付け切り替え |
| Scheduler | `switch-to-patch` / `switch-to-normal` | パッチ運用への切り替え / 通常運用への戻し |

実環境では、必要に応じてシステム名や環境名を前置します。

例: `<SYSTEM_NAME>-normal-operation-policy`、`<SYSTEM_NAME>-patch-operation-policy`

<br>

## ディレクトリ構成

```text
.
├── README.md
├── LICENSE
├── docs
│   ├── architecture.md
│   ├── deployment-guide.md
│   ├── operation-notes.md
│   └── security-considerations.md
├── parameters
│   └── aws-nfw-policy-switch-poc.example.json
└── templates
    └── aws-nfw-policy-switch-poc.yml
```

<br>

## CloudFormation テンプレート

| テンプレート | 内容 |
| --- | --- |
| [templates/aws-nfw-policy-switch-poc.yml](templates/aws-nfw-policy-switch-poc.yml) | Lambda、EventBridge Scheduler、IAM Role、CloudWatch Logs を作成 |

<br>

## パラメータファイル

パラメータファイルのサンプルは以下に格納しています。

| ファイル | 内容 |
| --- | --- |
| [parameters/aws-nfw-policy-switch-poc.example.json](parameters/aws-nfw-policy-switch-poc.example.json) | CloudFormation デプロイ時のパラメータ例 |

実際にデプロイする場合は、サンプルファイルをコピーして使用します。

```bash
cp parameters/aws-nfw-policy-switch-poc.example.json parameters/aws-nfw-policy-switch-poc.json
```

`parameters/aws-nfw-policy-switch-poc.json` には、自分の環境に合わせた ARN やスケジュールを設定します。実環境用の値を含むため、GitHub にはコミットしない運用とします。

<br>

## パラメータ

| パラメータ | デフォルト | 説明 |
| --- | --- | --- |
| `FirewallArn` | なし | 切り替え対象の AWS Network Firewall ARN |
| `NormalFirewallPolicyArn` | なし | 通常運用 Firewall Policy ARN |
| `PatchFirewallPolicyArn` | なし | パッチ運用 Firewall Policy ARN |
| `LambdaFunctionName` | `nfw-policy-switcher` | 作成する Lambda 関数名 |
| `PatchScheduleExpression` | `cron(0 1 ? * SUN *)` | パッチ運用ポリシーへ切り替えるスケジュール例 |
| `NormalScheduleExpression` | `cron(0 5 ? * SUN *)` | 通常運用ポリシーへ戻すスケジュール例 |
| `ScheduleTimezone` | `Asia/Tokyo` | EventBridge Scheduler のタイムゾーン |
| `ScheduleState` | `DISABLED` | Scheduler の初期状態 |
| `LogRetentionInDays` | `30` | Lambda ログの保持日数 |

<br>

## 出力値

| 出力値 | 内容 |
| --- | --- |
| `PolicySwitcherFunctionName` | 作成された Lambda 関数名 |
| `PolicySwitcherFunctionArn` | 作成された Lambda 関数 ARN |
| `SwitchToPatchScheduleName` | パッチ運用ポリシーへ切り替える Scheduler 名 |
| `SwitchToNormalScheduleName` | 通常運用ポリシーへ戻す Scheduler 名 |
| `TargetFirewallArn` | 切り替え対象の AWS Network Firewall ARN |

<br>

## 前提条件

- AWS Network Firewall が作成済みであること
- 通常運用用とパッチ運用用の Firewall Policy が作成済みであること
- Systems Manager Patch Manager の対象インスタンスが管理対象ノードとして認識されていること
- Patch Manager の Maintenance Window と、ポリシー切り替え時刻の前後関係を設計していること

<br>

## デプロイ方法

詳細な手順は [docs/deployment-guide.md](docs/deployment-guide.md) に記載しています。

運用時の確認観点は [docs/operation-notes.md](docs/operation-notes.md) に記載しています。

セキュリティ上の考慮点は [docs/security-considerations.md](docs/security-considerations.md) に記載しています。

### スタック作成

```bash
aws cloudformation create-stack \
  --stack-name aws-nfw-policy-switch-poc \
  --template-body file://templates/aws-nfw-policy-switch-poc.yml \
  --parameters file://parameters/aws-nfw-policy-switch-poc.json \
  --capabilities CAPABILITY_NAMED_IAM
```

### Lambda テスト

パッチ運用ポリシーへ切り替える場合:

```json
{
  "targetPolicy": "patch"
}
```

通常運用ポリシーへ戻す場合:

```json
{
  "targetPolicy": "normal"
}
```

### スケジュール有効化

初期状態では `ScheduleState` を `DISABLED` にしています。

本番運用前に Lambda の手動テストと Firewall Policy の切り替え確認を実施し、問題がないことを確認してから `ENABLED` に変更します。

### 削除

CloudFormation スタックを削除すると、Lambda、EventBridge Scheduler、IAM Role、CloudWatch Logs Log Group が削除されます。

既存の AWS Network Firewall と Firewall Policy は削除されません。

<br>

## 参考

- [AWS Network Firewall - AssociateFirewallPolicy](https://docs.aws.amazon.com/network-firewall/latest/APIReference/API_AssociateFirewallPolicy.html)
- [AWS Network Firewall - Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsnetworkfirewall.html)
- [AWS Network Firewall - Stateful domain list rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-domain-names.html)
- [Amazon EventBridge Scheduler - Lambda Invoke target](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-templated.html)
- [AWS Systems Manager Patch Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/patch-manager.html)
