---
title: K8sのシークレット管理になぜExternalSecretやESOが必要なのか？
tags:
  - kubernetes
  - ExternalSecret
private: false
updated_at: '2024-10-16T10:40:49+09:00'
id: 1c68f122f602592470d3
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

アプリケーションエンジニア向けに、K8s のシークレット管理について説明する機会があり
その際に ExternalSecret やその管理ツール（ESO）についてわかりやすく解説するのが意外と難しいと感じました。

**ESO について書かれたとても優れた記事は他にたくさんあるのですが、その多くは K8s にすでに馴染みのある SRE 向けの内容です。**
K8s ふんわり理解のアプリエンジニア向けに「これを読め」というのはどうしても酷に思いました。そこでもう少し前段の、`なぜExternalSecretやESOが必要なのか？`という所から自分なりに説明してとっかかりを作ってみようという意図で、この記事を執筆しました。

# 書かないこと

この記事では、ESO の導入方法やその周辺のマニフェストの書き方は紹介していません。

「なぜこれが必要なのか？」「各コンポーネントの役割は？」「どのような順序で動作しているのか？」といった概念的なところにフォーカスして書いています。

# ExternalSecret がなぜ必要なのか？

## K8s Secret 単体運用のデメリット

そもそも ExternalSecret 以外に、`K8s Secret`（kind: secret）というネイティブのシークレット管理用オブジェクトがあります。

しかしながら、K8s Secret には、以下のデメリットがあります。

1. **セキュリティリスク**: k8s Secret は Base64 エンコードされているだけなのでその情報を見れるだれでもデコードして値を参照できてしまいます。そのため Git リポジトリに置くと、そのマニフェストが漏洩したときに大きな脅威になります。
2. **動的更新の難しさ**: Secret の値を変えたい時、手動でマニフェストを更新する必要があります。
3. **監査情報取得の難しさ**: その Secret にだれがいつアクセスしたかなどの監査ログを取得することが難しいです。
4. **一元管理の難しさ**: 複数の環境やクラスタで同じ秘匿情報を使用したい場合でも、マニフェストで個別に管理する必要があり、管理が煩雑になります。

特に、1.のセキュリティリスクが致命的です。これを許容できるプロジェクトはほぼ無いのではないでしょうか。

↓ Secret マニフェスト記述例

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

## ExternalSecret が解決すること

そこで `ExternalSecret` がこれらを解決します。
ExternalSecret は、k8s クラスタ外部の、専用のシークレット管理サービスに格納している機密情報を参照して K8s Secret に格納するオブジェクトです。

※ ExternalSecret を使ったとしても、アプリケーション Pod から`K8s Secret` を参照する点は変わりません。

以下のような運用上でのメリットがあります。（ほぼ先述した K8s Secret デメリットの裏返しです）

- マニフェストに Secret の値を書かなくて良いのでセキュリティ向上する。
- シークレット管理サービスの状態を定期的に確認して、更新したものを自動的に K8s Secret にも更新できる。
- 監査ログの取得を GCP や AWS に委任できる。
- Secrets 管理サービス側のアクセス制御を活用して、複数環境・複数クラスタであっても柔軟な権限管理ができる。

# ESO がなぜ必要なのか？

## シークレット管理コンポーネントの必要性

上記を踏まえて ExternalSecret を使おうと思うのですが、これ単体では運用できません。

これはわりと単純な話で、**外部 Secret 管理サービス（GCP/AWS SecretsManager 等）への接続や自動同期できるようにするには、`ESO` や `KES`（非推奨） のような シークレット管理コンポーネントを使う必要があります。**

ESO の公式ドキュメントでも以下の通り述べられています。

> The goal of External Secrets Operator is to synchronize secrets from external APIs into Kubernetes.

https://external-secrets.io/latest/

ちなみに、ESO は一般的に、Helm によりパッケージ化されたものをデプロイ・アップデートして運用します。

## 【補足】KES とは

:::note info
KES は現在非推奨となっています。
これから導入を考えている場合は ESO の把握で十分なので、この説は参考までにご覧ください。
:::

`KES`は`Kubernetes ExternalSecret`の略で、Node.js で書かれているツールです。

以下にある通り、2021 年 11 月に deprecated となり、開発は止まっています。
https://github.com/external-secrets/kubernetes-external-secrets/issues/864

代わりに後継の `ExternalSecret Operator（ESO）`が推奨されています。
ESO は Go 言語で書かれています。

## ESO の動作イメージ

ESO は以下の順序で Secret を同期しています。

1. `ExternalSecret` の内容を読み取る。ExternalSecret が参照している `SecretStore` の情報を得る
2. `SecretStore` を読み取り、外部シークレット管理サービスへの接続情報を得る
3. 2.の情報をもとに `外部シークレット管理サービス` に通信して秘匿データを取得する
4. ExternalSecret の `target` として指定した `K8s Secret` オブジェクトを作成 or 更新し、その Secret に取得した秘匿データを格納する

GoogleCloud での運用を想定してこれを図示すると、以下のようになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/120675b0-b2b9-cb4f-c4c8-fd5f9319af8c.png)

:::note info
`SecretStore` は KES 時代にはなく、ESO で新しく導入されたリソースです。
SecretStore は接続情報を定義するオブジェクトですが、KES 時代は ExternalSecret で接続情報を定義していました。
管理の柔軟性を上げる目的で、ESO で導入されました。
:::

# まとめ: 各コンポーネントの役割

まとめると、シークレット周りの各コンポーネントはざっくり以下の役割分担になっていると解釈できます。

- `K8s Secret`: Pod が参照する秘匿情報を格納する
- `ExternalSecret`: 作成する（=紐づく） K8s Secret のメタデータや、外部から実際に取得するシークレットデータ（シークレットキー）を定義する
- `SecretStore`: 外部のシークレット管理サービス（AWS/GCP Secrets Manager 等）への接続方法を定義。つまりどの外部サービスからシークレットを取得するかを指定する
- `ESO`: シークレットの継続的な同期と k8s Secret の更新をする

繰り返しになりますが、ExtenalSecrets を使うと K8s Secret が不要になるということではなく、Pod から秘匿情報を参照するのは K8s Secrets であるという点は変わりません。

このようにして、`ExternalSecret`や`ESO` は、セキュアかつ柔軟に K8s Secret の運用を実現しています。

# 参考

- https://external-secrets.io/latest/
- https://atmarkit.itmedia.co.jp/ait/articles/2209/29/news015.html
- https://sreake.com/blog/kubernetes-secret-management/
