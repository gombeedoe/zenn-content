---
title: "shadcn/ui(Radix Primitives)のScrollAreaでmax-heightと可変の高さを設定する"
emoji: "🦒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "shadcnui", "radix", "radixui"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー2024](https://qiita.com/advent-calendar/2024/galapagos)の15日目の記事です。

日頃からお世話になっているshadcn/uiですが、時々痒いところに手が届かない場面があり、カスタマイズして利用しています。

今回は、[Scroll Area](https://ui.shadcn.com/docs/components/scroll-area)について、`max-height`を指定したい場面があったのでその実装方法を紹介します。shadcn/uiのScroll-areaは[Radix PrimitivesのScroll Area](https://www.radix-ui.com/primitives/docs/components/scroll-area)から作られています。

https://ui.shadcn.com/docs/components/scroll-area
https://www.radix-ui.com/primitives/docs/components/scroll-area

## 完成したUI

下記のUIでは以下を満たしています。

- `<ScrollArea />`に`max-height`を当てていて、固定の`height`は当てていない。
- `max-height`の値になるまでは、コンテンツによって高さが可変である。
- `max-height`を超えたらscroll可能になる。
- scroll可能になったら`<ScrollBar />`を表示する

![完成UI](https://storage.googleapis.com/zenn-user-upload/1864d3ec4968-20241213.gif)

## `<ScrollArea />`に`max-height`を有効にする

残念ながら、shadcn/uiのデフォルトの構成では`<ScrollArea className="max-h-72" />`とするとスクロールができなくなります。

ただし、もともとの[Radix Primitivesのissue](https://github.com/radix-ui/primitives/issues/2307#issuecomment-1654571158)を見ると、`<ScrollArea.Viewport />`に`max-height`を指定するとスクロール可能になるそうで、実際にshadcn/uiの構成のViewportに指定したらスクロール可能になりました。

https://github.com/radix-ui/primitives/issues/2307#issuecomment-1654571158

しかし、ScrollAreaを各所で実際に使用しており、`<ScrollAreaPrimitive.Viewport />`を別コンポーネントとして`export`したり、Viewport用のclassName`props`や`slot`分割はしたくない動機がありました。

そこで参考になったのが下記のissueコメントでした。

https://github.com/shadcn-ui/ui/issues/296#issuecomment-2063777401

```tsx
const ScrollArea = React.forwardRef<
  React.ElementRef<typeof ScrollAreaPrimitive.Root>,
  React.ComponentPropsWithoutRef<typeof ScrollAreaPrimitive.Root>
>(({ className, children, ...props }, ref) => (
  <ScrollAreaPrimitive.Root
    className={classX('flex flex-col relative overflow-hidden', className)}
  >
    <ScrollAreaPrimitive.Viewport
      ref={ref}
      className="flex-grow w-full rounded-[inherit]"
      {...props}
    >
      {children}
    </ScrollAreaPrimitive.Viewport>
    <ScrollBar />
    <ScrollAreaPrimitive.Corner />
  </ScrollAreaPrimitive.Root>
))
```

このコメントでは以下の2つの変更により、

- `Root`に`flex flex-col`を指定
- `Viewport`に`flex-grow`を指定（`h-full`から変更）

これにより、`Root`明示的に`height`を渡さなくても`Viewport`が`Root`に合わせた高さになってくれます。

デフォルトの`Viewport`が`h-full`の場合、`Root`の`height`を指定しないと`Viewport`がコンテンツの高さに合わせられてしまい、`overflow-y: scroll;`が機能しなくなります。

これで、`<ScrollArea className="max-h-72" />`に指定することで、`max-height`以上になった時にscrollになることができました。

## UXを改善する

このままでも実用可能なのですが、スクロール可否の状態を視覚的にユーザーに知らせたくなります。

`Root`のAPIに`type`があります。デフォルは`"hover"`になっており、hoverするまでスクロールの可否は分かりません。

> Describes the nature of scrollbar visibility, similar to how the scrollbar preferences in MacOS control visibility of native scrollbars.
> 
> `"auto"` means that scrollbars are visible when content is overflowing on the corresponding orientation.
> `"always"` means that scrollbars are always visible regardless of whether the content is overflowing.
> `"scroll"` means that scrollbars are visible when the user is scrolling along its corresponding orientation.
> `"hover"` when the user is scrolling along its corresponding orientation and when the user is hovering over the scroll area.
> 
> MacOSのスクロールバー設定のように、スクロールバーの表示方法を制御するプロパティです。
> 
> `"auto"`: コンテンツがはみ出している場合にのみスクロールバーを表示します
> `"always"`: コンテンツのはみ出しに関係なく、常にスクロールバーを表示します
> `"scroll"`: ユーザーがスクロールしている時のみスクロールバーを表示します
> `"hover"`: ユーザーがスクロールしている時、またはスクロールエリアにホバーした時にスクロールバーを表示します

https://www.radix-ui.com/primitives/docs/components/scroll-area#root

これは単に`<ScrollArea type="auto" className="max-h-72" />`とすることで解決できました。

## 結び

冒頭でshadcn/uiを"痒いところに手が届かない"と表現しましたが、むしろ利用者がカスタマイズできる柔軟性こそがshadcn/uiの素晴らしいコンセプトだと感じました。そして、Radix Primitives と Tailwind CSS にも感謝したい2024年冬。
