---
title: VPC Lambdaのセキュリティベストプラクティス
emoji: 🔌
type: tech
topics:
  - aws
  - lambda
  - vpc
published: false
---

# VPC Lambdaの仕組み

AWSでは、Lambda関数がVPC内のリソースにアクセスできるよう設定することができます。この機能は、Lambda関数の実行時にENIをVPC内に自動作成することで実現されています。

![VPC Lambdaのアーキテクチャ](/images/lambda-vpc-security/architecture.png)
*[[発表] Lambda 関数が VPC 環境で改善されます | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/announcing-improved-vpc-networking-for-aws-lambda-functions/) より引用*

そのため、関数をVPCにアタッチするには、関数の実行ロールに以下のアクセスポリシーを追加するか、同等のAWS管理ポリシー[`AWSLambdaVPCAccessExecutionRole`](https://docs.aws.amazon.com/ja_jp/aws-managed-policy/latest/reference/AWSLambdaVPCAccessExecutionRole.html)を追加しなければならないとAWS公式ドキュメントに書かれています。

```json
{
  "Effect" : "Allow",
  "Action" : [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DescribeSubnets",
    "ec2:DeleteNetworkInterface",
    "ec2:AssignPrivateIpAddresses",
    "ec2:UnassignPrivateIpAddresses"
  ],
  "Resource" : "*"
}
```

https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html#configuration-vpc-permissions

# 問題点

上記のポリシーにはセキュリティ上の懸念があります。関数の実行ロールに上記のアクセスポリシーを追加するということは、Lambda関数が上記の操作を自由に行えるということです。
すなわち、**Lambda関数のコードからENIの作成や削除、プライベートIPの付け替えが自由にできるようになってしまいます**。これは最小権限の原則の観点から望ましくないでしょう。

# 解決策

この問題を解決するのに必要なアクセス許可は、**LambdaサービスはENIの作成を行うことができるが、Lambda関数の実行環境からはそれができない**、といったものです。これは以下のポリシーにより実現できます。

```json
{
  "Effect" : "Allow",
  "Action" : [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DescribeSubnets",
    "ec2:DeleteNetworkInterface",
    "ec2:AssignPrivateIpAddresses",
    "ec2:UnassignPrivateIpAddresses"
  ],
  "Resource" : "*",
  "Condition": {
    "Null": {
      "lambda:SourceFunctionArn": "true"
    }
  }
}
```

このポリシーではIAM条件キー`lambda:SourceFunctionArn`を使用しています。
Lambda関数の実行環境からAWS APIにリクエストを送信すると、関数のARNが`lambda:SourceFunctionArn`としてコンテキストに挿入されます。一方でLambdaサービスがENIの作成を行う際には、`lambda:SourceFunctionArn`はセットされません。この挙動は以下のページに明記されています。

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/permissions-source-function-arn.html

上記のポリシーではこの仕様を利用し、`lambda:SourceFunctionArn`がセットされていない場合のみENIの作成を許可しています。

# 参考

この記事は、以下の公式ドキュメントの節「Security best practices」の内容に沿っています。

https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html#configuration-vpc-best-practice
