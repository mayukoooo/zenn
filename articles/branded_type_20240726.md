---
title: "意味のタグ付けする Branded Typeで型の一意性を守るテクニック" # 記事のタイトル
emoji: "🚀" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["フロントエンド", "typescript", "brandedtype"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---
## 1. TypeScript の問題点
TypeScriptは、型システムとして構造的型付けを採用しています。
構造的型付けとは、「型の名前ではなく、型の構造に基づいて型の互換性を判断する仕組みのこと」です。

構造的型付けは、型の柔軟性やコードの再利用性を向上させるメリットがある一方で、**意図せず型の互換性を生じさせてしまう可能性があります😱**

例えば、以下のコードを見てみましょう。
```typescript
type User = { 
  id: string
  name: string
}
type Post = {
  id: string
  name: string
}

const post: Post = {
  id: "1",
  name: "post"
}

const print = (user: User) => {
  console.log(user)
}

// User 型と Post 型は構造が同じであるため、同じ型として扱われる
// そのため、print 関数に post を渡すことができる
print(post)
```

この例では、`User` 型と `Post` 型が同じ構造（`id` と `name` プロパティを持つ）であるため、TypeScriptはこれらを互換性のある型とみなします。

この結果、`print` 関数に `Post` 型のオブジェクトを渡してもコンパイルエラーが発生しません🤔

このような問題を解決するために、**branded type** というテクニックがあります。


## 2. branded typeとは
> ### ブランド型
> ブランド型(または幽霊型(phantom type)、opaque type)は、型を区別するためのプロパティを型に持たせることで、その型を明確に区別するデザインパターンです。これは、型にメタデータのようなタグをつけることで、構造的には同じであっても型と型を区別できるようにします。
>
> 出典：　[サバイバルTypeScript「TypeScriptと構造的型付け」ブランド型　2024/08/02閲覧](https://typescriptbook.jp/reference/values-types-variables/structural-subtyping#%E3%83%96%E3%83%A9%E3%83%B3%E3%83%89%E5%9E%8B)

branded typeは、構造的型付けの問題を解決するための手法の1つです。

**型の意味をタグ付けし構造を変えることで、タグ以外の構造が同等なものとの互換性を切る**ことができます。


以下のコードを見てみましょう。
```typescript
type UserId = string & { __brand: "UserId" }  // __brand というダミープロパティに "UserId" キーを付与
type PostId = string & { __brand: "PostId" }　// __brand というダミープロパティに "PostId" キーを付与

type User = {
  id: UserId
  name: string
}
type Post = {
  id: PostId
  name: string
}

const createUserId = (id: string): UserId => id as UserId
const createPostId = (id: string): PostId => id as PostId

const post: Post = {
  id: createPostId("1"),
  name: "post"
}

const print = (user: User) => {
  console.log(user)
}

// これでコンパイルエラーが発生する
print(post) // Error: Argument of type 'Post' is not assignable to parameter of type 'User'.
```
このコードは、`UserId` 型と `PostId` 型それぞれに `__brand` というプロパティを追加し、プロパティのキーとして、それぞれの型の意味を示す文字列（`"UserId"`, `"PostId"`）を指定しています。

このように、**`UserId` 型と `PostId` 型にはそれぞれ固有の意味のタグが付与されているため、`User` 型と `Post` 型は互換性がなくなり、コンパイルエラーが発生する**ようになります。

コンパイル時の安全性をチェックすることでデバッグ時間を短縮し、実行時のエラーを防ぐことができます。尚、branded type は実際のランタイム（プログラムが実行されるとき）には存在せず、コンパイル時にのみ存在しています。

## 3. branded typeの実装方法
### 3.1 シンボルを使う
ハードコードされた`__brand`というプロパティを使用すると、 ブランド・プロパティが[インテリセンス](https://e-words.jp/w/%E3%82%A4%E3%83%B3%E3%83%86%E3%83%AA%E3%82%BB%E3%83%B3%E3%82%B9.html)に表示されてしまい、開発者に混乱を招く可能性があります。

そこで、一意のシンボルを使用し、ブランド定義をモジュール内にカプセル化する方法があります。

以下のコードを見てみましょう。
```typescript
// プロパティへの読み取りアクセスを防ぐために、Brand ユーティリティを独自のファイルに記述する
declare const __brand: unique symbol
export type Brand<K, T> = K & { [__brand]: T }
```
```typescript
type UserId = Brand<string, "UserId">
```
ここでは、`unique symbol` を使用して、ブランド・プロパティを定義しています。

**ブランド・プロパティに固有のシンボルを使用することで、インテリセンスからプロパティが隠され、混乱を避けることができます。**

### 3.2【疑問】ブランド内のキーが重複する場合もあるのでは...🤔？
branded type をプロジェクトで運用するにあたって、疑問に思ったことがありました。

それは、「**ブランド内のキーが将来的に被る可能性がなくはないよな...?**」という疑問です。

例えば、以下のコードでは `UserId` と `UserId2` が同じキー `"userId"` を使用しているため、同じ型として扱われてしまいます。 

```typescript
declare const __brand: unique symbol
type Brand<B> = { [__brand]: B }
export type BrandedType<Type, State extends string> = Brand<State> & Type

// UserId と UserId2 は同じブランド・キー "userId" を使用しているので同じ型として扱われる
export type UserId = BrandedType<string, "userId">
export type UserId2 = BrandedType<string, "userId">

const userId: UserId = "userId" as UserId
const userId2: UserId2 = "userId" as UserId2

const print = (id: UserId2) => {
  console.log(id)
}

// コンパイルエラーが発生しない
print(userId2)
print(userId)
```
上記のように意図せず型の一意性が損なわれてしまうため、運用上の問題が発生する可能性があります🤔

### 3.3【解決】ブランド内のキーを完全に一意にする
そこでフロントエンドチームでは、ブランド内のキーが重複する可能性を完全に排除するために、都度ブランドを定義する方法を採用することにしました。
```typescript
export type BrandedType<Type, Id extends symbol> = Type & { [K in Id]: never }

declare const brand: unique symbol
export type UserId = BrandedType<string, typeof brand>

const userId: UserId = "userId" as UserId
```
こうすることで型の一意性を保つことができると同時に、管理する手間を減らすことができます。

(都度ブランドを定義することになるので少し冗長に感じるかもしれません>< 何かアイデアがあればご教示いただけますと幸いです🙏)

## 4. まとめ
この記事では、branded type とその実装方法について紹介しました。

構造が同じ型でも別々の型として区別できるので、より厳密にコンテキストに沿った型の使用が制約できます。便利なテクニックですね👏

今回 branded type を導入するにあたって、フロントエンドチームに知見のシェアと運用方法を提案してくださった [ShebangDog](https://github.com/ShebangDog) さんに感謝申し上げます。ありがとうございました！


## 参考文献
- [Improve Runtime Type Safety with Branded Types in TypeScript](https://egghead.io/blog/using-branded-types-in-typescript)
- [TypeScriptと構造的型付け](https://typescriptbook.jp/reference/values-types-variables/structural-subtyping#%E6%A7%8B%E9%80%A0%E7%9A%84%E5%9E%8B%E4%BB%98%E3%81%91%E3%81%AE%E6%B3%A8%E6%84%8F%E7%82%B9)
- [Branded Type について理解する](https://qiita.com/yamatai12/items/e972833b3105883202a3)