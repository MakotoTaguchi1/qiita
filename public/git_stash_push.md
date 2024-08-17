---
title: 【備忘】gitの退避にはgit stash pushが良さそう
tags:
  - Git
private: false
updated_at: "2024-08-17T09:29:33+09:00"
id: 4671a28266b85d801765
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

git で現在の作業を退避（stash）するとき、
今までは `$ git stash -u` や `git stash save {コメント}`を使っていました。

しかしながら、退避コメント付与したい時や特定ファイルのみ退避したいときに、
オプションを忘れてググったり、うまくできずに手間取ることがよくありました。

そんな中、`git stash push`というコマンドがあることを最近知り、
結構便利に感じたので自分用の備忘もかねて紹介します。

# 使い方

## 一番使う退避コマンド

個人的には、今後退避時はこのコマンドだけ使うことになりそうです。

```bash
# 退避対象ファイルをgitステージしておく
$ git stash push -m {コメントをここに} -S
```

## 少し詳しく

これだけだと寂しいので（自分もオプションの意味とか忘れそう）、
`git stash push`の使い方を少し説明します。

### 基本文法

```bash
# 追跡済みのファイル と ステージ済みのファイルを　退避
$ git stash push

# 追跡やステージ関わらず全ファイルを退避（.gitignoreされていないファイルも含む）
$ git stash push -a

# 追跡やステージ関わらず全ファイルを退避（.gitignoreされていないファイルは除く）
$ git stash push -u

#  特定のファイルのみ退避
$ git stash push -- {ファイルパス}
```

### コメント付与や特定ファイルのみ退避したいとき

ファイルパスの指定や変更分の一部ファイルだけを退避したい時に、上記だと面倒です。
そこで、退避したいファイルをステージした後、stash するのが一番楽だと思いました。

```bash
# 退避したいファイルをステージに追加 （自分はVSCodeでやる）
# cliでやるなら以下
$ git add file1 file2 file3

#  ステージしたファイルをすべて退避
$ git stash push -S

# コメントも残したい時
$ git stash push -m {コメントをここに} -S
```

ちなみに`-S`ではステージ済みであれば、`-u`（--include-untracked）をつけなくても未追跡ファイルも退避してくれます。

# まとめ

`$ git stash push -m {コメントをここに} -S`

stash はよく使うけど毎回オプションを忘れてしまうという方はこれがオススメです！

# 参考

https://git-scm.com/docs/git-stash#_commands

https://www.webdesignleaves.com/pr/plugins/git-stash.html
