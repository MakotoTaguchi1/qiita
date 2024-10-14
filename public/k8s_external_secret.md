---
title: k8s_external_secret
tags:
  - k8s
  - ExternalSecret
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# はじめに

ExternalSecret やその管理ツール（ESO）について、アプリケーションエンジニア向けに説明する場面がありました。
このあたりシークレット管理について書かれたとても優れた記事はあるのですが、その多くは K8s にすでに馴染みのある SRE 向けの内容であり、
K8s ふんわり理解のアプリエンジニア向けに「これを読め」というのがどうしても酷に感じたので、
もう少し前段の、なぜこのような構成になっているのか？という所から自分なりに書いてみようと思い、執筆してみました。

# External Secrets がなぜ必要なのか？

Secret 管理であれば、K8s Secret（kind: secret）というネイティブのシークレット管理リソースがあります。
しかしながら、K8s Secre には、以下のデメリットがあります。
特に、1.のセキュリティリスクが致命的です。これを許容できるプロジェクトは少ないのではないでしょうか。

1. セキュリティリスク: k8s Secret は Base64 エンコードされているだけなのでその情報を見れるだれでもデコードして値を参照できてしまいます。そのため Git リポジトリに置くと、そのマニフェストが漏洩したときに大きな脅威になります。
2. 動的更新の難しさ: Secret の値を変えたい時、手動でマニフェストを更新する必要がある。
3. 監査情報取得の難しさ: その Secret にだれがいつアクセスしたかなどの監査ログを取得することが難しい。
4. 一元管理の難しさ: 複数の環境やクラスタで同じ秘匿情報を使用したい場合でも、マニフェストで個別に管理する必要があり、管理が煩雑になります。

↓Secret 定義例

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
type: Opaque
data:
  username: YWRtaW4= # "admin" をbase64エンコード
  password: UEBzc3dvcmQ= # "P@ssword" をbase64エンコード
  # デコードして簡単に値が取得できてしまう
```

そこで External Secrets がこれらを解決します。
External Secrets は、k8s クラスタ外部の専用シークレット管理サービスに格納している機密情報を参照して K8s Secret に格納するオブジェクトです。
以下のように、運用上でのメリットがあります。

- マニフェストに Secret の値を書かなくて良いのでセキュリティ向上する。
- シークレット管理サービスの状態を定期的に確認して、更新したものを自動的に K8s Secret にも更新できる。
- 監査ログの取得を GCP や AWS に委任できる
- Secrets 管理サービス側のアクセス制御を活用して、複数環境・複数クラスタであっても柔軟な権限管理ができる。

# 外部シークレット管理ツール

## 外部シークレット管理ツールの必要性

では、ExternalSecrets を使おうと思うのですが、残念ながらこれ単体では運用が難しいです。
そこで、ESO のような外部シークレット管理ツールを使います。

## KES とは

Kubernetes External Secrets（KES）で、Javascript で書かれている。

以下にある通り、2021 年 11 月に deprecated となり、開発は止まっています。
https://github.com/external-secrets/kubernetes-external-secrets/issues/864

代わりに後継の External Secrets Operator（ESO）が推奨されています。
ESO は Go 言語で書かれています。

# ESO のアーキテクチャ

## 各コンポーネントの役割

ざっくり、Secret に関係する各コンポーネントはそれぞれ以下の役割を担っています。

- ExternalSecret: 作成する K8s Secret のメタデータや、外部から実際に取得するシークレットデータ（シークレットキー）を定義する。
- SecretStore: 外部のシークレット管理サービス（AWS/GCP Secrets Manager 等）への接続方法を定義。つまりどの外部サービスからシークレットを取得するかを指定。
- ESO: シークレットの継続的な同期と k8s Secret の更新を担う。
- K8s Secret: Pod が参照する秘匿情報を格納する

## ESO の取得順序

ESO は以下の順序で Secret を同期しています。

1. ExternalSecret の内容を読み取る
2. ExternalSecret で参照している SecretStore の情報を読み取る
3. SecretStore を読み取り、その情報をもとに外部シークレット管理サービスに通信して秘匿データを取得する
4. ExternalSecret の target として指定した K8s Secret オブジェクトを作成 or 更新し、その Secret に取得した秘匿データを格納する

上記を図示すると、以下のようになるかと思います。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/828bf640-a1bd-87c0-aa43-4b77df3d6186.png)
