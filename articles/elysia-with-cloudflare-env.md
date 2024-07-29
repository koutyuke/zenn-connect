---
title: "Cloudflare WorkersでElysiaJSを使うときのEnvの渡し方"
emoji: "🦊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [
  "Cloudflare", "ElysiaJS", "Wrangler"
]
published: true
---

初めまして、NITICのこうちゅけです。
最近`ElysiaJS`というバックエンドを構築するためのWebフレームワークを用いて個人開発を行っています。

https://elysiajs.com/

この`ElysiaJS`で作成したAPIサーバーをCloudflare Workersで公開するために`Wrangler`で開発を行っていました。
その際にElysiaJSで環境変数を扱うのが少し面倒だったので共有します。(誰かの助けになるといいな...)

# 0. 環境

この記事では以下の環境で開発、コード提案を行なっています。

```sh
ProductName:		macOS
ProductVersion:		14.5
BuildVersion:		23F79
```

```
* Bun: 1.1.12
* ElysiaJS: 1.1
* Wrangler: 3.63.1
```

# 1. ことの経緯

ElysiaJSはWinterCGに準拠しているためEdge環境下で使用することができると公式から明記されています。

https://elysiajs.com/patterns/mount.html#mount


> WinterCG is a standard for web-interoperable runtimes. Supported by Cloudflare, Deno, Vercel Edge Runtime, Netlify Function, and various others, it allows web servers to run interoperably across runtimes that use Web Standard definitions like Fetch, Request, and Response.

そのためCloudflare Workers環境はもちろんのこと、Next.jsのAPI routerにもmountすることが可能です。
ここで過去の自分は「お、ならWorkersで使ったろ」と思い至り、いざ開発を進めていくとこの今回のenvどうやって渡すか問題にぶち当たった次第です。

あくまでこの公式のページは「WinterCGに準拠している」と言っているだけであって「Cloudflare Workersでの開発をサポートしている」とは一言も言ってません。
そのためドキュメントどころか公式のDiscord鯖にもこう言った会話は見られませんでした。(あったら教えてください)

しかし、`Hono`はWorkersに対応しているためフレームワーク単位でサポートしています。
これと同じような感触でElysiaJSのclassを拡張してあげればいいやと思い開発したのがことの発端になります。

# 2. 結論

チャチャっと結論を申しますと以下のコードをコピペすれば即解決すると思います。(他のいい案あったらコメントください)

__コード__

```ts
import { Value } from "@sinclair/typebox/value";
import { Elysia, type ElysiaConfig, type Static } from "elysia";

// D1以外のenvのschema
const AppEnvWithoutCFModuleEnvSchema　= t.Object({
	APP_ENV: t.Union([t.Literal("development"), t.Literal("production"), t.Literal("test")]),
});

type AppEnvWithoutCFModuleEnv = Static<typeof AppEnvWithoutCFModuleEnvSchema>;

type CFModuleEnv = {
	DB: D1Database;
};

type AppEnv = AppEnvWithoutCFModuleEnv & CFModuleEnv;

type CustomSingleton = {
	decorator: {
		env: AppEnvWithoutCFModuleEnv;
		cfModuleEnv: CFModuleEnv;
	};
	store: {};
	derive: {};
	resolve: {};
};

class ElysiaWithEnv<const BasePath extends string = "", const Scoped extends boolean = false> extends Elysia<
	BasePath,
	Scoped,
	CustomSingleton
> {
	constructor(config?: ElysiaConfig<BasePath, Scoped>) {
		super(config);
	}

	public setEnv(env: AppEnv): this {
		const { DB, ...otherEnv } = env;

		const preparedEnv = Value.Clean(
			AppEnvWithoutCFModuleEnvSchema,
			Value.Convert(AppEnvWithoutCFModuleEnvSchema, otherEnv),
		);

		if (!Value.Check(AppEnvWithoutCFModuleEnvSchema, preparedEnv)) {
			console.error("🚨 Invalid environment variables");
			throw new Error("🚨 Invalid environment variables")
		}

		this.decorate("env", preparedEnv satisfies CustomSingleton["decorator"]["env"]).decorate("cfModuleEnv", {
			DB,
		} satisfies CustomSingleton["decorator"]["cfModuleEnv"]);

		return this;
	}
}

export { ElysiaWithEnv };
```

__使い方__

```ts:index.ts
const root = new ElysiaWithEnv({
	aot: false,
});

root
  .get("/", async ({ env, cfModuleEnv }) => { // Type Safe!
		return "Hello World!";
	});

export default {
	fetch: async (request: Request, env: AppEnv) => {
		return root.setEnv(env).fetch(request);
	},
} satisfies ExportedHandler<AppEnv>;

```

今回は`D1`を使用しているためこのようなコードになっています。
envの命名等はその都度変えてください。

# 3. 解説

ここで開発譚を述べたところで時間と労力の無駄なのでclassの解説をざっくり書いていきます。

まずこの拡張したclassに欲しい機能は以下通りです。

* envの設定
* envが設定されているかの検証
* pluginへの型情報の共有
* elysia classを直接置換しても問題ない

## envの設定

workersではenvはrequestが発生した時にfetch関数の引数として渡されます。このenvには直接型を指定することができるのでそれを利用しつつ、elysiaのインスタンスに渡す必要があります。正直これが一番実装に時間がかかりました。

このenvはelysiaのcontextの一部として提供したいため`Decorate`メソッドを使用します。

https://elysiajs.com/essential/context.html#decorate

またenvを設定するメソッド`setEnv`を作成し引数としてenvを受け取り、処理として`Decorate`メソッドを実行しています。

```ts
public setEnv(env: AppEnv): this {...}
```

## envが設定されているかの検証

検証のライブラリとしてElysiaJSの内部で使用されている`TypeBox`を使用しました。

https://github.com/sinclairzx81/typebox

`Check`を利用することによって渡されたenvとschemaが完全に一致しているかを確かめます。

```ts
Value.Check(AppEnvWithoutCFModuleEnvSchema, preparedEnv)
```

また、`Clean`と`Convert`を組み合わせて実行することによってschema通りのdateになります。


```ts
const preparedEnv = Value.Clean(
	AppEnvWithoutCFModuleEnvSchema,
	Value.Convert(AppEnvWithoutCFModuleEnvSchema, otherEnv),
);
```

## pluginへの型情報の共有

ElysiaJSは型引数として`BasePath`と`Scoped`があります。そのためまず、ElysiaWithEnv classではこれの二つを型引数と指定しています。

```ts
class ElysiaWithEnv<const BasePath extends string = "", const Scoped extends boolean = false> {...}
```

そしてElysia classを継承しつつ型情報を渡します。
Elysia classは第三型引数としてcontextの型情報を取ります

https://github.com/elysiajs/elysia/blob/main/src/index.ts#L147-L155

これのdecoratorにenvの型を渡すことによりpluginなどでもcontext内でenvの型補完が効くようになります。

```ts
class ElysiaWithEnv<...> extends Elysia<
	BasePath,
	Scoped,
	CustomSingleton
> {...}
```

## elysia classを直接置換しても問題ない

正直言ってこれはelysia classを拡張しているので特筆することはありません。
強いて言えばsetEnvの返り値が`this`になっていることぐらいでしょうか

```ts
public setEnv(env: AppEnv): this {
  // ...

  return this;
}
```

# 4. 終わりに

今回はElysiaJS + Workersでenvを渡すために、elysia classを拡張したというものです。
これで苦しんでいる誰かの助けになれば幸いです。