---
title: "shadcn/ui(Radix Primitives)のScrollAreaでmax-heightと可変の高さを設定する"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "shadcnui", "radix", "radixui"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー2024](https://zenn.dev/galapagos/articles/bb5713c3a5d147)の15日目の記事です。

日頃からお世話になっているshadcn/uiですが、時々痒いところに手が届かないことがありカスタマイズして利用しています。
今回は、[Scroll Area](https://ui.shadcn.com/docs/components/scroll-area)について、`max-height`を指定したい場面があったのでその実装方法を紹介します。
shadcn/uiのScroll-areaは[Radix PrimitivesのScroll Area](https://www.radix-ui.com/primitives/docs/components/scroll-area)から作られています。

https://ui.shadcn.com/docs/components/scroll-area
https://www.radix-ui.com/primitives/docs/components/scroll-area

# dddd

![完成UI](https://storage.googleapis.com/zenn-user-upload/1864d3ec4968-20241213.gif)
