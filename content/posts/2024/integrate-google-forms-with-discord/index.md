---
title: "Google Formsの回答をDiscordに通知する"
date: 2024-11-24T12:08:03+09:00
draft: false
summary: 'Google App Scriptを使ってGoogle Formsの回答をDiscordに通知する仕組みを作りました。'
tags: ['Google App Script', 'Google Forms', 'Discord']
---

## はじめに
BOOTHで販売しているアセットに関するフィードバックを収集するためにGoogle Formsを使ったフォームを設置しています。

今まではメールで回答の通知を設定していたのですが、メールを見落とすことがあるため回答内容をDiscordに通知する仕組みを作りました。

この記事ではその紹介とスクリプトの詳細を記載します。

## 作ったもの
今回は以下のフォームに通知を設定していきます。

{{< figure src="img/form.webp" alt="フォームの内容" >}}

上記のフォームに回答すると、以下のようなメッセージがDiscordに通知されます。

{{< figure src="img/notification.webp" alt="Discordに通知されるメッセージ" >}}

## スクリプトと設定

今回作成したスクリプトは以下のようになりました。

{{< details "script.gs" >}}

{{< highlight javascript "linenos=true" >}}
function onFormSubmit(event) {
  processResponse(event.response);
}

function processResponse(response) {
  const payload = buildDiscordMessage(response);
  const endpoint = getSecret("webhookEndpoint");
  const postFunc = buildPostToDiscordFunc(endpoint);
  postFunc(payload);
}

function buildDiscordMessage(response) {
  const form = FormApp.getActiveForm();

  const kind = buildFetchElementResponseFunc(123456780)(response);
  const productId = buildFetchElementResponseFunc(123456781)(response);
  const content = buildFetchElementResponseFunc(123456782)(response);

  const embed = {
    "description": `${form.getTitle()} に回答があります`,
    "fields": [
      {
        "name": "種別",
        "value": kind,
        "inline": true,
      },
      {
        "name": "URL",
        "value": buildBoothProductUrl(productId),
        "inline": true,
      },
      {
        "name": "内容",
        "value": content,
        "inline": false,
      },
    ],
  };
  switch (kind) {
    case "要望":
      embed["color"] = 0x609dff;
      break;

    case "不具合報告":
      embed["color"] = 0xff7777;
      break;
  }

  return {
    "embeds": [embed],
  };
}

function buildBoothProductUrl(id) {
  if (/^\d+$/.test(id)) {
    return `https://booth.pm/ja/items/${id}`;
  }
  return "";
}

function buildPostToDiscordFunc(endpoint) {
  return function(payload) {
    return UrlFetchApp.fetch(
      endpoint,
      {
        "method": "POST",
        "contentType": "application/json",
        "payload": JSON.stringify(payload),
      },
    );
  };
}

// === GAS/Form functions ===

function buildFetchElementResponseFunc(elementId) {
  return function(response) {
    for (let item of response.getItemResponses()) {
      if (item.getItem().getId() == elementId) {
        return item.getResponse();
      }
    }
    throw new Error(`Form element whose id is ${elementId} not found`);
  }
}

function getSecret(key) {
  return PropertiesService.getScriptProperties().getProperty(key);
}

// === Debug functions ===

// Dumps form elements with their id
function dumpFormElements() {
  const form = FormApp.getActiveForm();
  const items = form.getItems().map(item => `id: ${item.getId()}, title: ${item.getTitle()}`);
  console.info(items.join("\n"));
}

// Processes first response to test
function testSendFirstResponse() {
  const form = FormApp.getActiveForm();
  const responses = form.getResponses();
  if (responses.length == 0) {
    console.warn("No response recorded")
    return;
  }
  processResponse(responses[0]);
}
{{< /highlight >}}

{{< /details >}}

上記のスクリプトをGoogleフォームのスクリプトエディタに貼り付け、Discordのサーバー設定からWebhookを作成し、そのURLを取得しておきます。

このWebhookのURLは秘匿しておくためにGASのスクリプトプロパティに保存します。

{{< figure src="img/gas_script_property.webp" alt="スクリプトプロパティの設定" >}}

最後に、以下のようにトリガーを設定します。

{{< figure src="img/gas_add_trigger.webp" alt="トリガーの設定" >}}

## スクリプトの詳細

### エントリーポイント
`onFormSubmit`をフォーム回答時に実行されるトリガーとして設定し、フォームの回答1件に対して`processResponse`を呼び出しています。

`processResponse`では`buildDiscordMessage`でフォームの回答内容をDiscordに投稿するメッセージに変換し、`buildPostToDiscordFunc`でWebhookURLから作成した送信用の関数にメッセージを渡しています。

### メッセージの構築
`buildDiscordMessage`ではFormResponseオブジェクトから回答内容を[Embed](https://discord.com/developers/docs/resources/message#embed-object)に変換した上でペイロードとして返します。

フォームの回答内容はFormResponseオブジェクトとして取得できるので、必要な情報はこのオブジェクトを経由して取り出します。
{{< linkcard "https://developers.google.com/apps-script/reference/forms/form-response" />}}

ここでは要望と不具合報告の種別によってEmbedの色を変え、必要な回答項目を埋め込んでいます。

また回答項目は文言が変化してもスクリプトの修正が不要となるように、一意のIDによって取得するようにしています。このIDは後述する`dumpFormElements`で取得します。

### デバッグ、補助関数
`dumpFormElements`はフォームの項目のIDと名前を取得してコンソールに出力します。

実際に実行すると、以下のような出力が得られます。
```plain
id: 123456780, title: フィードバック種別
id: 123456781, title: 商品ID (自動入力されるため編集不要です)
id: 123456782, title: 内容
```

この結果から、`buildFetchElementResponseFunc`で取得する回答項目のIDを特定します。

`testSendFirstResponse`はフォームの一番最初の回答を引数として`processResponse`を呼び出します。
回答のトリガーのみでテストすると何回も回答を入力する必要があるため、繰り返し動作を確認する際に利用しています。

## おわりに
Google App Scriptを使ってGoogle Formsの回答をDiscordに通知する仕組みを作りました。
この仕組みにより、フォームの回答の見逃がしを防ぐことができそうです。
