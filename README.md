# qiita

https://github.com/increments/qiita-cli

最初に `qiita` ディレクトリで依存関係をインストールしてください。
公式の CLI は `@qiita/qiita-cli` です。未インストールの状態で `npx qiita` を実行すると、別の `qiita` パッケージを取得して失敗します。

```bash
cd qiita
npm ci
```

初回のプレビュー前に、[Qiita の設定画面](https://qiita.com/settings/tokens/new)でアクセストークンを発行し、`read_qiita` と `write_qiita` を有効にしてログインしてください。トークンはリポジトリに保存しません。

```bash
npx qiita login # 表示された入力欄にトークンを入力
```

`credentials.json` が見つからないエラーは、CLI にまだログインしていない場合に発生します。ログイン後に `npx qiita preview` を再実行してください。

```bash
# 記事の作成
$ npx qiita new {記事のベース名} # .md は付けない
$ npx qiita new book_2026_01    # public/book_2026_01.md を作成

# 記事のプレビュー
$ npx qiita preview

# 記事ファイルの同期
# Qiita 上で更新を行い、手元で変更を行っていない記事ファイルのみ同期されます。
$ npx qiita pull
# 強制的に Qiita 上の内容を記事ファイルに反映
$ npx qiita pull --force

# 記事の公開
$ npx qiita publish {記事のベース名}
```
