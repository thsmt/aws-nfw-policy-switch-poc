# デプロイガイド

## 事前準備

以下を事前に確認します。

| 項目 | 確認内容 |
| --- | --- |
| AWS Network Firewall | 切り替え対象の Firewall ARN を確認 |
| 通常運用 Firewall Policy | 通常時に関連付ける Firewall Policy ARN を確認 |
| パッチ運用 Firewall Policy | Patch Manager 実行時に関連付ける Firewall Policy ARN を確認 |
| Patch Manager | Maintenance Window の開始時刻と終了時刻を確認 |
| 実行権限 | CloudFormation で IAM Role、Lambda、Scheduler を作成できる権限を確認 |

## パラメータファイル作成

サンプルファイルをコピーします。

```bash
cp parameters/aws-nfw-policy-switch-poc.example.json parameters/aws-nfw-policy-switch-poc.json
```

コピーしたファイルに、自分の環境の値を設定します。

```json
[
  {
    "ParameterKey": "FirewallArn",
    "ParameterValue": "arn:aws:network-firewall:ap-northeast-1:123456789012:firewall/<FIREWALL_NAME>"
  },
  {
    "ParameterKey": "StrictFirewallPolicyArn",
    "ParameterValue": "arn:aws:network-firewall:ap-northeast-1:123456789012:firewall-policy/<STRICT_FIREWALL_POLICY_NAME>"
  },
  {
    "ParameterKey": "RelaxedFirewallPolicyArn",
    "ParameterValue": "arn:aws:network-firewall:ap-northeast-1:123456789012:firewall-policy/<RELAXED_FIREWALL_POLICY_NAME>"
  }
]
```

`ScheduleState` は、初回デプロイ時は `DISABLED` を推奨します。

## スタック作成

```bash
aws cloudformation create-stack \
  --stack-name aws-nfw-policy-switch-poc \
  --template-body file://templates/aws-nfw-policy-switch-poc.yml \
  --parameters file://parameters/aws-nfw-policy-switch-poc.json \
  --capabilities CAPABILITY_NAMED_IAM
```

スタック作成後にパラメータを変更する場合は、以下のようにスタックを更新します。

```bash
aws cloudformation update-stack \
  --stack-name aws-nfw-policy-switch-poc \
  --template-body file://templates/aws-nfw-policy-switch-poc.yml \
  --parameters file://parameters/aws-nfw-policy-switch-poc.json \
  --capabilities CAPABILITY_NAMED_IAM
```

## Lambda テスト

まずは EventBridge Scheduler を有効化せず、Lambda のテストイベントで動作確認します。

### パッチ運用ポリシーへ切り替え

```json
{
  "mode": "relaxed"
}
```

### 通常運用ポリシーへ戻す

```json
{
  "mode": "strict"
}
```

テスト後、AWS Network Firewall の画面で、対象 Firewall に関連付けられている Firewall Policy が切り替わっていることを確認します。

## EventBridge Scheduler 有効化

Lambda の手動テストが完了したら、Patch Manager の Maintenance Window に合わせてスケジュールを調整します。

例:

| スケジュール | 目的 | 例 |
| --- | --- | --- |
| `RelaxedScheduleExpression` | Maintenance Window 開始前にパッチ運用ポリシーへ切り替え | `cron(0 1 ? * SUN *)` |
| `StrictScheduleExpression` | Maintenance Window 終了後に通常運用ポリシーへ戻す | `cron(0 5 ? * SUN *)` |

スケジュールの時刻は、Patch Manager の実行開始前と終了後に余裕を持たせて設定します。

本番運用に入る場合は、`ScheduleState` を `ENABLED` に変更してスタックを更新します。

## 削除

```bash
aws cloudformation delete-stack \
  --stack-name aws-nfw-policy-switch-poc
```

このスタックを削除しても、既存の AWS Network Firewall と Firewall Policy は削除されません。

削除前に、対象 Firewall が通常運用 Firewall Policy に戻っていることを確認してください。
