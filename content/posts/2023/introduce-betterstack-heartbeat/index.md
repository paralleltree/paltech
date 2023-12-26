---
title: "Better StackのHeartbeatでバッチの監視をする"
date: 2023-12-26T21:49:48+09:00
summary: 'Better StackのHeartbeatを使って定期実行するバッチの死活監視を設定しました。'
tags: ['Better Stack']
---

運用しているMastodonインスタンスのバックアップを見直すタイミングで、死活監視としてBetter StackのHeartbeatを導入してみることにしました。

## Better Stack Heartbeatについて

> Track your CRON jobs and serverless workers and get alerted if they don't run correctly. Never lose a database backup!

cronやサーバレス環境のワーカーなど、定期実行するジョブの死活監視ができるサービスです。

## Heartbeatの作成

Better Stackの管理ページから `Heartbeats > Create heartbeat` へ進みます。

まずは定期実行するジョブについての設定を行います。

{{< figure src="img/monitor_setting.webp" alt="新規Heartbeat登録画面" >}}

* Expect a heartbeat every
  * 定期実行するジョブの実行頻度を指定します。
* with a grace period of
  * ジョブの実行間隔からアラートを通知するまでの猶予時間を指定します。

次にアラート通知の設定を行います。

{{< figure src="img/escalation_setting.webp" alt="新規Heartbeat登録画面" >}}

* When there's a new incident
  * 通知先を指定します。
  * フリープランなのでSend e-mailのみにチェックを入れています。
* If the on-call person doesn't acknowledge the incident
  * オンコール担当が応答しなかった場合の通知設定を指定します。
  * 自分1人しかいないので `Do nothing` を指定しています。

設定を入力したら`Create heartbeat`をクリックしてHeartbeatを作成します。

## 実際の動作

実際に以下の条件で動作をテストしてみます。
* Expect a heartbeat every 5 minutes
* with a grace period of 1 minute

{{< figure src="img/heartbeat_details.webp" alt="新規作成したHeartbeat" >}}

最初のHeartbeatを送るために、表示されているエンドポイントをcurlで叩きます。
リクエストが届くと、ステータスがPendingからUpに変化します。

{{< figure src="img/heartbeat_details_after_request.webp" alt="リクエストを送った後のHeartbeat" >}}

最後のリクエストを行ってから、`実行間隔で指定した5分+猶予時間として指定した1分`の6分が経過すると、新たなインシデントが作成され、Downステータスへ変化します。

{{< figure src="img/heartbeat_details_alert.webp" alt="アラート状態へ変化したHeartbeat" >}}

Slack integrationを導入していれば同時に通知されます。

{{< figure src="img/slack_heartbeat_alert.webp" alt="HeartbeatのSlack通知" >}}


ここから再度エンドポイントへリクエストを送るとUpステータスへと復帰します。

{{< figure src="img/heartbeat_details_resolved.webp" alt="復帰したHeartbeat" >}}

## おわりに

日次で取得しているMastodonのデータベースのバックアップ処理にheartbeatの送信を入れ、バックアップに失敗している場合に通知するように設定しました。

{{< figure src="img/db_backup_heartbeat.webp" alt="復帰したHeartbeat" >}}

エンドポイントへアクセスする処理の追加で監視ができるので導入しやすく、体験としても良いなと感じます。
ただ実行間隔が長くなるほど障害の際の検知は遅れてしまうので、日次レベルだとSlackのIncoming Webhookあたりを活用するのが適している気もします。
