---
title: "コミット履歴を保ちつつ、作業をやり直す"
emoji: "🦀"
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
```
