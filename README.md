# qiita

https://github.com/increments/qiita-cli

```bash
# 記事の作成
$ npx qiita new {記事のベース名}

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
