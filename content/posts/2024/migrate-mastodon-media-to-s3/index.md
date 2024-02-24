---
title: "MastodonのメディアをS3へ移行した"
date: 2024-02-24T21:52:17+09:00
draft: false
summary: 'セルフホストしているMastodonインスタンスのメディアをS3+CloudFrontで配信するようにしました'
tags: ['Mastodon', 'Amazon S3', 'AWS', 'Amazon CloudFront']
---

セルフホストしているMastodonインスタンスのメディアをS3+CloudFrontで配信するようにしました。

## 背景

セルフホストしているMastodonインスタンスですが、日次でデータベースのバックアップを取得している一方、ホストのDocker Volumeに保存している画像などのメディアはバックアップを取得していません。

大きな問題なくかれこれ2年ほどが経過していますが、現状の運用で自宅サーバに障害が発生した場合、完全なインスタンス復元ができなくなることとなります。

メディアはバックアップをとるにはサイズも大きく、可用性を加味した結果オブジェクトストレージへの移行を行うこととしました。
個人のインスタンスなのでそれほどストレージ容量は使わないという前提のもと、S3+CloudFrontの構成を選定しています。

## 環境構築

### 証明書をリクエスト

後述するCloudFormationによるスタック作成の前に、必要な証明書をus-east-1リージョンでリクエストしておきます。

本当はここもCloudFormationでやっておきたいのですが、us-east-1リージョンの証明書しか使えない都合で手発行しておく必要がありました。

必要なCNAMEレコードを設定して証明書が発行されることを確認します。

{{< figure src="img/cname_setting.webp" >}}

### CloudFormationでスタック作成

S3バケット、CloudFrontのDistribution、アプリケーションからアクセスするためのIAM Userを以下のテンプレートを使って作成します。

S3バケットへのパブリックアクセスは全てブロックし、オリジンアクセスコントロールを定義しました。
これを用いてバケットポリシーとしてCloudFront経由のアクセスのみ許可するようにしています。
また、バケットポリシーにはアプリケーション用のユーザーに対して読み取り、書き込みを含めた権限を許可しています。

{{< details "template.yaml">}}

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Mastodon media stacks

Parameters:
  MediaBucketName:
    Type: String
  MediaDistributionDomain:
    Type: String
  MediaDistributonDomainCertificateArn:
    Type: String
    Description: Arn of surving media domain's certificate in us-east-1 region.

Resources:
  AppUser:
    Type: AWS::IAM::User

  ManageMediaPolicy:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action: sts:AssumeRole
            Principal:
              AWS: !GetAtt AppUser.Arn
      Policies:
        - PolicyName: ManageObjectPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - "s3:*"
                Resource:
                  - !Sub "${MediaBucket.Arn}"
                  - !Sub "${MediaBucket.Arn}/*"

  AccessMediaPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref MediaBucket
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action:
              - s3:GetObject
            Resource: !Sub "${MediaBucket.Arn}/*"
            Principal:
              Service: cloudfront.amazonaws.com
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${MediaDistribution}"
          - Effect: Allow
            Action:
              - s3:ListBucket
            Resource: !GetAtt MediaBucket.Arn
            Principal:
              AWS: !GetAtt AppUser.Arn
          - Effect: Allow
            Action: s3:*
            Resource: !Sub "${MediaBucket.Arn}/*"
            Principal:
              AWS: !GetAtt AppUser.Arn

  MediaDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        DefaultCacheBehavior:
          AllowedMethods: [HEAD, GET]
          CachedMethods: [HEAD, GET]
          CachePolicyId: !Ref MediaDistributionCachePolicy
          TargetOriginId: !Sub mastodon-media-origin-${AWS::StackName}
          ViewerProtocolPolicy: redirect-to-https
        Origins:
          - Id: !Sub mastodon-media-origin-${AWS::StackName}
            DomainName: !GetAtt MediaBucket.DomainName
            OriginAccessControlId: !Ref MediaOAC
            S3OriginConfig:
              OriginAccessIdentity: ''
        Aliases:
          - !Ref MediaDistributionDomain
        ViewerCertificate:
          AcmCertificateArn: !Ref MediaDistributonDomainCertificateArn
          SslSupportMethod: sni-only
          MinimumProtocolVersion: TLSv1.2_2021

  MediaDistributionCachePolicy:
    Type: AWS::CloudFront::CachePolicy
    Properties:
      CachePolicyConfig:
        Name: !Sub mastodon-media-distribution-cache-policy-${AWS::StackName}
        DefaultTTL: 300
        MaxTTL: 600
        MinTTL: 60
        ParametersInCacheKeyAndForwardedToOrigin:
          CookiesConfig:
            CookieBehavior: none
          EnableAcceptEncodingBrotli: true
          EnableAcceptEncodingGzip: true
          HeadersConfig:
            HeaderBehavior: none
          QueryStringsConfig:
            QueryStringBehavior: none

  MediaOAC:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub mastodon-media-oac-${AWS::StackName}
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

  MediaBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName:
        Ref: MediaBucketName

Outputs:
  AppUser:
    Description: Application user ARN
    Value: !GetAtt AppUser.Arn
  MediaDistributionDomain:
    Description: Media distribution domain name
    Value: !GetAtt MediaDistribution.DomainName
```

{{< /details >}}

### バケットへのアクセス、コンテンツの配信テスト

メディア用のドメインに対して、`Outputs.MediaDistributionDomain`で出力されたドメインを向き先とするCNAMEレコードを追加しておきます。

{{< figure src="img/media_cname.webp" alt="メディア用のCNAME追加" >}}

アプリ用として作成したIAMユーザーからS3バケットへファイルが転送できること、CloudFrontを経由してメディア用ドメインからアクセスできることを確認します。

```plain
$ aws configure --profile don
$ aws s3 cp --profile don karin.png s3://mastodon-media/
```

{{< figure src="img/don_media.webp" alt="CloudFront経由で配信確認" >}}

## データ転送
### 古いキャッシュの削除

次に転送するデータ量の削減のため、削除可能なデータを削除しておきます。

今回は作業時点で約17GB程度のメディアオブジェクトが存在していました。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media usage
Attachments:    7.28 GB (640 MB local)
Custom emoji:   317 MB (431 KB local)
Preview cards:  5.66 GB
Avatars:        1.35 GB (137 KB local)
Headers:        2.9 GB (619 KB local)
Backups:        0 Bytes
Imports:        0 Bytes
Settings:       0 Bytes
```

まずはどこからも参照されていないメディアを`tootctl media remove-orphans`で削除します。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media remove-orphans
Found and removed orphan: cache/accounts/avatars/110/830/347/706/730/467/original/841a983bcb247d94.png
Found and removed orphan: cache/accounts/avatars/111/210/061/034/048/246/original/cb1a9869468eb0dd.png
Found and removed orphan: cache/accounts/avatars/111/595/372/276/464/869/original/f076d159908d3870.png
Found and removed orphan: cache/accounts/avatars/111/756/638/470/376/624/original/e02bb6c0b646f12b.png
Found and removed orphan: cache/accounts/headers/110/166/896/700/973/177/original/074262d8fa781a7c.jpeg
Found and removed orphan: cache/accounts/headers/110/689/523/849/527/471/original/0d40d99792c46e30.png
Found and removed orphan: cache/accounts/headers/110/830/347/706/730/467/original/64b84d818e106ebe.png
Found and removed orphan: cache/accounts/headers/111/756/638/470/376/624/original/1d23c5ad4a3455e5.png
Found and removed orphan: cache/preview_cards/images/000/001/857/original/b947f54673a9ca40.png
Found and removed orphan: cache/preview_cards/images/000/097/900/original/01e61bc98e62d7ff.png
163585/163585 |===========================================================================================| Time: 00:04:32
Removed 10 orphans (approx. 3.57 MB)
```

次に古いメディアキャッシュを`tootctl media remove`で削除します。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media remove --days 2 --dry-run
6557/6557 |===========================================================================================| Time: 00:00:02
Removed 6557 media attachments (approx. 5.84 GB) (DRY RUN)
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media remove --days 2
6557/6557 |===========================================================================================| Time: 00:00:48
Removed 6557 media attachments (approx. 5.84 GB)
```

続けて古いプロフィール画像、ヘッダ画像のキャッシュも削除しておきます。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media remove --days 2 --remove-headers
21112/21112 |===========================================================================================| Time: 00:04:49
Visited 21112 accounts and removed profile media totaling 2.82 G
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media remove --days 2 --prune-profiles
9249/9249 |===========================================================================================| Time: 00:02:05
Visited 9249 accounts and removed profile media totaling 360 MB
```

`tootctl preview_cards remove`でプレビューカードも消しておきます。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl preview_cards remove --days 2 --dry-run
83690/83690 |===========================================================================================| Time: 00:00:28
Removed 83690 preview cards (approx. 5.59 GB) (DRY RUN)
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl preview_cards remove --days 2
83690/83690 |===========================================================================================| Time: 00:07:23
Removed 83690 preview cards (approx. 5.59 GB)
```
こちらは結構時間がかかりました。

ここまでの作業で、メディアの総容量が4GBを切るくらいになりました。

```plain
paltee@userver ~/docker/don/don_paltee_net $ docker compose exec web bin/tootctl media usage
Attachments:    1.82 GB (648 MB local)
Custom emoji:   317 MB (431 KB local)
Preview cards:  58.3 MB
Avatars:        1020 MB (137 KB local)
Headers:        87.9 MB (619 KB local)
Backups:        0 Bytes
Imports:        0 Bytes
Settings:       0 Bytes
```

## バケットへの転送

残ったメディアを転送します。

`docker-compose.yml`ではメディアのvolumeを以下のようにマウントしています。

```yml
services:
  web:
    build: .
    # ...
    volumes:
      - web_public:/mastodon/public/system
```

今回はデータ転送用のコンテナを用意し、メディア用volumeをマウントした上で転送することとしました。
以下の流れでsyncします。

```bash
$ docker run -it --rm -v don_paltee_net_web_public:/public ubuntu /bin/bash
$ apt update && apt install curl unzip
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && ./aws/install
$ aws configure
$ cd /public
$ aws s3 sync ./ s3://mastodon-media/
```

## オブジェクトストレージへの向き先切り替え

### アプリケーションの環境変数設定

必要な環境変数を設定します。

```env
# File storage (optional)
# -----------------------
S3_ENABLED=true
S3_REGION=ap-northeast-1
S3_BUCKET=mastodon-media
AWS_ACCESS_KEY_ID=XXX
AWS_SECRET_ACCESS_KEY=XXX
S3_ALIAS_HOST=don-media.paltee.net
S3_PERMISSION=private
```

今回はS3バケットで全てのパブリックアクセスをブロックしているので、`S3_PERMISSION=private`を追加で指定しています。

https://docs.joinmastodon.org/admin/optional/object-storage/#s3_permission

### nginxのリダイレクト設定

移行前のメディアURLへのアクセスに対応するため、リダイレクトを指示する以下のディレクティブを追加しました。

```
location /system {
  rewrite ^/system(.*)$ https://don-media.paltee.net$1 permanent;
}
```

設定ファイルの検証をしておきます。

```
$ nginx -t
```

### 環境変数反映と最終sync

アプリケーションを停止し、最終的なメディアの状態をバケットへ再syncしてからアプリケーションを起動します。

```bash
# コンテナ停止
$ docker compose stop
# 停止時点のメディアをsync
$ docker run -it --rm -v don_paltee_net_web_public:/public ubuntu /bin/bash
$ apt update && apt install curl unzip
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && ./aws/install
$ aws configure
$ cd /public
$ aws s3 sync ./ s3://mastodon-media/
$ exit
# メディアのリダイレクト設定を反映
$ sudo systemctl restart nginx
# 環境変数を反映しコンテナ再起動
$ docker compose up -d --force-recreate
```

## おわりに

MastodonのメディアをS3+CloudFront構成のオブジェクトストレージへ移行しました。
当初の目的としていたメディアの可用性を確保でき、サーバが回復不可能な障害に遭っても影響が抑えられる環境になったので良かったです。
