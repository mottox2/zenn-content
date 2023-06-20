---
title: "ReactのUIコンポーネントなら@shadcn/uiがちょうどいい"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "tailwind", "javascript", "shadcn", "radix"]
published: true
---

Reactでは様々なスタイリングの手法があり、その上に構築されるコンポーネント実装も多くの選択肢があります。
styled-componentsやemotionといった CSS in JSライブラリによるスタイリングや、それらのライブラリ上に構築されたMUIやChakra UIといったコンポーネントライブラリ。一方でTailwind CSSやCSS ModulesなどCSS寄りなスタイリング手法があります。
私自身としてはTailwindを利用し、コンポーネントに関しては自作することが多いです。Tailwind CSSをベースにしたコンポーネントライブラリはいくつかありますが、自分の肌に合うものはありませんでした。

しかし、最近ちょうどいい選択肢と思われる`@shadcn/ui`というものを見つけました。
一言でいうと、カスタマイズ性が高く、使いやすいUIコンポーネント集です。
本記事では、`@shadcn/ui`がどのようなものなのか、どういうメリットがあるのかを見ていきます。

![](https://ui.shadcn.com/og.jpg)

ただしTailwind CSSが好きじゃない方にはNot for youな可能性が高いです。

## `@shadcn/ui`とは？

https://ui.shadcn.com/

ドキュメントページには以下のようなことが書かれています。

>コピーペーストでアプリに組み込める美しく設計されたコンポーネント集
>依存関係としてインストールすることなく、npmを通して提供されないことから、コンポーネントライブラリではない。
>必要なコンポーネントを選んで、プロジェクトにコピペ、必要なカスタマイズを行います。コードはあなたのものです。

どういうことでしょうか？
とりあえず、実際に利用するフローを見てみましょう。（動作は2023/6/19時点のものです）

ライブラリを使うには[数ステップの設定](https://ui.shadcn.com/docs/installation)が必要です。試すだけであれば[Next.jsテンプレートを使う](https://ui.shadcn.com/docs/installation#new-project)のがおすすめです。
ドキュメントにあるコンポーネントの中から利用したいコンポーネントを選びます。例えば、今回はButtonを使ってみます。
[ドキュメントページ](https://ui.shadcn.com/docs/components/button)に表示されているコマンドを入力するとcomponents/uiディレクトリにbutton.tsxが追加されます。

```shell
$ npx @shadcn/ui add button
```

```tsx:components/ui/button.tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"

import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none ring-offset-background",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "underline-offset-4 hover:underline text-primary",
      },
      size: {
        default: "h-10 py-2 px-4",
        sm: "h-9 px-3 rounded-md",
        lg: "h-11 px-8 rounded-md",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

このファイルは次のような方法で利用します。利用方法はドキュメントを見るといくつか例が載っています。

```tsx:page.tsx
import { Button } from "@/components/ui/button"

function Page() {
  return <div>
    <Button variant="outline">Button</Button>

    {/* `@radix-ui/react-slot`によってLinkにButtonの見た目を渡せる。  */}
    <Button asChild>
      <Link href="/login">Login</Link>
    </Button>
  </div>
}
```

簡単ですね。もちろん、button.tsxをいじれば、サイズ感やフォントを変えるのも簡単です。また、他のコンポーネントを使いたい時は、渡すコマンドを変えるだけでそのコンポーネントがプロジェクト内に展開されます。

ここまで見てピンと来ない方もいると思います。単なるコピペをCLI経由で行っているだけにも見えるかもしれません。
この方法にどういうメリット（またデメリット）があるのか説明します。

## `@shadcn/ui`のもたらすメリット

単体で見るのではなく、MUIやChakra UIと比較するとメリットが見えてきます。

### カスタマイズ、アップデートに追従しやすい

いわゆるコンポーネントライブラリをReactで使用する場合、通常はnpm経由でライブラリをインストールし、それをimportして使用します。この方法の利点は、利用者がライブラリの内部構造について意識する必要がないという点です。しかし一方で、ライブラリの作者が想定していないカスタマイズを行いたい場合には制約があります。
`@shadcn/ui`ではTailwindのコードを直接書き換えたり、CSS Variablesで提供されている変数を書き換えるだけでカスタマイズできるので、`@shadcn/ui`のために覚えることは最小限で済みます。

また、ライブラリの更新内容を把握しなくていいのも良い点です。
運用しているアプリケーションの依存関係を新しくしておきたい気持ちはすごくありますが、UIコンポーネントのアップデートは躊躇しがちです。
Visual Regression Testで変更前後の見た目を検証することもできますが、テスト構築のコストもそれなりにかかります。
一方、`@shadcn/ui`は依存関係に追加されません。なのでコンポーネントのためのパッケージ管理はコンポーネントの依存するTailwind CSSとRadixのみを意識すれば大丈夫です。
更新を取り込みたい時は再度`@shadcn/ui add [component]`を行えば最新のコードを取得できます。この変更は`git diff`で確認できるので安心して取り込めます。

ドキュメントページの「コードはあなたのものです」といった言葉はこのあたりから来るものでしょう。

### 自作コンポーネントの追加がしやすい
ライブラリから提供されていないコンポーネントを追加しやすいのもメリットです。
MUIやChakra UIではsx propsといってデザイントークンを考慮してスタイリングをするインターフェースがあるのですが、自作コンポーネントでも同じような動きを期待してしまいます。ちゃんとやろうとするとコンポーネントライブラリの知識を求められます。

一方、`@shadcn/ui`にはPaginationがありませんが、`@shadcn/ui`の作法に合わせてコンポーネントを作るのは簡単です。
他のコンポーネントのコードが露出しているので、同じインタフェースのコンポーネントを実装すれば、開発者からは`@shadcn/ui`から展開したコンポーネントと同じ使用感で利用できます。

### ヘッドレスUIの実装が組み込まれている
MUIやChakra UIを使っていると、挙動も含めて実装されたコンポーネントが使えますが、Tailwind CSSでは自分で実装しなければなりません。
Tailwind CSSでスタイリングされたコンポーネントに挙動を追加する際、Headless UIやRadixといった挙動だけを提供するヘッドレスUIライブラリがよく使われます。
`@shadcn/ui`で展開されるコンポーネントはRadixで実装されたものになっており、挙動は実装されつつも、Tailwind CSSで見た目をいじれる状態になっています。

もっと知りたい方は、microCMSのブログでヘッドレスUIを紹介している記事がおすすめです。
https://blog.microcms.io/radix-ui-headless-ui/

### CLIによるコンポーネントの追加が簡単

コピペでプロジェクトに追加できるHTML集のようなものは過去にも多数ありました。
Tailwind公式からもTailwind UIというテンプレート集が提供されていますが、手動でコピペしてプロジェクトに追加する必要があり、必要な依存関係がある場合、手動で`npm install`を行わないといけません。
カスタマイズ性が担保されてはいますが、動くまでが面倒になりがちです。`@shadcn/ui`はコマンド1つでコンポーネントファイルの設置（しかもファイル名が一意に決まる）と依存関係の追加を行ってくれるので、開発者が考えることはほとんどありません。

### 離脱が容易で捨てやすい
ウェブフロントエンドのライブラリで悲しくもよくあるのが、ライブラリのメンテが止まったり、メジャーバージョンアップによる破壊的変更についていけないという事態です。
そういう事態があるため、特に影響の大きいUIコンポーネントライブラリは慎重に選ぶのが大切です。
しかし、`@shadcn/ui`はTailwindとRadixで実装されたコンポーネントをプロジェクトに追加するだけで、`@shadcn/ui`自体についていく必要はありません。
Tailwindの作法でカスタマイズする必要がありますが、TailwindやRadixに依存する決心さえついていれば、（比較的）気軽に導入できると感じています。

## まとめ

`@shadcn/ui`はTailwindに対して抵抗感がない方にとってはかなり面白いものだと思います。今までUIコンポーネントライブラリに苦しめられたことが多かったので救世主になる可能性を感じています。逆に深刻な負債になる可能性も秘めていますが、それはしばらく運用してわかっていくでしょう。
気に食わない挙動があったらTailwindの作法で変更できる、そんな`@shadcn/ui`に期待をしています。

それにしても`shadcn`ってどう発音するんでしょうか…？

### 余談: CSS in JSの動向にも要注目？
React Server Componentの登場で一気にCSS in JS離脱の動きが強くなりTailwindやCSS Modulesに人口が戻ってきた気がします。
その一方でChakra UIの[Panda CSS](https://panda-css.com/)や国産の[Kuma UI](https://www.kuma-ui.com/)などランタイムではなくビルド時になんとかするようなアプローチが使われ始めています。これらはCSS in JSの開発体験をもちつつもRSCでも動くので、Tailwindが合わない人はこういった動きを追っておくといいかもしれません。
