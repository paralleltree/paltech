---
title: "Mastoshieldを作成しました"
date: 2024-02-18T16:20:37+09:00
draft: false
summary: 'ActivityPub用フィルタリングリバースプロキシを作成し、セルフホストしているインスタンスに投入しました。'
tags: ['Fediverse', 'ActivityPub', 'Go']
---

先週末にかけてFediverse方面で大量のスパムメッセージが送信され始め、自分のインスタンスにも投稿が届くようになったのでフィルタリングを目的にリバースプロキシを実装してみました。

自分も使っているMastodonではユーザーレベルでフィルタ機能がありますが、少なからずサーバーのリソースを消費するためより上流でフィルタリングできるようにすることにしました。

今回はプロキシ自体の実装に加え、ログ設定と集計までを行う環境を設定しています。

## プロキシサーバ

作成したリポジトリは以下です。

https://github.com/paralleltree/mastoshield

Goで実装し、フィルタリングの条件に基づいてリクエストを処理するリバースプロキシとして動作します。
スパムの対応にあたり、フィルタリングの条件を柔軟に組み合わせて定義できるように設計しています。

また新たなルールの種類が欲しくなった際に見通しが悪くならないよう、ルール種別ごとに実装を切り出しました。

### ルールの設定

```yml
rulesets:
  - action: deny
    rules:
      - source: note_body
        contains: blocked_text
```

ルールセットはYAMLファイルに記述した順に評価され、ルールセット内に含まれるルールすべてに一致した場合に指定のアクションが実行されます。

複数のルールセットを定義することで、信頼できるかつ普段から交流のあるサーバーは無条件で通すなどの設定ができます。

## systemdの設定

自インスタンスへの導入にあたり、今回はホストマシンのsystemdで管理することとしました。
以下のユニットファイルを作成し、有効化します。

```
[Unit]
Description=A reverse proxy server for filtering requests to mastodon
Documentation=https://github.com/paralleltree/mastoshield
After=network.target

[Service]
Type=simple
Environment=PORT=2900
Environment=UPSTREAM_ENDPOINT=http://localhost:3333
ExecStart=/usr/local/bin/mastoshield --rule-file /etc/mastoshield/config.yml
Restart=on-failure
StandardOutput=append:/var/log/mastoshield/mastoshield.ltsv.log
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

## logrotateの設定

ユニット定義では標準出力をファイル出力することとしたので、logrotateの設定を加えておきます。

```plain
/var/log/mastoshield/mastoshield.ltsv.log {
  daily
  missingok
  rotate 12
  compress
  notifempty
  copytruncate
}
```

最初にcopytruncateを設定していなかったのでその日の夜のログが切り替わりませんでした(1敗)

`logrotate -f`でちゃんと確認しておかないとダメですね。

## ログ収集の設定

今回はルールに一致したリクエスト数を集計したかったので、fluentdをインストールしてNew Relicで集約することにしました。

以下の観点から、設定ファイルを作成します。

* ログレベルがDebugのものは除外
  * 今回はInfo, Errorを集計します
* どのサービスからのログかを識別するためのラベルを追加

```plain
<source>
  @type tail
  <parse>
    @type ltsv
    time_key time
    keep_time_key
  </parse>
  path /var/log/mastoshield/mastoshield.ltsv.log
  tag mastoshield.stdout
  pos_file /var/log/mastoshield/mastoshield.ltsv.log.pos
</source>

<filter mastoshield.stdout>
  @type grep
  <exclude>
    key level
    pattern /^Debug$/
  </exclude>
</filter>

<filter mastoshield.stdout>
  @type record_transformer
  <record>
    service_name ${tag}
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match mastoshield.stdout>
  @type newrelic
  license_key XXX
</match>
```

rootでfluentdを実行した際は意図した通りに処理されていたのですが、サービスとして起動したところパーミッションエラーが出ていました。

```plain
unexpected error error_class=Errno::EACCES error="Permission denied @ rb_sysopen - /var/log/mastoshield/mastoshield.log.pos
```

fluentdのユニットファイルでは`_fluent`ユーザーで起動する設定になっていたので、posファイルのownerを変更して対応しました。

## NewRelicでの可視化

雑多にアクション別のリクエスト数とフィルタされたリクエストのパスを時系列で可視化してみました。
また必要な切り口が出てきたら適宜追加していこうと思います。

```sql
-- action count
SELECT count(1) FROM Log WHERE service_name = 'mastoshield.stdout' FACET action TIMESERIES

-- denied endpoint
SELECT count(1) FROM Log WHERE service_name = 'mastoshield.stdout' AND action = 'deny' FACET path TIMESERIES
```

{{< figure src="img/nr_dashboard.webp" alt="New Relic上に作成したダッシュボード" >}}

## おわりに

週末にスパムの件が起きてから土日で実装して実用まで持って行くことができたので良かったです。
動かすために急いでいたこともあり、ルール周りはよりよい設計にできそうであれば改善したいです。
