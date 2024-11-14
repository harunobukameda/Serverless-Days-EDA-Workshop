# Serverless-Days-EDA-Workshop

このワークショップでは以下のサービスを活用したEDAアーキテクチャを構築を行います。それぞれのサービスをWebhookやAPIコールで接続し、全体で一つの大きなアーキテクチャを構成します。
それぞれのサービスの学習の第一歩としてください。

### 利用サービス
AWS: Amazon Event Bridge, Amazon Identity and Management (IAM), Amazon S3<br>
Momento: Momento Cache, Momento Topics<br>
Postman: Postman Collenctions, Postman Flows<br>
PingCap: TiDB Serverless<br>
Cloudflare Cloudflare Workers、Wrangler<br>

![画像1](https://github.com/user-attachments/assets/c7d9ca05-2a4d-4039-991f-d4fdb0a754c1)

### 事前準備と必要なもの
IDE: クラウド型は不可です。ローカルで動作する物を準備してください。VSCode推奨<br>
Node.js実行環境:18以降。[こちらから](https://nodejs.org/en)ダウンロードできます。<br>
npm: Nodeと一緒にインストールされるはずです。
ブラウザ：Webhookのテスト受信にも利用しますので、想定通り動作しない場合は、ZTNAやVPN等通信を制御する系のアプリは切って試してください。Chrome推奨<br>
アカウント：AWS以外はクレジットカード不要です。
<br>  [AWS](https://console.aws.amazon.com/)
<br>  [Momento](https://console.gomomento.com/)
<br>  [Postman](https://identity.getpostman.com/signup)
<br>  [TiDB Serverless](https://auth.tidbcloud.com/)
<br>  [Cloudflare](https://dash.cloudflare.com/login)

### 1. Momento Setup と Cacheの操作
[https://zenn.dev/momentobigfun/articles/aa24ff7817e06c](https://zenn.dev/momentobigfun/articles/aa24ff7817e06c)
Amazon Linuxと記事には記載されていますが、WindowsやMacでも動作します。

Momento AUTH_TOKENはMOMENTO_API_KEYに名前が変更となっているため、MOMENTO_API_KEYで統一して作業してください。

### 2. Momento Topics と Webhook 送出
[https://zenn.dev/momentobigfun/articles/9dbaf46e654ebe](https://zenn.dev/momentobigfun/articles/9dbaf46e654ebe)

### 3. Amazon S3 → Amazon EventBridge → Momento Cache
[https://zenn.dev/momentobigfun/articles/f3fe271f6c9ef9](https://zenn.dev/momentobigfun/articles/f3fe271f6c9ef9)

### 4. Amazon S3 → Amazon EventBridge → Momento Topics
先の手順ではMomento Cacheへデータを書き込んでいるためMomento Topicsへデータを書き込むよう変更します
APIの送信先を編集します。
![image](https://github.com/user-attachments/assets/888b14b4-558d-4ef6-98c6-37941e74aecf)
`送信先エンドポイント`を以下にします。
`https://api.cache.cell-ap-northeast-1-1.prod.a.momentohq.com/topics/*`
更新したらルールのターゲットを更新します。
![image](https://github.com/user-attachments/assets/2a70925c-63c8-439e-ac55-7fb4721409d1)
![image](https://github.com/user-attachments/assets/62a47ff4-7330-4e07-9ea0-543cdf2af2cf)
`serverless/test`は`{CacheName}`/`{TopicName}`ですので皆さんの環境に合わせて設定してください。

APIの送信先はPUTからPOSTに変更してください。
![image](https://github.com/user-attachments/assets/228022e7-b3b2-435f-9ab3-3de28c0d55d1)

Momento Cache でWebhookの飛ばし先を[https://webhook.site/](https://webhook.site/)で設定したパラメータにしておくと以下のようにS3バケットに保存されたファイル名を受信しながら
同時にWebhookを外部に出していることがわかります。
![image](https://github.com/user-attachments/assets/5b99a369-571a-410e-8dfb-82c9ddc176ae)
```json
{
   "cache":"serverless",
   "topic":"test",
   "event_timestamp":1726292279849,
   "publish_timestamp":1726292279850,
   "topic_sequence_number":3,
   "token_id":"",
   "text":"\"testfile2.txt\""
}
```

#### 5.Postman Collections と Postman Flows でAPIコールの作成
[https://qiita.com/KameMan/items/3e8d3a6e138fc47abcc4](https://qiita.com/KameMan/items/3e8d3a6e138fc47abcc4)　<br>
[https://qiita.com/KameMan/items/b3313738bf29ec469dd1](https://qiita.com/KameMan/items/b3313738bf29ec469dd1) <br>
まずはこの記事に従いPostman CollectonとFlowの関係性を学んでおきます。
簡単に説明するとCollectionはAPIコールを設定します。FlowsはCollectionsで設定されたAPIコールをもとに
一連のワークフローを作成できます。
次に以下の手順を行います。手順の中には新規でWebhookを作成するようになっていますが、手順4で作成したWebhookの向き先を`webhook.site`から変更して下さい。
[https://qiita.com/KameMan/items/7e072c8ac704baae821f](https://qiita.com/KameMan/items/7e072c8ac704baae821f) <br>
正しく設定できれば以下の通りメッセージ（S3バケットのファイル名)を正しく受信します。（余計なサニタイズされた文字は後で削除します）
![image](https://github.com/user-attachments/assets/f7a5fd7e-770b-477c-bfd9-c160880e4224)

#### 6. Cloudflare Workers でのHello World
[https://zenn.dev/kameoncloud/articles/1fac9762aab4ec](https://zenn.dev/kameoncloud/articles/1fac9762aab4ec)
をもとにHello Worldまでを実行します。

```
mkdir serverless
cd serverless
wrangler init serverless
```
TypeScriptではなくJavaScriptを使用します。
`index.js`をいかに置き換えて`Wrangler deploy`を実行します。
```javascript
export default {
	async fetch(request, env, ctx) {
		const url = new URL(request.url)
		const params = url.searchParams
		params.forEach((value, key) => {
			console.log(value);
		  })

		return new Response('Hello World!');
	},
};
```
次にPostman Collectionで先ほどRequest Catcherを向いていた物を以下に変更します。
![image](https://github.com/user-attachments/assets/d2a4519d-c8fc-439b-9123-bfdcd7ab6b4b)
`https://serverless.harunobukameda.workers.dev`は皆さん毎に異なるWorkers関数に割り当てられたドメイン名です。
`保存`を押しFlowsの画面で再度`Publish`をクリックします。

CloudflareのマネージメントコンソールからWorkersをクリックし、先ほど作成した関数の詳細を開きます。
![image](https://github.com/user-attachments/assets/4c8d46ec-4200-4871-8595-e7bbd63218da)
`ログ`のタブから`ログストリームの開始`をクリックしブラウザはそのままにしておきます。
![image](https://github.com/user-attachments/assets/c449b588-5408-4fbf-b0e6-acb7177eb658)
再度CloudshellからS3にファイルをアップロードします。
以下のようにファイル名を受け取っています。
```json
{
  "truncated": false,
  "outcome": "ok",
  "scriptVersion": {
    "id": "9b5bbf62-8650-41dc-a392-1044cf9fbcc0"
  },
  "scriptName": "serverless",
  "diagnosticsChannelEvents": [],
  "exceptions": [],
  "logs": [
    {
      "message": [
        "\"testfile3.txt\""
      ],
      "level": "log",
      "timestamp": 1726295836417
    }
```

#### 7. TiDB Serverlessの起動とテスト
[https://zenn.dev/kameping/articles/2248cb2833785e](https://zenn.dev/kameping/articles/2248cb2833785e) <br>
でまず、Serverlessクラスターの簡単な操作を学びます。
次にこちらの記事をもとにブラウザからSQLを実行できる`SQL Editor / Chat2Query`の操作を学んでおきます。
[https://zenn.dev/kameping/articles/9ebee487d30359](https://zenn.dev/kameping/articles/9ebee487d30359) <br>

#### 8. Cloudflare Workers と TiDB Serverless の連携テスト
[https://zenn.dev/kameoncloud/articles/99d3ed9d5ce4fd](https://zenn.dev/kameoncloud/articles/99d3ed9d5ce4fd) <br>
この手順を別ディレクトリで試します。7.の手順で作成したWorkers関数と混ざらないように別ディレクトリで作業を行ってください。
（先ほどと異なり今度はTypeScriptを使用します）
`index.ts`を以下に置き換え`wrangler deploy`を実行します。
```typescript
import { connect } from '@tidbcloud/serverless'


export interface Env {
   DATABASE_URL: string;
}

export default {
   async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
      const conn = connect({url:env.DATABASE_URL})
      
	  const url = new URL(request.url);
      const value1 = url.searchParams.get("value1");
	  const value_clear = value1.replace(/"/g, ""); // すべての " を削除

	  console.log(value_clear);

	  const resp = await conn.execute("INSERT INTO `bookshop`.`users` (`id`, `nickname`, `balance`) VALUES (1, '"+value_clear+"', 100.00);")

      return new Response(JSON.stringify(resp));
   },
};
```
次に先ほどと同じ手順でPostman CollectionsとFlowsを変更しPublishをクリックします。
CloudShellでファイルをアップロードします。
以下のようにTiDB Serverlessにデータが書き込まれていることがわかります。
![image](https://github.com/user-attachments/assets/8ab2b4d3-5b32-4845-bae9-baa1bfdb25ab)

## Optional Scenario : EDA の技術特性と対処法
今出来上がった環境は正常時には動作しますが、障害時のエラーハンドリングがなされていないことに注意してください。そしてEDAアーキテクチャには重要な３つの要素をあらかじめ設計しておく必要があります。<br>

1. 順不同メッセージ送出
2. 重複したメッセージ送出
3. メッセージの消失
4. コール先の障害

### 1. 順不同のメッセージ送出
Webhookの送出に用いたMomento Topicsは順番を保証していません。このためメッセージは順番を追い越す可能性があります。
Amazon SQSなどはFIFOキューという順番を保持する基盤がありますが、これもあくまで**送出**までです。受け取り側のネットワーク事象ではメッセージが順不同となる可能性を把握しておく必要があります。

### 2. 重複したメッセージ送出
メッセージに基盤の技術特性はベンダー間によって様々です。例えばAmazon SQSやCloudflare Queueは最低１回のメッセージ送出を保証しています。つまりタイミングで２回目があり得るということです。
これについての対象方はいくつかあります。

#### TiDB Serverless におけるUnique属性の付与
重複したデータの書き込みを受け付けたくない場合、当該カラムの属性をUniqueにしてしまうのが一番手っ取り早いです。
この場合Workers側のInertで重複処理はエラーになります。

```sql
ALTER TABLE `bookshop`.`users`
ADD CONSTRAINT constraint_name UNIQUE (`bookshop`.`nickname`);
describe `bookshop`.`users`;
```
一番最初のイベントのみが処理されます。

#### TiDB Serverless におけるUpsertの利用
上記とは逆に一番最後のイベントが最初の処理を上書きしても問題ない場合は`Insert`よりは`upsert`を使います。
`upsert`とはSQLのANSI標準ではないため、データベースエンジンごとに書式が異なりますが、データがあればUpdate、なければInsertを一度に行うコマンドです。
TiDB Serverlessの場合直接的なUpsertコマンドではなく以下のような構文になります。

```sql
INSERT INTO `bookshop`.`users` (`id`, `nickname`, `balance`)
VALUES (1, 'test', 100.00)
ON DUPLICATE KEY UPDATE `nickname` = 'test';
```

```javascript
import { connect } from '@tidbcloud/serverless'


export interface Env {
   DATABASE_URL: string;
}

export default {
   async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
      const conn = connect({ url: env.DATABASE_URL })

      const url = new URL(request.url);
      const value1 = url.searchParams.get("value1");
      const value_clear = value1.replace(/"/g, ""); // すべての " を削除

      const resp = await conn.execute("INSERT INTO `bookshop`.`users` (`id`, `nickname`, `balance`) VALUES (1, '" + value_clear + "', 100.00) ON DUPLICATE KEY UPDATE `nickname` = '" + value_clear + "',`balance`=100.00;")
      return new Response(JSON.stringify(resp));
   },
};
```
に変更することでUpsertを実装できます。

#### Cloudflare Workers ＋ KV のフラグ管理
WorkersにはNoSQL型KeyValue StoreであるKV、RDBであるD1というストレージが備わっています。
https://zenn.dev/kameoncloud/articles/7236a2c6ad35c0<br>
これらを用いることで既に処理済の文字列をKVに一時的に保存しておき重複処理を防ぐことが可能です。

以下のソースコードに変更して`wrangler deploy`を実行してください。
```javascript
import { connect } from '@tidbcloud/serverless'

export interface Env {
   DATABASE_URL: string;
}

export default {
   async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
      const conn = connect({ url: env.DATABASE_URL })

      const url = new URL(request.url);
      const value1 = url.searchParams.get("value1");
      const value_clear = value1.replace(/"/g, ""); // すべての " を削除

      const key = await env.serverless.get(value1)
      if (key == null) {
         await env.serverless.put(value1, "test");
         const resp = await conn.execute("INSERT INTO `bookshop`.`users` (`id`, `nickname`, `balance`) VALUES (1, '" + value_clear + "', 100.00) ON DUPLICATE KEY UPDATE `nickname` = '" + value_clear + "',`balance`=100.00;")
      }
      const resp = await conn.execute("SELECT * from bookshop.users;")
      return new Response(JSON.stringify(resp));
   },
};
```
**wrangler.toml**
```
#:schema node_modules/wrangler/config-schema.json
name = "tidb-cloud-cloudflare"
main = "src/index.ts"
compatibility_date = "2024-09-09"
compatibility_flags = ["nodejs_compat"]

[[kv_namespaces]]
binding = "serverless"
id = "66f3774ac18a4121ab49b60ce80dfe8c"
```
`bindings`や`id`は皆さんの環境ごとに異なります。

TiDB Serverlessに対してMomentoをインラインキャッシュとして実装するサンプルはこちらになりますので興味があればやってみて下さい。
https://zenn.dev/kameoncloud/articles/a21e0dcb92b67d<br>

### 3. メッセージ消失
Webhookの送出に用いたMomento Topicsはメッセージの到達を保証していません。一方Amazon SQSやCloudflare Queueはメッセージの到達を保証しています。
これはどちらが良い悪いではなく技術特性の違いによるものです。Momento Topicsのサブスクライバはステートフルコネクションによりメッセージを受信しています。<br>
このためかなり速いスピードでメッセージを受信します。一方Amazon SQSやCloudflare Queueはサブスクライバが自分のタイミングでメッセージを読みに行きます。（読んだ時点で消されます）
これはメッセージ到達性はより担保できますがパフォーマンスは低下します。

#### Momento Topics のSequence番号
Momento Topics には Sequence 番号が付与されています。これによりメッセージ順の追い越しや消失を知ることができます。
```json
{
   "cache":"serverless",
   "topic":"test",
   "event_timestamp":1726292279849,
   "publish_timestamp":1726292279850,
   "topic_sequence_number":3,
   "token_id":"",
   "text":"\"testfile2.txt\""
}
```
Postmanで受け取ったjsonから`text`に追加で`topic_sequence_number`を取り扱うにはもう一つEvaluateブロックを作成します。
一つのEvaluateブロックは一つの変数のみ取り扱うためです。

![image](https://github.com/user-attachments/assets/d9b8bd70-60d7-4f96-a6de-9a16c9bd3f47)
このようにvalue1は`text`、value2は`topic_sequence_number`をJSONから抜き出しています。
コレクションに`value2`という変数を追加すれば`topic_sequence_number`をWorkersに投げてくれます。
![image](https://github.com/user-attachments/assets/df9ec677-5173-4ad7-b42e-a5adb9a18c27)

Workersのコードを以下に修正すれば、CloudflareマネージメントコンソールのログからWorkersが`topic_sequence_number`を受け取っていることが確認できます。
```javascript
import { connect } from '@tidbcloud/serverless'


export interface Env {
   DATABASE_URL: string;
}

export default {
   async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
      const conn = connect({ url: env.DATABASE_URL })

      const url = new URL(request.url);
      const value1 = url.searchParams.get("value1");
      const value2 = url.searchParams.get("value2");
      const value_clear = value1.replace(/"/g, ""); // すべての " を削除

      console.log(value2);
      const resp = await conn.execute("INSERT INTO `bookshop`.`users` (`id`, `nickname`, `balance`) VALUES (1, '" + value_clear + "', 100.00) ON DUPLICATE KEY UPDATE `nickname` = '" + value_clear + "',`balance`=100.00;")
      return new Response(JSON.stringify(resp));
   },
};
```

ただし注意点があります。Momento Topics は高速なリアルタイムデータストリーミングに特化した基盤です。このためデータの再送が行えません。データの再送処理が必須な場合はメッセージ基盤にAmazon EventBridge+DQLを使うなどの設計が必要です。その分Momento Topicsは高速に大量のデータ配信を得意としています。

### 4. コール先の障害
Amazon EventBridgeがMomento Topicsにメッセージを承知する場合、Momento Topicsに障害が発生しているとそのリクエストは設定に従いリトライされたのち失敗します。
そのイベントが消失しないように、EventBridgeではDead Letter Queue が準備されています。

#### Amazon EventBridge によるDLQの設定
EventBridgeのルール編集画面のターゲット設定画面の下部にて以下を設定します。
![image](https://github.com/user-attachments/assets/5792d5e5-f568-4e1e-a3cc-2eb610a47053)
以下に詳細がありますので参考にしてください。
https://docs.aws.amazon.com/ja_jp/eventbridge/latest/userguide/eb-rule-dlq.html

#### TiDB の CDCモード
TiDB Serverlessでは残念ながらこの機能は使えませんが、DedicatedクラスターではChange Data Caputre (CDC)モードによるChangefeedを設定可能です。これによりInsert,Update,Deleteがイベントとして送出され
データ同期が可能です。
https://zenn.dev/kameping/articles/8ac97a06ce3de8




























