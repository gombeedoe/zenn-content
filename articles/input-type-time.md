---
title: '<input type="time"> を諦めない'
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "input", "time", "zod", "reacthookform"]
publication_name: "galapagos"
published: true
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/galapagos)の 2 日目の記事です。

## <input type="time"> の使いどころと仕様

フォームの時間入力要素として、`<input type="time">` を使うと、`type="text"` よりも入力値を限定できるようになり、入力値の整合性の考慮がグッと減ります。

`type="text"`を使った場合、下記のようなことを考えなければなりません。

- 時間 / 分 / 秒 をどのような形式で入力してもらうか？
  - 区切り文字は何か？別々の input 要素にするか？
- そもそも "秒" を許容するか？
- `25:61` は有効な時間か？

などなど、`<input type="time">`を使わない場合はそれらをイチから考える必要があります。

`<input type="time">`を使うと、時間は`23`まで、分は`59`までの入力になり、
全角文字入力のことも考慮する必要がなくなります。

しかし、`<input type="time">`は時間 / 分 / 秒と、同じ要素の中でフォーカスの移動があったり、未入力状態が「`--:--`」になっていたりと、どのような挙動をするのかパッと見わからないことがあります。

この記事では、`<input type="time">`で取得できる値と、未入力時（入力途中）の挙動について見ていきたいと思います。

## `<input type="time">` の値の扱い

`<input type="time">` は、時間入力に特化した HTML の入力要素です。ユーザーは時刻を直接入力したり、ピッカーから選択したりできます。
そして、`value` / `valueAsNumber` / `valueAsDate` とそれぞれ異なる形式で値を扱うことができます。

### `value` プロパティ

`<input type="time">` の `value` プロパティは、`HH:mm` / `HH:mm:ss` 形式の文字列を返します。
例えば、14 時 30 分の場合は `"14:30"` が返されます。

> 値は常にゼロが先行する 24 時間表記の HH:mm または HH:mm:ss に書式化された時刻です。
>
> [MDN のドキュメント](https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time))

### `valueAsNumber` プロパティ

`<input type="time">` の `valueAsNumber` プロパティは、**真夜中（00:00:00.000）からの経過ミリ秒数**を返します。
例えば、14 時 30 分の場合は `14 * 60 * 60 * 1000 + 30 * 60 * 1000` つまり `52200000` が返されます。

> The algorithm to convert a string to a number, given a string input, is as follows: If parsing a time from input results in an error, then return an error; otherwise, return the number of milliseconds elapsed from midnight to the parsed time on a day with no time changes.
>
> [仕様](https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time))

値が未入力の場合は `NaN` が返されるため、`isNaN()` などで入力の有無を判定できます。

### `valueAsDate` プロパティ

`<input type="time">` の `valueAsDate` プロパティは、`Date` オブジェクトを返します。このオブジェクトの日付部分は基準日（**UTC の** 1970 年 1 月 1 日）で、時刻部分は入力された時間になります。

例えば、14 時 30 分の場合は以下と同等のオブジェクトが返されます：

```javascript
new Date(Date.UTC(1970, 0, 1, 14, 30, 0));
// Thu Jan 01 1970 23:30:00 GMT+0900 (日本標準時)
```

> The algorithm to convert a string to a Date object, given a string input, is as follows: If parsing a time from input results in an error, then return an error; otherwise, return a new Date object representing the parsed time in UTC on 1970-01-01.
>
> [仕様](https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time))

#### タイムゾーンの注意点

ここで注意が必要なのが、**UTC として解釈される**という点です。

例えば、日本（UTC+9）で `"09:00"` と入力した場合：

- `valueAsDate` は「1970 年 1 月 1 日 **UTC 9 時**」を表す Date オブジェクトを返す
- この Date オブジェクトに対して `getHours()` を呼ぶと、ローカルタイム（JST）に変換されて **18** が返る
- 正しく 9 を取得するには `getUTCHours()` を使う必要がある

## 時間 / 分 / 秒 の入力途中の値について

`<input type="time">` では、**時間だけ入力して分がまだ未入力の状態**だと、`value` は空文字列になります。

（`valueAsNumber` の場合は`NaN`、`valueAsDate`の場合は`null`）

### 実際の挙動

| ユーザー操作         | UI 表示 | `value` の値 |
| -------------------- | ------- | ------------ |
| 初期状態             | `--:--` | `""`         |
| 「1」を時間に入力    | `01:--` | `""`（空！） |
| 「2」を時間に入力    | `12:--` | `""`（空！） |
| 「3」「0」を分に入力 | `12:30` | `"12:30"`    |
| delete を押下        | `12:--` | `""`（空！） |
| 全削除               | `--:--` | `""`         |

UI 上は「12」と表示されていますが、`value` を取得すると空文字列が返ってきます。これは、入力途中の値が有効な時間として解釈できないためです。

[`validity.badInput`](https://developer.mozilla.org/ja/docs/Web/API/ValidityState/badInput) プロパティを使うと、入力が有効かどうかを判定できます。入力途中の `--:--`や`12:--` は無効な値とみなされるので、`badInput` が `true` になります。

## おまけ: スキーマ定義するときのおすすめ

React Hook Form & Zod などで時間を`<input type="time">`で入力させたい場合で、日付を一緒に扱う必要がある場面があるかと思います。その場合は、無理にプロパティを一緒にして扱わずに、分けた方が良いです。

日付は DatePicker ライブラリによって返り値が Date オブジェクトだったり、独自のフォーマットだったりとするので、それぞれの型を別々に定義するのが良いかと思います。

```ts
// 例: スキーマ定義
const schema = z.object({
  date: z.date(),
  time: z.union([
    z.literal(""), // 未入力 or validity.badInput状態を許容
    z.iso.time(),
  ]),
});
```

あとは、`onSubmit` ハンドラで `date`と`time` をマージする方が各 UI の state に悩むことが少ないと思います。
