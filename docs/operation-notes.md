# 運用メモ

## 運用の流れ

この PoC は、Patch Manager の Maintenance Window に合わせて AWS Network Firewall の Firewall Policy を切り替えることを想定しています。

通常時は厳格な通常運用ポリシーを関連付け、パッチ適用前にパッチ運用ポリシーへ切り替えます。

パッチ適用後は、通常運用ポリシーへ戻します。

<br>

## 切り替えタイミング

EventBridge Scheduler の実行時刻は、Maintenance Window の開始時刻と終了時刻に対して少し余裕を持たせます。

| タイミング | 実行内容 | 理由 |
| --- | --- | --- |
| Maintenance Window 開始前 | パッチ運用ポリシーへ切り替え | パッケージ取得前に通信許可を反映するため |
| Maintenance Window 終了後 | 通常運用ポリシーへ戻す | パッチ適用後の不要な通信許可を残さないため |

<br>

## 事前確認

本番運用に入る前に、以下を確認します。

| 項目 | 確認内容 |
| --- | --- |
| Lambda 手動実行 | `targetPolicy` に `normal` と `patch` を指定し、両方で正常終了すること |
| Firewall Policy | AWS Network Firewall の関連付けが切り替わること |
| Patch Manager | パッチ対象インスタンスが管理対象ノードとして認識されていること |
| Scheduler | タイムゾーンと cron 式が Maintenance Window と合っていること |

<br>

## 戻し忘れ対策

通常運用ポリシーへ戻すスケジュールは、パッチ適用後の安全側の時刻に設定します。

手動でパッチ運用ポリシーへ切り替えた場合も、作業終了後に `{"targetPolicy":"normal"}` の Lambda テストイベントを実行し、通常運用ポリシーへ戻っていることを確認します。

<br>

## 障害時の確認

Lambda 実行に失敗した場合は、CloudWatch Logs の Lambda ロググループを確認します。

Network Firewall の関連付け状態は、AWS Network Firewall コンソール、または `DescribeFirewall` API で確認します。
