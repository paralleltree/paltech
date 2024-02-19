---
title: ".devドメインを生やした"
date: 2023-06-25T20:11:56+09:00
draft: false
summary: 'Gandiでpaltee.devドメインを取得しました。'
tags: ['Gandi']
---

2015年から使っている `paltee.net` でもろもろのホスティングなどをしていたのですが、今回.devドメインを生やしたくなったので取得してみました。

## レジストラ選定

.devドメインといえばあのGoogleが導入したTLDですが、ちょうどGoogle Domainsが提供終了するという発表のあとだったので、どのレジストラを利用するか調べることにしました。

### お名前.com

`paltee.net` はお名前.comで登録していましたが、.devドメインを確認してみると年2600円ほどと割高でした。

{{< figure src="img/dev_domain_price_onamae.webp" alt="お名前.comでの.devドメイン 年2603円" >}}

いかんせんメールもちょっと……という部分があるので他のレジストラを探してみることに。

### Gandi

Gandiでは.devドメインが年1803円となっていました。
お名前.comの.netドメインが1507円(2023年現在はサービス維持調整費が別途追加)と考えると価格帯としては同等です。

{{< figure src="img/dev_domain_price_gandi.webp" alt="Gandiでの.devドメイン 年1803円" >}}

今回はGandiでドメインを取得することにしました。

## Gandiでドメイン取得

早速Gandiのアカウントを作成したところ、認証にセキュリティーキーが使えたので登録しておきました。

{{< figure src="img/gandi_recovery.webp" alt="Gandiのセキュリティキー登録" >}}

ドメインはインフラとして重要な資産となるのでセキュアに管理できるのはありがたいですね。
YubiKeyを買った甲斐があるものです。

ドメイン登録の手続きを済ませると、ドメイン登録処理が開始されました。

{{< figure src="img/gandi_registering.webp" alt="ドメイン登録処理" >}}

しかし、メールを見ると支払いに失敗してしまったようでした。

> Gandiをご利用いただきありがとうございます。
>
> 大変恐縮でございますが、Gandiの支払いシステムがお客様のご注文(xxxxx)を受け付けませんでした。
>
> お支払いされた金額は2-3日後に返金されます。返金が完了される具体的な日時についてはご使用されたクレジットカードや銀行にお問い合わせください。
>
> 別の新しいご注文をしたい場合は、ユーザーアカウントの所有者のIDのコピー(パスポート、運転免許証など)をGandiのカスタマーケアまでPDFかJPG形式でご送信ください。

再度ドメインの購入手続きをしてみたものの、今度は支払い画面でエラーとなってしまい先に進めない状態に。

どうにもならなくなってしまい問い合わせしようとしましたが、少し待った間にドメイン登録が完了していました。

{{< figure src="img/gandi_registered.webp" alt="ドメイン登録完了" >}}

## DNSレコードとネームサーバの設定

Gandi側の管理画面を確認すると、既にいくつかのレコードが登録されていました。


{{< figure src="img/gandi_dns_records.webp" alt="Gandiで登録済みのDNSレコード" >}}

これらのレコードをCloudflareのDNS設定にコピーしておきます。

{{< figure src="img/cf_dns_records.webp" alt="Cloudflareに複製したDNSレコード" >}}

続けて、Gandiの管理画面からCloudflareのネームサーバを設定します。

{{< figure src="img/gandi_nameservers.webp" alt="Gandiのネームサーバ設定" >}}

## おわりに

今回取得したドメインは主にステージング環境を立てる際に使っていく予定です。

というのも、`foo-stg.paltee.net`のような名前でステージング環境を立てる場合、サービス名と環境名が同じレベルとなってしまうことが気になっていたためです。
トップレベルをdevドメインにして、階層へ影響しにくい命名をできればと考えています。

.devドメインはTLS通信が必須となりますが、今はLet's EncryptやACMで簡単にTLS化できるのでいい時代ですね。
自動更新も設定して、2つ目のドメインとして大切にしていこうと思います。