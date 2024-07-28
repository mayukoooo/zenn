---
title: "branded type" # 記事のタイトル
emoji: "🤔" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["フロントエンド", "TypeScript"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---
## 背景
書こうとおもった理由を書く

## TypeScript の問題点
下記のコードでは、引数に `User` 型を受け取る `displayUser` 関数に `Post` 型の `post` を渡していますが、コンパイルエラーが発生しません。

これは、TypeScript が構造的部分型を採用しているため、`User` 型と `Post` 型が構造的に互換性があると判断されているからです。
```typescript
type User = {
  id: string
  name: string
}
type Post = {
  id: string
  name: string
}

const post = {
id: "1",
name: "post"
} as Post

const displayUser = (user: User) => {
  console.log(user)
}

// Use 型と Post 型は構造が同じであるため、同じ型として扱われる
// そのため、displayUser 関数に post を渡すことができる
displayUser(post)
```
## branded typeとは
簡単なコードの紹介
メリット
前提知識（構造

## branded typeの実装方法
シンボルを使う
プロパティキーが被る場合もあるよね。。。？

## まとめ
感想と謝辞

## 参考文献