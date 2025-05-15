---
title: "Redash v10.0.0 から v25.1.0 に大幅アップグレード"
emoji: "📈"
type: "tech"
topics: ["AWS", "Redash", "ECS"]
published: true
publication_name: "primenumber"
---

## はじめに

下記の記事の通り、primeNumber では ECS で Redash を稼働させています。
https://zenn.dev/primenumber/articles/a410343c35b341

その後、2021/11/24 の v10.1.0 を最後に停止していた Redash のリリース(v25.1.0)があったのでアップグレードを行いました。([Releases · getredash/redash](https://github.com/getredash/redash/releases))

[Release v25\.1\.0 · getredash/redash](https://github.com/getredash/redash/releases/tag/v25.1.0) では v10.1.0 からのアップグレードが記載されています。  
v10.0.0 を利用していたので、まずは v10.1.0 にアップグレードし、その後 v25.1.0 にアップグレードしました。

::::details 利用者目線での機能差分 by ChatGPT
Redash を v10.1.0 から v25.1.0 にアップデートすることで、ユーザーインターフェースに以下のような改善・変更があります。

---

## 🎨 クエリエディタの改善

- クエリ補完（オートコンプリート）の精度が向上し、SQL 編集がより快適に。
- クエリ保存後の画面遷移やボタン配置が改善。
- クエリパラメータの入力 UI が直感的に改善（例：日付入力、ドロップダウンなど）。

---

## 📊 ビジュアライゼーションの改良

- チャート編集画面で軸・凡例・色などの設定 UI が整理され、編集しやすく。
- 一部のチャートに新オプション追加（例：凡例の表示位置、データラベルのオンオフ）。
- 日付軸のフォーマット指定の柔軟性が向上。

---

## 🧾 ダッシュボード機能の拡張

- ダッシュボード内ウィジェットのリサイズ操作が改善。
- ウィジェットのドラッグ＆ドロップ配置がスムーズに。
- テキストウィジェットの Markdown 表示が改良（レイアウト崩れの修正など）。

---

## 🔍 クエリ・ダッシュボード検索の改善

- クエリ/ダッシュボードの検索でフィルタや候補の表示が改善。
- タグ機能が強化され、タグでの絞り込みがしやすく。

---

## 🌐 UI 表示ラベル・メッセージの改善

- ボタンやエラーメッセージなどの UI 文言がわかりやすく整理。
- 一部通知メッセージが操作ガイド付きの内容に変更。

---

## 👤 アカウント・ログイン画面の改善

- ログイン画面でのスタイル不具合が修正。
- パスワード変更の UI が改善され、入力エラー時のメッセージが明確に。

---

## 📌 補足

- 管理者向け機能（SAML 認証設定・Docker マイグレーション・セキュリティパッチなど）は本ドキュメントには含んでいません。
- より詳細な技術的な変更については以下を参照ください：  
  [🔗 GitHub 差分（v10.1.0...v25.1.0）](https://github.com/getredash/redash/compare/v10.1.0...v25.1.0)

---

## ✅ 対象者

- Redash を日常的に利用している一般ユーザー
- クエリ作成者
- ダッシュボードの閲覧・共有を行うユーザー
  ::::

本稿では Tips を添えて ECS で稼働している Redash のアップグレード手順を説明します。

## ECS で稼働している Redash のアップグレード手順

Redash のアップグレードには DB のマイグレーションが不要な場合と必要な場合があります。  
今回のアップグレードはちょうど両方のケースを含んでいました。

- v10.0.0 → v10.1.0: DB のマイグレーションが不要
- v10.1.0 → v25.1.0: DB のマイグレーションが必要

今回は具体的なバージョンを示しながら手順を紹介しますが、DB のマイグレーションが必要/不要な場合のアップグレード手順として以降のアップグレードの際にも流用可能な見込みです。

### v10.0.0 から v10.1.0 へのアップグレード(DB マイグレーション不要)

ref: [Release v10\.1\.0 · getredash/redash](https://github.com/getredash/redash/releases/tag/v10.1.0)

1. [[事前作業] Redash 本体のイメージを更新するための PR の Approval を取得しておく](#%5B%E4%BA%8B%E5%89%8D%E4%BD%9C%E6%A5%AD%5D-db-%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%A6%E3%81%8A%E3%81%8F)
1. [サービス停止](#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E5%81%9C%E6%AD%A2)
1. [DB のバックアップ](#db-%E3%81%AE%E3%83%90%E3%83%83%E3%82%AF%E3%82%A2%E3%83%83%E3%83%97)
1. [アップグレード](#%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89)
1. [サービス再開](#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E5%86%8D%E9%96%8B)
1. [動作確認](#%E5%8B%95%E4%BD%9C%E7%A2%BA%E8%AA%8D)

### v10.1.0 から v25.1.0 へのアップグレード(DB マイグレーション必要)

ref: [Release v25\.1\.0 · getredash/redash](https://github.com/getredash/redash/releases/tag/v25.1.0)

1. [[事前作業] DB マイグレーションタスクのイメージを更新しておく](#%5B%E4%BA%8B%E5%89%8D%E4%BD%9C%E6%A5%AD%5D-db-%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%A6%E3%81%8A%E3%81%8F)
1. [[事前作業] Redash 本体のイメージを更新するための PR の Approval を取得しておく](#%5B%E4%BA%8B%E5%89%8D%E4%BD%9C%E6%A5%AD%5D-db-%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%A6%E3%81%8A%E3%81%8F)
1. [サービス停止](#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E5%81%9C%E6%AD%A2)
1. [DB のバックアップ](#db-%E3%81%AE%E3%83%90%E3%83%83%E3%82%AF%E3%82%A2%E3%83%83%E3%83%97)
1. [DB のマイグレーション](#db-%E3%81%AE%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)
1. [アップグレード](#%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89)
1. [サービス再開](#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E5%86%8D%E9%96%8B)
1. [動作確認](#%E5%8B%95%E4%BD%9C%E7%A2%BA%E8%AA%8D)

## 手順詳細

ECS Service や Task の粒度など、アーキテクチャは過去の記事を参照していただければと思います。

https://zenn.dev/primenumber/articles/a410343c35b341#%E6%9C%80%E7%B5%82%E6%A7%8B%E6%88%90

### [事前作業] DB マイグレーションタスクのイメージを更新しておく

:::message
Redash のリリースページなどでは手続き的な作業は docker compose で行う方法しか案内されません。
ECS で Redash を管理している場合、コマンドを実行するための one-shot タスクを用意しておくと便利です。
:::

Redash 本体(server, scheduler, worker など)よりも前に、DB マイグレーションを行う ECS Task のイメージを更新しておきます。

DB マイグレーションは公式では `docker-compose run --rm server manage db upgrade` と案内されており、server コンテナでコマンドを実行していることがわかります。  
そのため、DB マイグレーションは server の ECS Task をベースに定義しました。

```diff
@@ -3,14 +3,8 @@
     {
-      "name": "server",
-      "image": "redash/redash:10.1.0.b50633",
+      "name": "db-migrate",
+      "image": "redash/redash:25.1.0",
       "cpu": 0,
-      "portMappings": [
-        {
-          "containerPort": 5000,
-          "hostPort": 5000,
-          "protocol": "tcp"
-        }
-      ],
+      "portMappings": [],
       "essential": true,
-      "command": ["server"],
+      "command": ["manage", "db", "upgrade"],
       "environment": [
@@ -70,3 +64,3 @@
           "awslogs-region": "ap-northeast-1",
-          "awslogs-stream-prefix": "server"
+          "awslogs-stream-prefix": "db-migrate"
         }
@@ -76,3 +70,3 @@
   ],
-  "family": "redash-server",
+  "family": "redash-db-migrate",
   "networkMode": "awsvpc",
@@ -111,3 +105,3 @@
       "key": "Name",
-      "value": "redash-server"
+      "value": "redash-db-migrate"
     }
```

### [事前作業] Redash 本体のイメージを更新するための PR の Approval を取得しておく

primeNumber では ECS で稼働する Redash の構成管理は Terraform で行っています。  
PR がマージされると、terraform apply が実行され、ECS のタスク定義が更新されます。

tf ファイルの管理方法はそれぞれだと思いますので、ECS のタスク定義ベースで説明します。  
以下のような差分となるようにしておきます。

```diff
@@ -4,3 +4,3 @@
       "name": "server",
-      "image": "redash/redash:10.1.0.b50633",
+      "image": "redash/redash:25.1.0",
       "cpu": 0,
```

### サービス停止

Redash のコンポーネントごとに ECS Service を定義しているため、個別に全て停止します。

公式では `docker-compose stop server scheduler ...` のように案内される部分です。

```bash
ECS_CLUSTER=redash

SERVICES=(
  redash-adhoc-worker
  redash-scheduled-worker
  redash-scheduler
  redash-server
  redash-worker
)
for SERVICE in "${SERVICES[@]}"; do
  echo "Setting desired count to 0 for $SERVICE..."
  aws ecs update-service \
    --no-cli-pager \
    --cluster "$ECS_CLUSTER" \
    --service "$SERVICE" \
    --desired-count 0
done
```

### DB のバックアップ

Redash のための DB として RDS(PostgreSQL) を利用しています。  
もしアップグレードで問題が発生した際に直前のデータで復旧できるように、明示的に手動でスナップショットを取得しておきます。

[Amazon RDS のシングル AZ DB インスタンスの DB スナップショットの作成 \- Amazon Relational Database Service](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html)

### DB のマイグレーション

[[事前作業] DB マイグレーションタスクのイメージを更新しておく](#%5B%E4%BA%8B%E5%89%8D%E4%BD%9C%E6%A5%AD%5D-db-%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%A6%E3%81%8A%E3%81%8F) で更新されたタスク定義を利用して one-shot タスクを実行します。

```bash
aws ecs run-task --cluster redash --task-definition "redash-db-migrate:N" --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxx],securityGroups=[sg-yyyyy]}"
```

### アップグレード

[[事前作業] Redash 本体のイメージを更新するための PR の Approval を取得しておく](#%5B%E4%BA%8B%E5%89%8D%E4%BD%9C%E6%A5%AD%5D-db-%E3%83%9E%E3%82%A4%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E6%9B%B4%E6%96%B0%E3%81%97%E3%81%A6%E3%81%8A%E3%81%8F) の PR をマージして、ECS のタスク定義を更新します。

### サービス再開

[アップグレード](#%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89) を実行するとタスク定義のみでなく、ECS Service も更新されるようになっています。
(Terraform の ECS Service のリソースが ECS Task のリソースに依存するようにしているため)

ECS Service が更新されることで、新しいタスク定義を利用したタスクが起動するため、サービスが再開されることになります。

### 動作確認

WebUI をぽちぽちして確認するなり、自前のチェックスクリプトを実行するなり、各々の確認方法で動作確認を行います。

## さいごに

v10.0.0 から v25.1.0 ということで大幅なアップグレードとなりましたが、特に問題なく完了しました。  
v25.1.0 が久しぶりのリリースだったのですが、それも 2025/01/09 だったということで継続的なアップデートが行われているのかは気になるところです。

(私は副業でのジョインではあるのですが) TROCCO を開発する primeNumber では、このような取り組みを行う SRE エンジニアを絶賛募集中とのことです！  
取り組んでいる内容の具体例は [2024年ふりかえり 〜SREとSecurityとEMと時々CorpIT〜](https://zenn.dev/primenumber/articles/bb5a8d0ebae4a7) などをご参照いただければと思います！

- [SREの知見を全社横断で適用し、組織の信頼性を高める Corporate SRE \- 株式会社primeNumber](https://herp.careers/v1/primenumber/mLveddJUpVFp)
- [信頼性向上が選ばれる理由になる、裁量の大きいProduct SRE【データ分析基盤総合支援SaaS TROCCO®/地方フルリモート有】 \- 株式会社primeNumber](https://herp.careers/v1/primenumber/yKLDM8pAkJjb)
