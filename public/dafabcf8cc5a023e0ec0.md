---
title: Pulumiの気になること調べてみた
tags:
  - IaC
  - Terraform
  - Pulumi
private: false
updated_at: '2024-01-01T21:00:41+09:00'
id: dafabcf8cc5a023e0ec0
organization_url_name: null
slide: false
ignorePublish: false
---
# これは何？
インフラ構成管理ツールといえば、Terraformや、AWSならCloudFormation, CDKなどが現在一般的かと思います。
その他では、Pulumiも一時期話題になっていました。最近では2023年8月、Terraformの開発元であるHashicorp社からライセンス変更のアナウンスがあり、その影響でPulumiにも関心が集まりました。

ex: https://zenn.dev/the_exile/articles/b90fe8c5c41694

ご多分に漏れず、自分もPulumiが気になったため、個人的に気になったことをいくつか調査検証してみました。
本記事はそれを書いたものです。

# 要約
- ドリフト検出（コードと実体の差分検出）は可能。ただterraformより少し面倒。
- 無料枠はterraformより少し狭い。ただ小さいプロジェクトなら無料範囲内で十分導入できそう。
- Pulumi AIは賢くなれば結構便利そうだけど、今のところ使用感は微妙。

# Pulumiについて
Pulumiの概要については、他に優れた記事がたくさんあるため、この記事ではすごくさらっと説明します。

Pulumiは、`Terraform`や`CloudFormation`のような独自言語を使用せずとも、TypescriptやGo, Cなど幅広い言語で書くことができ、開発者フレンドリーな構成管理ツールという特徴があります。

https://www.pulumi.com/docs/languages-sdks/

また、`AWS CDK`はAWS限定ですが、Pulumiはそれに限らず様々なプロバイダ（GCP, Azure, K8s, その他クラウドに限らず色々）をサポートしていることも特徴的です。

https://www.pulumi.com/registry/

それでは早速、以下本題です。




# 気になること1: ドリフト検出

`ドリフト検出`とは、CloudFormationでよく使われている表現な気がしますが、
`構成管理コードから作成したインフラに対し、マネジメントコンソールやCLIなど、コード以外から手動変更した際に、コード側が実リソースとの差分を検出してくれるかどうか`
という認識です。

自分が現在terraformユーザなこともあり、個人的にここは検出してほしいと強く感じるところです。
terraformでは、`$ terraform plan`や`apply`実行時にドリフトを自動検出してくれるため、apply時に意図しない変更を上書きする危険がないです。

ドリフト検出はPulumiでも可能なのでしょうか？


### 検証
実際に試してみます。
今回は、ライフサイクルルールを設定したS3バケットを作成して検証しました。

```Pulumi.yaml
name: quickstart
runtime: yaml
description: Test

resources:
  my-bucket:
    type: aws:s3:Bucket
    properties:
      lifecycleRules:
        - enabled: true
          expiration:
            days: 90
          id: test-rule
          prefix: log/
          tags:
            autoclean: 'true'
            rule: log
          transitions:
            - days: 30
              storageClass: STANDARD_IA
            - days: 60
              storageClass: GLACIER
```

まずこちらを`$ pulumi up`します。

その後、デプロイされたバケットのライフサイクルルールを、AWSマネジメントコンソール上から修正してみます。
今回は、ライフサイクルルールの中身を変更しました（STANDARD_IAへの移行を30日から50日に、Glacierへの移行を60日から80日に変更）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/c809ac62-c47b-b21b-6a9a-72657f5f5339.png)


その後 `$ pulumi preview`（terraformでいうplan的な）をしてみましたが、unchangedと出て、**実体との差分は検出されませんでした。（`pulumi up`でも同様でした）**
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/0039f6cf-fb7f-5e1c-ea0e-81ad72d85f2c.png)

それでは、Pulumiではドリフト検出は不可能なのでしょうか？
調べてみたところ、検出方法は用意されているようでした。

### pulumi refreshによる検出

`$ pulumi refresh`というコマンドによりドリフト検出ができました、
**`refresh`とは、PlumiのStateと実リソースの同期を行うコマンドのようです。**

実行したところ、terrformのように詳細には表示されませんが、どこに差分があるかが出力されました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/157ef4d6-b45c-5cbf-3701-96afbbd6de24.png)

このあと、`Do you want to perform this refresh?`と聞かれるので`yes`と入力すると、Stateが更新されました。

この状態で再度、`$ pulumi preview`したところ、実体との差分が表示されるようになりました。
**※ 正確には、コードとStateの差分という認識が正しいです。**
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/43c99abc-d4a5-1afd-0c1e-c4cfb185366a.png)

また、`$ pulumi up --diff`で詳細な差分を表示してくれました。
実運用ではこちらのコマンドのほうが役立ちそうです。
（ちなみに`$ pulumi up`→`details`オプション選択でも、同じように詳細表示されます）
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/9c2f3335-f797-3a42-467b-12a482e6f81a.png)

結論、ドリフト検知方法はあるが、`$ pulumi refresh`→`$ pulumi up --diff`の２段が必要なため、terraformより少し面倒という所感でした。



# 気になること2: 無料枠の範囲
Pulumi, terraformいずれもオープンソースであるため、個人でのツール利用自体は無料です。
ただし、チーム規模が大きくなってきて、権限管理・承認ステップつきのデプロイ・変更履歴管理など、最適な制御を行いたくなるケースが多いと思います。そんな時は、それぞれの専用Cloudを利用します。

Pricingの公式ページ
- terraform cloud: https://www.hashicorp.com/products/terraform/pricing
- pulumi cloud: https://www.pulumi.com/pricing/

各クラウドの機能にも少しずつ違いがあるため単純な比較はできませんが、2023年12月現在で無料利用できる範囲だと、下記の通りでした。

|   | リソース数  | ユーザ数   |
|----|----|----|
| terraform cloud  | 500以下  | 制限なし  |
| pulumi cloud  | 約210以下  | 10人以下   |



※ pulumi cloudのリソース数は、Teamプランの150k無料クレジットで計算しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/8596ad55-85c0-8a05-4bdd-369a4aee46c5.png)


もともとterraform cloudはFreeプランの制限が、ユーザ数5人上限でした。しかし2023年5月にリソース数500に変更となり、ユーザ数制限は撤廃されました。

そのため、**リソース数・ユーザ数だけの視点でみるとpulumiの方が上限は厳しく、terraform cloudの方が気軽に導入できる印象です。**


# 気になること3: Pulumi AI
構成管理ツールの中でも、Pulumiで特徴的なのは「**Pulumi AI**」の存在だと思います。
これは、自然言語からインフラのコードを自動生成するサービスです。

Pulumi AIは2023年4月に、AIを活用した新サービス群「**Pulumi Insights**」のひとつとしてPulumi社より発表されました。

https://www.publickey1.jp/blog/23/pulumipulumi_aiawsazurecloudflarekubernetesdatadog130infra-as-code.html


すでに色々な記事でも述べられていますが、どのような使用感なのか体感したいと思い、実際に試してみました。

https://www.pulumi.com/ai

↑のページにアクセスし、画面下側の四角の枠に、言語と作りたいインフラ構成を入力します。
注意点なのが、日本語での入力できず、英語に翻訳する必要があります。自分は今回はGoogle翻訳で英語に訳して入力しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/9ec8eb3a-3725-586a-eb17-b9d858edc7b3.png)

Lambdaなど簡単なサーバーレス構成でも全然良かったのですが、ちょっと味気なかったのでEKSクラスタとそれを配置するVPC, サブネットを書いてもらいました。

言語は`yaml`を指定し、インフラ構成を英文入力すると、コードを自動生成されました。

![Pasted Graphic.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/fe660ff2-1159-c326-36c2-17b62e02a002.png)

これをそのままコピペして、`$ pulumi up`したところ、エラーが出てデプロイできませんでした。

![Pasted Graphic 2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/e74c46c4-0c66-e830-b27c-9231435ce737.png)


出てきたエラー文を、Pulumi AIにコピペして入力すると、修正したコードを提示してくれました。
ここでは、他リソースやoutputsで参照するプロパティ名が、Pulumiでサポートされていないものだったようです。
![Pasted Graphic 3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/9631e459-e955-947e-cb7a-ca7d6cfcd270.png)

以後省略しますが、これを再度pulumi upしてもエラーとなり、そのエラーメッセージからPulumi AIに修正を提示してもらい、、ということが3回ほど続きようやくデプロイできました。

リージョン指定やネットワークの要件など全くしていなかったのでこちらのプロンプトも雑すぎたと反省しつつ、
AI側も存在しないプロパティを提示してきたりした（pulumiのコード仕様も最近で大きく変わっているため？）ので、**現時点では使用感は微妙でした**。
複雑な構成であれば、最初からドキュメント見ながら書いたほうが早いかもしれません。

これから徐々にサービスが安定版に入っていき、ユーザ数も増えていけば、より正確なコードを返してくれるようになると期待しています。


# 最後に

最近個人的に気になっていたPulumiについて、調べてみました。今回手を動かして検証したので、他ツールとの使用感の違いを体感できてよかったです。

ツールごとに設計思想が違うので単純比較できませんが、だからこそPulumiが今後成長していき、プロジェクトの特性に応じてterraform等との使い分けができるようになったりすると面白いなと思います。

# 参考
- https://tech.guitarrapc.com/entry/2019/12/23/000000
- [ドキュメント: Patterns for Drift Detection with Pulumi](https://www.pulumi.com/blog/patterns-drift-detection/)
