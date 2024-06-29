---
title: "IAMポリシーの許可アクションを一括で更新する"
date: 2024-06-28T22:50:23+09:00
draft: false
summary: '管理コンソール上から複数のIAMポリシーを更新するのが面倒だったのでスクリプト化しました。'
tags: ['AWS', 'IAM']
---

## 背景

複数のS3バケットに個別のポリシーを割り当てて運用されている環境で、権限の見直しに伴い既存ポリシーを更新する必要が出てきました。
早い話が権限つけすぎなのでちゃんと必要最小になるように削るべきだよね。というやつです。

ただ手作業でやるには時間がかかりすぎるのでスクリプト化しました。

## 前提

S3バケット`paltee-foo-data-<id>`に対して、以下のような個別のポリシーを割り当てたユーザー`foo-data-<id>`を用意して利用しています。
これらのポリシーの許可アクションから、`s3:PutObject`, `s3:DeleteObject`を削除することを考えます。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::paltee-foo-data-<id>/*",
                "arn:aws:s3:::paltee-foo-data-<id>"
            ]
        }
    ]
}
```

方針としては、処理対象のIAMポリシーのリストアップを行い、各ポリシーのドキュメントを編集したものを再度アップロードする形をとります。

## 操作用IAMポリシーの作成
今回はアドホックな対応のため、事前に専用のIAMポリシーを作成し作業ユーザーにアタッチしておきます。
他のリソースに影響を与えないよう、使うことが想定されるアクション、リソースのみ許可するように範囲を絞っておきます。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetPolicy",
                "iam:GetPolicyVersion",
                "iam:ListPolicyVersions",
                "iam:CreatePolicyVersion"
            ],
            "Resource": "arn:aws:iam::<AWS Account ID>:policy/AmazonS3Access_foo-data-*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:ListPolicies",
            "Resource": "*"
        }
    ]
}
```

## 更新スクリプトの作成

最終的に作成したスクリプトが以下となります。

{{< highlight bash "linenos=true" >}}
#!/bin/bash
set -eu

# 更新対象ポリシーのARN取得
aws iam list-policies --output json --no-paginate --query "Policies[?contains(PolicyName, 'AmazonS3Access_foo-data-')]" | jq -r '.[] | .Arn' | \
while read arn; do
  # 現在のポリシードキュメントの取得
  aws iam get-policy-version --output json \
    --policy-arn "$arn" \
    --version-id "$(aws iam get-policy --policy-arn "$arn" | jq -r '.Policy.DefaultVersionId')" | jq -r '.PolicyVersion.Document' > policy.json &&
  # ポリシードキュメントからアクションを削除
  cat policy.json | jq -r 'del(.Statement[] | .Action | .[] | select(test("^s3:PutObject$") or test("^s3:DeleteObject$")))' > modified_policy.json &&
  # 更新したポリシードキュメントを新たなバージョンとしてアップロード
  aws iam create-policy-version \
    --policy-arn "$arn" \
    --policy-document "file://$(pwd)/modified_policy.json" \
    --set-as-default > /dev/null &&
  rm policy.json modified_policy.json &&
  echo "successfully updated policy: $arn"
done
{{< /highlight >}}

* 5行目: `list-policies`でポリシー名をベースに更新対象のポリシーを検索し、jqでそのARNを抜き出します。
* 8行目: `get-policy-version`でARNごとにポリシーのデフォルトバージョンのドキュメントを取得し保存します。
  * クエリに利用できるJMESPathは前方一致や正規表現が使えなさそうなのでcontainsとしています。意図しないポリシーが含まれないことは`list-policies`単体で確認したほうが良さそうです。
* 12行目: 保存したドキュメントから削除対象の項目をjqで削除した上で別のファイル(`modified_policy.json`)に保存します。
* 14行目: `create-policy-version`で新たなバージョンのポリシードキュメントをアップロードし、デフォルトバージョンとして設定します。

## 複数バージョンが既に存在する場合

IAMポリシーはバージョン管理されていますが、5つを超えるバージョンを保持することができません。
上記スクリプトは5つ分のバージョンが存在するケースを考慮していないので、場合によっては既存のバージョンを削除した上でcreate-policy-versionを実行する必要があります。

以下の形でデフォルトバージョンでなく最も古いバージョンIDが取得できるので、この結果の有無によりdelete-policy-versionを実行しておくのが良さそうです。

```bash
aws iam list-policy-versions --output json --no-paginate --policy-arn "$arn" | \
  jq -r 'first(.Versions | sort_by(.CreateDate) | .[] | select(.IsDefaultVersion == false)) | .VersionId'
```

## おわりに

前述した通り作業対象のIAMポリシーがそれなりにあったので、このスクリプトのおかげで楽に作業を進めることができました。
jqはたまによく使うものの、リファレンスを読みながらでないとなかなか使いこなせないことが多いのでもっと活用したいですね。
