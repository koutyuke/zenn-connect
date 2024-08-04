---
title: "【tailwindcss】configから直接、型を取り出してみた"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "tailwindcss"]
published: true
---

初めまして、NITICのこうちゅけです。

皆さん、Tailwindcss使ってますか？
`.css`ファイルを作成せずにスタイルコーディングできるTailwindcssはとてもいいものですよね。

しかし、Typescriptでコーディングしていると、Tailwindcssのデザイントークンに関する型情報がなくてモヤモヤすることが時たまあります。

また、動的にclassを変更する関数を実装しようとした際に型情報がなく型をハードコードすることもあると思います。

今回は設定ファイルから直接、`color`の型や`padding`, `margin`などで使用されているScale(lgや4,8など)の型情報を取得します。

# 0. 環境

この記事では以下の環境で開発、コード提案を行なっています。

```sh
ProductName:		macOS
ProductVersion:		14.5
BuildVersion:		23F79
```

```sh
❯ bun -v   
1.1.12
```

```
* typescript: 5.5.4
* tailwindcss: 3.4.7
```

# 1. 結論

結論を先に出すと以下のようにすればできます。

```ts:tailwindcss.config.ts
import type { Config } from "tailwindcss";

const config = {
  content: [],
  theme: {
    extend: {
      colors: {
        primary: "#ff0000",
        secondary: "#00ff00",
        tertiary: "#0000ff",
      },
    },
  },
  plugins: [],
} satisfies Config;

export default config;
```

例として`primary`, `secondary`, `tertiary`を定義しています。

```ts:src/type.ts
import type resolveConfig from "tailwindcss/resolveConfig";
import type config from "../tailwind.config";

type ResolveConfig = ReturnType<typeof resolveConfig<typeof config>>;

type ResolveConfigTheme = ResolveConfig["theme"];

type Color = keyof ResolveConfigTheme["colors"];
// type Color = "primary" | "secondary" | "tertiary" | keyof DefaultColors
// --- 展開すると ---
// type Color = "primary" | "secondary" | "tertiary" | "inherit" | "current" |
//              "transparent" | "black" | "white" | "slate" | "gray" | "zinc" | 
//              "neutral" | "stone" | "red" | "orange" | ... 20 more

type FontSize = keyof ResolveConfigTheme["fontSize"];
// type FontSize = "xs" | "sm" | "base" | "lg" | "xl" | "2xl" | "3xl" | 
//                 "4xl" | "5xl" | "6xl" | "7xl" | "8xl" | "9xl"

type ZIndex = keyof ResolveConfigTheme["zIndex"];
// type ZIndex = "0" | "10" | "20" | "30" | "40" | "50" | "auto"
```

`tailwindcss.config.ts`で定義している`color`情報などもあせて取得することができます。

つまり、configが変わったから型を変える…というようなことをしなくて済むようになります。

# 2. 詳細

さて、結論だけで終わりにしても一体何してるん？ってなるだけなのでコードの詳細と注意事項を書いていきます。

## tailwind.config.ts

### ファイルの作成

tailwindcssのconfigファイルを作成します。

```sh
bunx tailwindcs init
```

コマンドで作成すると`.js`ファイルが生成されますが __`.ts`__ に直してください。

型を取得するのに`.js`だと取れるものも取れませんからね。

:::message
tailwindcssの設定ファイルは __.ts__ ファイルで作成してください
:::

### カスタマイズする

次に自分好みにconfigを書いていきます。
今回はcolorだけを定義しましたが、特に制約はないので好きに書いてください。

```ts:tailwind.config.ts
import type { Config } from "tailwindcss";

const config = {
  content: [],
  theme: {
    extend: {
      colors: {
        primary: "#ff0000",
        secondary: "#00ff00",
        tertiary: "#0000ff",
      },
    },
  },
  plugins: [],
} satisfies Config;

export default config;
```

ここで注意しないといけないのが型の付与の仕方です。
`Config`の型の変数を定義するのではなく、`Config`の型に沿ったオブジェクトを定義するようにします。
そのため、`satisfies`構文を使用して変数を定義します。

```ts
// ❌
const config: Config = {
  ...
}

// ⭕️
const config = {
  ...
} satisfies Config;
```

:::message
型制約をかける時には`satisfies`構文を使用する。
:::

Config型のObjectデータにしてしまうと、後述する設定の結合時にカスタマイズした型情報がのらず型抽出ができないためです。

## 型を抽出する

configができたら次はデフォルトのconfigとマージする必要があります。
tailwindcssは設定の結合する関数を`tailwindcss/resolveConfig`で用意しているのでこれを使用します。

```ts
import resolveConfig from "tailwindcss/resolveConfig";
import config from "../tailwind.config";

resolveConfig(config);
```

このような記述でデフォルト設定とカスタマイズした設定を結合します。

しかし、欲しいのはこれの返り値の型情報です。
なので型だけになるように修正します。
`tailwindcss/resolveConfig`で用意されている関数の型情報はこのように定義されています。

```ts:tailwindcss/resolveConfig.d.ts
import { Config, ResolvableTo, ThemeConfig } from './types/config'
import { DefaultTheme } from './types/generated/default-theme'
import { DefaultColors } from './types/generated/colors'

type ResolvedConfig<T extends Config> = Omit<T, 'theme'> & {
  theme: MergeThemes<
    UnwrapResolvables<Omit<T['theme'], 'extend'>>,
    T['theme'] extends { extend: infer TExtend } ? UnwrapResolvables<TExtend> : {}
  >
}

type UnwrapResolvables<T> = {
  [K in keyof T]: T[K] extends ResolvableTo<infer R> ? R : T[K]
}

type ThemeConfigResolved = UnwrapResolvables<ThemeConfig>
type DefaultThemeFull = DefaultTheme & { colors: DefaultColors }

type MergeThemes<Overrides extends object, Extensions extends object> = {
  [K in keyof ThemeConfigResolved | keyof Overrides]: (K extends keyof Overrides
    ? Overrides[K]
    : K extends keyof DefaultThemeFull
    ? DefaultThemeFull[K]
    : K extends keyof ThemeConfigResolved
    ? ThemeConfigResolved[K]
    : never) &
    (K extends keyof Extensions ? Extensions[K] : {})
}

declare function resolveConfig<T extends Config>(config: T): ResolvedConfig<T>
export = resolveConfig
```

まぁ、要約すると`resolveConfig`の型引数にマージしたいObjectの型を渡せばいいだけです。

```ts
import type resolveConfig from "tailwindcss/resolveConfig";
import type config from "../tailwind.config";

type ResolveConfig = ReturnType<typeof resolveConfig<typeof config>>;
```

これで結合後のconfigの型情報が取得できました。
例として、`color`の型を取得しようと思います。colorはtheme/colorsにあるのでその通りに記述していきます。

```ts
import type resolveConfig from "tailwindcss/resolveConfig";
import type config from "../tailwind.config";

type ResolveConfig = ReturnType<typeof resolveConfig<typeof config>>;

type ResolveConfigTheme = ResolveConfig["theme"];

type Color = keyof ResolveConfigTheme["colors"];
```

これで抽出ができました!!
あとは欲しい型情報を自分で抽出してみてください。

# 終わりに

今回はtailwindcssのconfigから型情報を抽出してみました。
思ったより上手く抽出できたのでよかったです。

もっと良い記述があれば教えてください。

では、また!