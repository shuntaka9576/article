---
title: "AWS Cloud Development Kit (AWS CDK) v2の翻訳"
description: ""
thumbnail: ""
publish: true
tags:
  - "tech/aws/cdk"
  - "misc/読書"
---

# はじめに

[AWS Cloud Development Kit Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)の翻訳です。随時更新します。個人用のメモを含むのでご了承願います。

# Concepts

## Constructs

Constructsは、AWS CDKアプリの基本的な構成要素です。Constructは「クラウドコンポーネント」を表し、AWS CloudFormatinがコンポーネントを作成するために必要なもの全てをカプセル化します。

:::message note
Constructsは、Constructsプログラミングモデル(CPM)の一部です。これらはCDK for Terraform(CDKtf)、CDK for Kubernetes(CDK8s)、Projenなどの他のツールでも使用されます。
:::


Constructは、Amazon Simple Storage Service(Amazon S3)バケットなどの単一のAWSリソースを表すことができます。Constructは、複数の関連するAWSリソースで構成される高レベルの抽象化である場合もあります。このようなコンポーネントの例には、ワーカーキューとそれに関連づけられたコンピューティング能力、またはリソースとダッシュボードを監視するスケジュールされたジョブが含まれます。

AWS CDKには、AWS Construct Libraryと呼ばれるConstructのコレクションが含まれており、これにはすべてのAWSサービスのConstructが含まれています。

Construct Hubは、AWS、サードパーティ、オープンソースCDKコミュニティから追加のコンストラクトを発見するのに役立つリソースです。

:::message error
AWS CDK v2では、Construct基本クラスはCDKコアモジュール内にありました。CDK v2には、このクラスを含むConstructsと呼ばれる別のモジュールがあります。
:::

:::message mymemo
[Constructsと呼ばれる別のモジュール](https://github.com/aws/constructs)
:::


## AWS Construct library

AWS CDKには、AWSリソースを表すConstructが含まれるAWS Construct Libraryが含まれています。

このライブラリには、AWSで利用可能なすべてのリソースを表すConstructが含まれています。たとえば、s3.BucketクラスはAmazon S3バッケットを表し、dynamodb.TableクラスはAmazon DynamoDBテーブルを表します。

### L1 constructs

このライブラリには3つの異なるレベルの構造があり、CFNリソース(またはL1、レイヤー1の略)と呼ばれる低レベルの構造から始まります。これらの構造は、AWS CloudFormatinで利用可能なすべてのリソースを直接表します。

### L2 constructs

次のレベルのConstructであるL2もAWSリソースを表しますが、より高いレベルのインテントベースのAPIを使用します。これらは同様の機能を提供しますが、CFN Resource constructを使用して自分で作成するデフォルト、ボイラープレート、およびグループロジックが組み込まれています。AWS Constructは便利なデフォルトを提供し、それが表すAWSリソースに関するすべての詳細を知る必要性を減らします。また、リソースの操作を簡単にする便利なメソッドも提供します。たとえば、s3.Bucketクラスは、バケットにライフサイクルルールを追加するbucket.addLifeCycleRule()などの追加のプロパティとメソッドを持つAmazon S3バケットを表します。

### L3 constructs

最後に、AWS Construct Libraryには、パターンと呼ばれるL3コンストラクトが含まれています。これらの構造は、複数の種類のリソースが関係することが多い、AWSでの一般的なタスクを完了できるように設計されています。
たとえば、aws-ecs-patterns.ApplicationLoadBlancedFargateServiceコンストラクトは、Application Load Balancerを使用するAWS Fargateコンテナクラスターを含むアーキテクチャを表します。aws-apigateway.LambdaRestApiコンストラクトは、AWSS Lambda関数によってサポートされているAmazon API Gateway APIを表します。

ライブラリ操作方法の詳細や、アプリの構築に役立つConstractを見つける方法のための情報は、[API Reference](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html)を参照してください。


## Composition

Compositionは、Constructを通してより高度な抽象化を定義するための重要なパターンである。高レベルのコンストラクトは、任意の数の低レベルのコンストラクトから構成することができる。そして、それらはされに低レベルのコンストラクトから構成され、最終的にはAWSリソースから構成される。

ボトムアップの観点から、デプロイしたい個々のAWSリソースを整理するためにコンストラクトを使用する。目的に応じて、必要なレイヤー数だけ、都合のいい抽象化を使えばいい。

コンポジションでは、再利用可能なコンポーネントを定義し、他のコードと同じように共有することできる。例えば、バックアップ、グローバルレプリケーション、自動スケーリング、モニタリングなど、DynamoDBテーブルのベストプラクティスを実装するConstructを定義することができる。チームは、そのConstructを組織内の他のチームと共有することも、公開することもできる。

チームは、他のライブラリ、パッケージを使用するのと同様に、好みのプログラミング言語でこのコンストラクトを使用し、テーブルを定義し、ベストプラティスに準拠することができる。

ライブラリが更新されると、開発者は他の種類のコードで既に使っているワークフローを通じて、新バージョンのバグ修正や改良にアクセスできるようになる。

## Initialization

Constructは、Construct基底クラスを継承するクラスで実装されます。Constructを定義するには、クラスをインスタンス化します。すべてのコンストラクトは、初期化時に3つのパラメータを取ります。


### socpe

Constructの親または所有者。スタックまたは別のコンストラクトで、コンストラクトツリー内での位置を決定します。通常、スコープには現在のオブジェクトを表わすthis(Pythonではself)を渡します。

### id

このスコープ内で一意でなければならない識別子。この識別子は、現在のコンストラクト内で定義されているすべてのものの名前空間として機能します。リソース名やAWS CloudFormationの論理IDのような一意な識別子を生成するために使用されます。

### props





