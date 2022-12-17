---
title: "DenoとPlanetScaleを繋げて簡単なAPIを作る"
emoji: "🪐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [deno, planetscale]
published: true
---

この記事は、Deno Advent Calendar 2022の17日目です。

https://qiita.com/advent-calendar/2022/deno/

## はじめに
[PlanetScale](https://planetscale.com/)とは、サーバーレスでMySQLを利用できるサービスです。詳しい説明や始め方は以下記事を参照していただければわかりますが、アカウント作成することで簡単にMySQLのデータベースを作成できるサービスというイメージで大丈夫だと思います。
https://planetscale.com/docs/tutorials/planetscale-quick-start-guide

今回はPlanetScaleのデータベースを作成したあと、どのようにDenoのAPIに繋げるかを書いていきます。

### 環境
```
$ uname -sr
Linux 5.15.79.1-microsoft-standard-WSL2

$ deno --version
deno 1.29.1 (release, x86_64-unknown-linux-gnu)
v8 10.9.194.5
typescript 4.9.4
```

## DenoでAPIを作成
まずAPIを作成します。単純なものなので[oak](https://github.com/oakserver/oak)などフレームワークは使用せず、標準ライブラリのみを使用します。

```typescript:main.ts
import { type Handler, serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { contentType } from "https://deno.land/std@0.168.0/media_types/mod.ts";

export const sampleApi = () => {
  // TODO: ここにPlanetScaleとの接続処理に置き換える
  const result = {
    "response": "test",
  };

  return new Response(JSON.stringify(result), {
    headers: {
      "Content-type": contentType("json"),
    },
  });
};

const handler: Handler = (req) => {
  const { pathname } = new URL(req.url);

  // API
  if (pathname.startsWith("/api")) return sampleApi();

  // 404画面
  return new Response("Not Found", { status: 404 });
};

const PORT = 9000;
serve(handler, { port: PORT });

```

上記のコードを実行してAPIの動作が確認できたところで、取り出すためのデータをPlanetScaleに保存していきます。

## PlanetScaleにデータを保存する
PlanetScale公式のクイックスタートガイドでは、[PlanetScale CLI](https://github.com/planetscale/cli)を利用したデータ保存が案内されています。しかし、TEXTデータ型など大量の文字列を表示すると非常に見づらいので、ここではPlanetScale公式Webアプリのコンソール画面でデータを保存していきます。

![PlanetScaleのWebアプリのコンソール画面](/images/deno-adapt-planetscale/planetscale-console.png)

このページには、作成したデータベースからテーブルページにアクセスすることで見ることができます。

![PlanetScaleのWebアプリのコンソール画面](/images/deno-adapt-planetscale/planetscale-database.png)

コンソール画面にSQLを入力することでデータ保存や修正、削除が可能です。  
まず、適当なテーブルを作成してください。

```sql
CREATE TABLE IF NOT EXISTS tbl_test (
  id INT UNSIGNED NOT NULL UNIQUE AUTO_INCREMENT COMMENT 'ID',
  title VARCHAR(255) COMMENT 'タイトル',
  body TEXT NOT NULL COMMENT '本文',

  PRIMARY KEY (id)
);
```

次に、適当なデータを追加してください。

```sql
INSERT INTO
  tbl_test(
    title,
    body
  )
VALUES
  (
    'タイトル1',
    '比較的自由な空気を呼吸している事があるのだと解釈して掛ります。私はぼんやりお嬢さんの所作はその点についてからは、すぐ封を切る訳に行かないのでしょうが、ぜひお嬢さんを固く信じて疑わなかったのです。お前のよく先生先生という言葉は、おそらく彼にもよく解っています。結婚の申し込みを拒絶されたものと見えていました。それご覧なと母が答えた。'
  ),
  (
    'タイトル2',
    '奥さんはそれじゃ私の知っているといって私を抑え付けるのです。しかしＫに説明を与えるために、今帰ったかといって、別に平生と変った点はありませんが、その晩その包みの中を調べて見た上でないという昔風の言葉を父の記念のようにぐるぐる巻きつけてあった先生の語気が不審であった。私は当然自分の心を曇らす不審の種とならないとなると足元を見てまた笑い出しました。そうして、何でもなくなるものだと極めていた。世の中が嫌いになるんだからといいました。'
  );
```

これで、APIで取得するデータを用意できました。  
それでは、実際にPlanetScaleとAPIを繋げていきましょう。


## PlanetScaleと繋ぐ
今回は、Deno v1.28で安定化したnpm互換性を活用して、PlanetScaleのDBドライバである[`@planetscale/database`](https://github.com/planetscale/database-js)を利用します。  
`npm:@planetscale/database@1.4.0`だけを指定すると型定義がつかないので困惑すると思いますが、これは既にモジュール側でも[Issue](https://github.com/planetscale/database-js/issues/76)が立っており回避策も提案されています。今回のコードにはあらかじめ回避策を盛り込んでいるので、そのままエディタに貼り付けても問題ないと思います。

最初のコードから変更を加えるのは`sampleApi`の部分なので、そこまでを記載します。

```typescript:main.ts
import { type Handler, serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { contentType } from "https://deno.land/std@0.168.0/media_types/mod.ts";
// @deno-types="https://esm.sh/@planetscale/database@1.5.0/dist/index.d.ts"
import { connect } from "npm:@planetscale/database@1.4.0";
import "https://deno.land/std@0.168.0/dotenv/load.ts";

export const sampleApi = async (req: Request) => {
  const conn = connect({
    host: Deno.env.get("PS_HOST"),
    username: Deno.env.get("PS_USERNAME"),
    password: Deno.env.get("PS_PASSWORD"),
  });
  const result = await conn.execute("SELECT * FROM tbl_test");

  return new Response(JSON.stringify(result.rows), {
    headers: {
      "Content-type": contentType("json"),
    },
  });
};

// 下にハンドラ関数が書かれている
```
### `@planetscale/database`を型定義ありで読み込む
先ほど書いた回避策は、
```typescript
// @deno-types="https://esm.sh/@planetscale/database@1.5.0/dist/index.d.ts"
```
の部分です。  
Denoのnpm互換性では、DefinitelyTypedなどnpmパッケージ外の型定義ファイルの参照も対応しているため、型定義ファイルのズレにも同じように対応が可能です。

### PlanetScaleの設定の渡し方と取得方法
次に、connect関数に渡す設定についてです。

```typescript
const conn = connect({
  host: Deno.env.get("PS_HOST"),
  username: Deno.env.get("PS_USERNAME"),
  password: Deno.env.get("PS_PASSWORD"),
});
```

ホストやユーザーネームなどの設定情報を、環境変数で読み込んでいます。  
このままだと`.env`に指定した環境変数は読み込めないのですが、以下のように標準ライブラリのdotenvモジュールの自動読み込みを使えば、利用が可能です。

```typescript
import "https://deno.land/std@0.168.0/dotenv/load.ts";
```

そしてその設定情報はデータベースページ右上のConnectボタンを押下し、Connect withのプルダウンから`@planetscale/database`を選択することで入手できます。

![PlanetScaleのWebアプリのConnectボタン](/images/deno-adapt-planetscale/planetscale-connect.png)
![PlanetScaleのWebアプリでライブラリと接続](/images/deno-adapt-planetscale/planetscale-connect-with.png)

### PlanetScaleからのデータ取得
設定でデータベースの位置を指定したので、あとはデータを取得するテーブルとフィールドの指定だけです。
`@planetscale/database`は、SQLの文字列を`execute`メソッドに渡すことで、PlanetScale上のデータベースにアクセスします。

```typescript
const result = await conn.execute("SELECT * FROM tbl_test");
```

SQLをそのまま渡すので自由度は高いです。  
SQLを学びつつ物が作りたいという人には、最適なのではないでしょうか。

そして最後に、出力結果の`rows`プロパティに格納されたデータを`Response`に渡して、APIレスポンスとして出力します。

```typescript
return new Response(JSON.stringify(result.rows), {
  headers: {
    "Content-type": contentType("json"),
  },
});
```

## 完成！
これで、PlanetScaleに接続してデータを取得するAPIができました。
改めてコードの全容を記載します。

```typescript:main.ts
import { type Handler, serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { contentType } from "https://deno.land/std@0.168.0/media_types/mod.ts";
// @deno-types="https://esm.sh/@planetscale/database@1.5.0/dist/index.d.ts"
import { connect } from "npm:@planetscale/database@1.4.0";
import "https://deno.land/std@0.168.0/dotenv/load.ts";

export const sampleApi = async () => {
  const conn = connect({
    host: Deno.env.get("PS_HOST"),
    username: Deno.env.get("PS_USERNAME"),
    password: Deno.env.get("PS_PASSWORD"),
  });
  const result = await conn.execute("SELECT * FROM tbl_test");

  return new Response(JSON.stringify(result.rows), {
    headers: {
      "Content-type": contentType("json"),
    },
  });
};

const handler: Handler = (req) => {
  const { pathname } = new URL(req.url);

  // API
  if (pathname.startsWith("/api")) return sampleApi();

  // 404画面
  return new Response("Not Found", { status: 404 });
};

const PORT = 9000;
serve(handler, { port: PORT });
```

PlanetScaleへのアクセスは`execute`メソッドにSQLを渡すだけなので、更新も削除もかなりシンプルになるはずです。
もしPOSTでデータを渡して追加更新がしたいという場合は、SQLを学べばなんとかなります。

### 注意点
npm互換性は、Deno製アプリのデプロイ先として最初に候補に上がる[Deno Deploy](https://deno.com/deploy)にはまだ搭載されていません。  
Deno Deployで利用する場合はesm.shを利用して、`npm:@planetscale/database@1.4.0`を`https://esm.sh/@planetscale/database@1.4.0`に置き換えてください。

またesm.shを利用し、かつGitHub ActionsでDenoを利用している場合、lockファイルの不整合でGitHub Actionsが落ちるので、`.gitignore`でlockファイルを弾くことをオススメします。  
これはesm.shの仕様の影響なので使っていなければ問題ないのですが、上記のコードでは型定義ファイルの指定に利用しているので弾いておくと安心です。

## 最後に
PlanetScaleに接続できるコードはGist上にも上げています。
https://gist.github.com/windchime-yk/d22f95396f779de8bf86bf89ca2f3934
`.env`を作成して設定情報を追加し、その直下で以下のコマンドを試してみてください。

```
deno run --allow-net=0.0.0.0:9000,aws.connect.psdb.cloud --allow-env --unstable https://gist.githubusercontent.com/windchime-yk/d22f95396f779de8bf86bf89ca2f3934/raw/f32aee01891f9afff2a527f7a271ec441705ba86/adapt-planetscale.ts
```

それでは、読んでくださってありがとうございました！  
18日目は、kn1chtさんの『Denoでもtextlint』です！
