---
title: '<input type="time"> を諦めない'
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "input", "time", "react", "reacthookform"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/galapagos)の 2 日目の記事です。

# <input type="time"> を使っていて困ったこと

React と React Hook Form （RHF） で `<input type="time">` を扱っていた際の話です。
フォームに使っていた `<input type="time">` が予期せぬ挙動をしていて、時間入力ができないという場面に遭遇しました。

原因は下記です。

- `<input type="time">` を DatePicker の値の一部として扱おうとしたこと
- `<input type="time">` の onChange ハンドラで受け取る値を正しく理解できていなかったこと

## `<input type="time">` の値の扱い

`<input type="time">` は、時間入力に特化した HTML の入力要素です。ユーザーは時刻を直接入力したり、ピッカーから選択したりできます。
そして、`value` / `valueAsNumber` / `valueAsDate` とそれぞれ異なる形式で値を扱うことができます。

### `value` プロパティ

`<input type="time">` の `value` プロパティは、`HH:mm`（`HH:mm:ss`） 形式の文字列を返します。
例えば、14 時 30 分の場合は `"14:30"` が返されます。

> 値は常にゼロが先行する 24 時間表記の HH:mm または HH:mm:ss に書式化された時刻です。
>
> [MDN のドキュメント](<https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time)>)

### `valueAsNumber` プロパティ

`<input type="time">` の `valueAsNumber` プロパティは、**真夜中（00:00:00.000）からの経過ミリ秒数**を返します。
例えば、14 時 30 分の場合は `14 * 60 * 60 * 1000 + 30 * 60 * 1000` つまり `52200000` が返されます。

> The algorithm to convert a string to a number, given a string input, is as follows: If parsing a time from input results in an error, then return an error; otherwise, return the number of milliseconds elapsed from midnight to the parsed time on a day with no time changes.
>
> [仕様](<https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time)>)

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
> [仕様](<https://html.spec.whatwg.org/multipage/input.html#time-state-(type=time)>)

#### タイムゾーンの注意点

ここで注意が必要なのが、**UTC として解釈される**という点です。

例えば、日本（UTC+9）で `"09:00"` と入力した場合：

- `valueAsDate` は「1970 年 1 月 1 日 **UTC 9 時**」を表す Date オブジェクトを返す
- この Date オブジェクトに対して `getHours()` を呼ぶと、ローカルタイム（JST）に変換されて **18** が返る
- 正しく 9 を取得するには `getUTCHours()` を使う必要がある

## 時間 / 分 / 秒 の入力途中の値について

ここが今回の問題の肝でした。

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

UI 上は「12」と表示されているのに、`value` を取得すると空文字列が返ってきます。

## どうしたら解決したか

上記については、RHF のスキーマ定義で、日付選択などとは別のプロパティとして扱うことで解決しました。

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

あとは、`onSubmit` ハンドラで `date`と`time` をマージしてから扱うようにしました。
（このマージも少し面倒だったので、`valueAsNumber` や `valueAsDate` だったら簡略化できたのかが気になっています。）

### おまけ

記事を書く中で、`validity.badInput` プロパティを使うと、入力が不正な状態かどうかを判定できるということを知りました。
入力途中の `--:--` は不正な値とみなされるので、`badInput` が `true` になります。
