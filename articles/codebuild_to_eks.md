---
title: "GitHub Actions Self-Hosted Runner + CodeBuild でマルチリージョンの EKS に安全にアクセス"
emoji: "🌍"
type: "tech"
topics: ["AWS", "EKS", "CodeBuild", "GitHubActions"]
published: false
publication_name: "primenumber"
---

## はじめに

[TROCCO](https://primenumber.com/trocco) の主要な機能は EKS Cluster 上で動いています。  
そのため、CI/CD では Private Subnet に存在する EKS(k8s apiserver) にアクセスする必要があります。  
Pull Request で k8s 関連のファイルの更新があれば CI として EKS に対して DryRun や diff 取得を行い、リリースの際には Apply を行っています。

これまでは CircleCI の [IP ranges](https://circleci.com/docs/ip-ranges/) 機能を利用して IP を固定し、そこからのアクセスを IP ベースで許可していました。  
上記の機能は有料であり、CircleCI ジョブ実行時のデータ転送量に応じて credit がかかります。  
コードベースも大きくなっており、Pull Request が更新されるたびにジョブを実行するとコードベースのクローンだけでも credit が嵩んでしまうことがありました。  
また、それを回避するために、Pull Request での CI 時に実行されるジョブを Hold し、必要なときのみ実行するようにしてみましたが、そうすると次は CI の実行漏れによって手戻りが発生してしまうことがありました。

費用を抑えつつ、CI ジョブの実行漏れを防ぐ手段はないものかと模索していたところ、GitHub Actions Self-Hosted Runner + CodeBuild に白羽の矢がたちました。  
CircleCI でのこれまでのジョブ実行時間を参考に Pull Request 更新の都度ビルドされる前提での試算を行いましたが、弊社の利用状況では移行した方が IP ranges 機能の利用がなくなる分金銭的コスト面は有利でした。

本稿では GitHub Actions Self-Hosted Runner + CodeBuild で EKS に安全にアクセスする環境を構築した手順を紹介します。

## 検討はしたものの採用しなかった案

[GitHub の IP アドレス](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/about-githubs-ip-addresses)からのアクセスを許可し、GitHub Actions を利用する方法を検討しましたが、以下の理由で採用しませんでした。
* GitHub の IP アドレスが変更されると対応が必要になってしまう
* 仮にアクセスを許可するとして、範囲が広範すぎる[^1]

## 構成の Before/After

CI ジョブの移行前、移行後の構成の概要は以下です。  
![](https://storage.googleapis.com/zenn-user-upload/eead8ad03812-20250118.png)

## 構築手順

上図の After の状態を構築します。より詳細化したものが下図です。

![](https://storage.googleapis.com/zenn-user-upload/09fa9b9e8c42-20250123.png)

TROCCO は韓国[^2]、インド[^3]でも利用可能です。 
それぞれの Region で VPC があり、その上に EKS Cluster が存在しています。  
構成のシンプルさを保つため、CodeBuild 関連のリソースもそれぞれの Region に作成していくことにしました。

### 0. AWS Connection

:::message
対象の GitHub リポジトリが Public である場合、AWS Connection の設定は必要ありません。
:::

今回の CI 対象のリポジトリは Private であるため、そのままでは CodeBuild のビルド環境でコードを取得することができません。  
AWS Connection を作成し、ビルド環境で Private なリポジトリのコードの取得を可能にします。

特にルールはありませんが、今回は GitHub Org に対して1つの Connection を作成することにしました。  
Connection に Region の概念はないため、それぞれの Region の CodeBuild から同じ接続を利用します。  
Connection の作成方法は [GitHub への接続を作成する \- デベロッパーツールコンソール](https://docs.aws.amazon.com/ja_jp/dtconsole/latest/userguide/connections-create-github.html) をご覧ください。

### 1. AWS CodeBuild

CodeBuild のビルド環境用の Private Subnet/Security Group をそれぞれ新たに作成し、その Security Group から EKS Control Plane の Security Group へのアクセスを許可します。

CodeBuild から Private Subnet に対してアクセスするためには、ビルド環境を VPC 上に作成する必要があります。  
CodeBuild のビルド用のコンピューティングリソースとして、EC2・Lambda が選択可能ですが、特定の VPC 上で動かすためには EC2[^4] を利用します。  
VPC、Subnet、Security Group を指定し、ビルドプロジェクトを作成します。  
ビルドプロジェクト作成時、前項で作成した Connection を利用すると Private リポジトリの参照も可能になります。  
ビルドプロジェクトの作成方法は [AWS CodeBuild でのビルドプロジェクトの作成 \- AWS CodeBuild](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/create-project.html) をご覧ください。

:::message
Region が異なれば同じ名前でビルドプロジェクトを作成可能ですが、GitHub Actions からはビルドプロジェクト名でトリガーすることになるため、同時にトリガーしない場合は異なる名前にした方が良いです。
:::

ビルドの Service Role に対して EKS へのアクセスを許可[^5]したら準備完了です。

### 2. GitHub Actions Self-Hosted Runner

[チュートリアル: CodeBuild がホストする GitHub Actions ランナーを設定 \- AWS CodeBuild](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/action-runner.html) を参考に進めました。

上記ドキュメントからの抜粋ですが、GitHub Actions ワークフローの YAML を以下のように設定するだけで、CodeBuild のビルドプロジェクトをビルドすることができて簡単です。

```yaml
runs-on:
- codebuild-<project-name>-${{ github.run_id }}-${{ github.run_attempt }}
```

k8s 関連のファイルの変更を含む Pull Request が更新されるたびに CI ジョブが実行されるような実装例は以下です。

:::details .github/workflows/k8s-dryrun.yml
```yaml
name: K8s Dryrun
on: [pull_request]

jobs:
  # k8s 関連のファイルに変更が含まれる場合のみ CodeBuild を利用するため。
  # 変更のあった Region の CodeBuild しか実行しないようにする。
  changes:
    runs-on: ubuntu-22.04
    permissions:
      pull-requests: read
    outputs:
      production: ${{ steps.filter.outputs.production }}
      in-production: ${{ steps.filter.outputs.in-production }}
      kr-production: ${{ steps.filter.outputs.kr-production }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            ja-production:
            - k8s/ja-production/**
            in-production:
            - k8s/in-production/**
            kr-production:
            - k8s/kr-production/**

  # For Japan
  dryrun-ja-production:
    needs: changes
    if: ${{ needs.changes.outputs.ja-production == 'true' }}
    runs-on:
      - codebuild-kubectl-to-trocco-ja-production-${{ github.run_id }}-${{ github.run_attempt }}
    steps: ...

  # For India
  dryrun-in-production:
    needs: changes
    if: ${{ needs.changes.outputs.in-production == 'true' }}
    runs-on:
      - codebuild-kubectl-to-trocco-in-production-${{ github.run_id }}-${{ github.run_attempt }}
    steps: ...

  # For Korea
  dryrun-kr-production:
    needs: changes
    if: ${{ needs.changes.outputs.kr-production == 'true' }}
    runs-on:
      - codebuild-kubectl-to-trocco-kr-production-${{ github.run_id }}-${{ github.run_attempt }}
    steps: ...
```
:::

## さいごに

k8s 関連のファイルが含まれる Pull Request の更新で自動で CI ジョブが実行されるため、ジョブの実行漏れによる手戻りは発生しなくなりました。  
また、実行回数がこれまでより増えることになったものの、かかる費用がその分増えることもありませんでした。  

(私は副業でのジョインではあるのですが) TROCCO を開発する primeNumber では、このような取り組みを行う SRE エンジニアを絶賛募集中とのことです！  
取り組んでいる内容の具体例は [2024年ふりかえり 〜SREとSecurityとEMと時々CorpIT〜](https://zenn.dev/primenumber/articles/bb5a8d0ebae4a7) などをご参照いただければと思います！

[信頼性向上が選ばれる理由になる、裁量の大きいSRE【データ分析基盤総合支援SaaS TROCCO®/地方フルリモート有】 \- 株式会社primeNumber](https://herp.careers/v1/primenumber/yKLDM8pAkJjb)


[^1]: https://api.github.com/meta
[^2]: [primeNumber、現地企業とのパートナーシップ戦略にて海外展開を韓国から本格化　現地SaaS企業・Plateer社と協業を開始 \| 株式会社primeNumberのプレスリリース](https://prtimes.jp/main/html/rd/p/000000072.000039164.html) 
[^3]: [primeNumber、データテクノロジー事業で韓国に続きインドに進出。インド現地でビジネス開発チームを発足 \| primeNumber](https://primenumber.com/news/1151)
[^4]: 執筆現在(2025/01/07)、Lambda では指定した VPC 上でビルドすることはできません。
[^5]: [IAM ユーザーおよびロールに Kubernetes API へのアクセスを付与する \- Amazon EKS](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/grant-k8s-access.html)
