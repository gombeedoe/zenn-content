---
title: "paginationのためのURL paramをZodで数値にする"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Zod"]
published: false
---

# Zod で pagination のための param を Zod で検証したい動機

URL はユーザーによって如何様にも変更できる内容なので、searchParams や dynamicParams は、
基本的に Zod で動的に検証しても良いと思っています。

その中でもページネーションを扱う時の URL の場合は、`"1"`、`"2"`、`"30"`のような numeric(数値的な) な文字列を想定すると思いますが、
`"hoge"`や`?page=`のように numeric ではない文字列や空文字の場合もあります。

想定していない文字列が入力された時の挙動をそれぞれエラーハンドリングをしていると骨が折れますし、
numeric ではない場合は数値`1`に丸めてしまいたいという欲求があり、Zod で実現できないか検証しました。

## 結論

```ts
const pageParam = z.coerce.number.min(1).catch(1);
const pageNumber = pageParam.parse(xxx);
```

上記のようなルールにすると、numeric ではない文字列や空文字が入力されたとしても`1`に丸めることができ、
numeric な文字列は数値に変換して、pagination のロジックで使用することができました。

## 各メソッドの詳細

### `coerce.number`

`coerce.number`では、parse 時に number 型に強制します。その場合、`NaN`になるような値(`hoge`など)はエラーがスローされます。

[coercion-for-primitives](https://zod.dev/?id=coercion-for-primitives)

### `.min(1)`

`.min(1)`は`1`未満の数値でエラーがスローされます。
pagination で`0`や`-1`など負の値は不要で、ロジック内でのバグの原因になりそうですので除外しています。

### `.catch(1)`

`.catch()`は parse でエラーになる値の場合に、エラーをスローする代わりに指定した値を返します。
`.catch(1)`とすることで、下記のエラーになる場合に数値`1`の返却を強制します。

- `hoge`など`NaN`になる文字列
- `0`(`.min(1)`でエラー)になる空文字

[catch](https://zod.dev/?id=catch)
