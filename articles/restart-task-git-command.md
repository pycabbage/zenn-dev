---
title: "コミット履歴を保ちつつ、作業をやり直す"
emoji: "📦"
type: "tech"
topics: ["git"]
published: false
---

## やりたいこと

- トピックブランチでの作業を全て削除し、やり直したい
- 一応コミット履歴は保持しておきたい

## コマンド

```shell
# 0. トピックブランチにいることを確認
git branch --show-current
# もしくは git switch topics/01-***

# 1. 最新のdevelopを取得
git fetch origin

# 2. マージコミットの準備
git merge --no-commit -s ours origin/develop

# 3. インデックスをdevelopに完全置換
git read-tree --reset -u origin/develop

# 4. コミットメッセージを指定せずコミット
git commit --no-edit

# 5. 検証
#   HEADとorigin/developの差分を確認
git diff HEAD origin/develop
#   何も出力されなければ成功
#   もしくは、ツリーオブジェクトのハッシュを比較
git rev-parse HEAD^{tree}
git rev-parse origin/develop^{tree}
#   2つのハッシュが同じなら完全に同じ状態
```

このコマンドにより、「トピックブランチの試行錯誤を記録として残しつつ、クリーンな状態にリセットする」というニーズを満たすことができます。

