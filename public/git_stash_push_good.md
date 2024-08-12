---
title: git stash pushが良さそう
tags:
  - Git
private: false
updated_at: "2024-07-31T18:36:33+09:00"
id: 0e67619abe7efb719f59
organization_url_name: null
slide: false
ignorePublish: true
---

# はじめに

# 本文

使い方

```bash
# 追跡済みのファイル と ステージ済みのファイルを　退避
$ git stash push

# 追跡やステージ関わらず全ファイルを退避（.gitignoreされていないファイルも含む）
$ git stash push -a

# 追跡やステージ関わらず全ファイルを退避（.gitignoreされていないファイルは除く）
$ git stash push -u

#  特定のファイルのみ退避
$ git stash push -- <ファイルパス>
```

ただし、ファイルパスをいちいち指定するのが面倒だったり、変更した全体のうちのいくつかのファイルを退避したい時に、
上記だと不便だと思いました。

```bash
# VSCodeで、退避したいファイルをステージに追加
# または、cliでやるなら以下
$ git add file1 file2 file3

#  ステージしたファイルをすべて退避
$ git stash push -S

# コメントも残したい時
$ git stash push -m 'コメントをここに' -S


```

ちなみに`-S`では、`-u`（--include-untracked）がなくても、ステージ済みの未追跡ファイルも退避してくれます。

# まとめ

# 参考
