[![FIWARE Banner](https://fiware.github.io/tutorials.Subscriptions/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Subscriptions.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
<br/> [![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

<!-- prettier-ignore -->

このチュートリアルでは、FIWARE ユーザにコンテキスト・データのサブスクリプション
を作成および管理する方法について説明しています。チュートリアルでは、ユーザが
[NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) サブスクリプショ
ン/通知のパラダイムを理解でき、NGSI サブスクリプションを独自のコードで使用する方
法を説明するため、前
の[例](https://github.com/FIWARE/tutorials.Accessing-Context/)で作成したエンティ
ティ
と[在庫管理のフロントエンド・アプリケーション](https://github.com/FIWARE/tutorials.Step-by-Step/tree/master/context-provider)を
ベースにしています。

このチュートリアルでは、[cUrl](https://ec.haxx.se/) コマンドを組み合わせて、ブラ
ウザ内で行われた在庫管理アクションを示します。cUrl コマンドは
、[Postman マニュアル](https://fiware.github.io/tutorials.Accessing-Context/) と
しても利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/18ea15017244c70d1fe4)

## コンテンツ

<details>
<summary>詳細 <b>(クリックして拡大)</b></summary>

-   [状態の変更をサブスクリプションする](#subscribing-to-changes-of-state)
    -   [在庫管理システム内のエンティティ](#entities-within-a-stock-management-system)
    -   [在庫管理フロントエンド](#stock-management-front-end)
-   [アーキテクチャ](#architecture)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [起動](#start-up)
-   [サブスクリプションの使用](#using-subscriptions)
    -   [単純なサブスクリプションの設定](#setting-up-a-simple-subscription)
    -   [`attrs` と `attrsFormat` でペイロードを削減](#reducing-payload-with--attrs-and-attrsformat)
    -   [`expression` でスコープ を縮小](#reducing-scope-with--expression)
-   [サブスクリプションの CRUD アクション](#subscription-crud-actions)
    -   [サブスクリプションの作成](#creating-a-subscription)
    -   [サブスクリプションの削除](#delete-a-subscription)
    -   [既存のサブスクリプションの更新](#update-an-existing-subscription)
    -   [すべてのサブスクリプションの一覧表示](#list-all-subscriptions)
    -   [サブスクリプションの詳細を読み込む](#read-the-detail-of-a-subscription)
-   [次のステップ](#next-steps)

</details>

<A name="subscribing-to-changes-of-state"></A>

# 状態の変更をサブスクリプションする

> "電話はいらないよ、こちらからかけるから"
>
> — Dorothy Kilgallen (The Voice Of Broadway)

FIWARE プラットフォームでは、エンティティは、実世界に存在する物理的または概念的
オブジェクトの状態を表します。すべてのスマートなソリューションは、特定の瞬間にこ
れらのオブジェクトの現在の状態を知る必要があります。

これらのエンティティのコンテキストは常に変化しています。たとえば、在庫管理の例で
は、新しいストアが開店したり、商品が販売されたり、価格が変化するなどして、状況が
変化します。IoT センサデータにもとづくスマートなソリューションでは、システムが常
に現実世界の変化に対応するため、この問題はさらに重要です。

これまで、システムの状態を変更するために使用したすべての操作は**同期**していまし
た。変更はユーザまたはアプリケーションによって直接行われ、結果が通知されました
。Orion Context Broker は、**非同期**の通知メカニズムも提供しています。アプリケ
ーションは、コンテキスト情報の変更をサブスクライブして、何か起きたときに通知を受
け取ることができます。これは、アプリケーションが継続的にポーリングまたはクエリ・
リクエストを繰り返す必要がないことを意味します。

したがって、サブスクリプション・メカニズムを使用すると、システム内のコンポーネン
ト間で渡されるリクエスト量とデータ量の両方が削減されます。このネットワーク・トラ
フィックの減少は、全体的な応答性を改善します。

<A name="entities-within-a-stock-management-system"></A>

## 在庫管理システム内のエンティティ

エンティティ間の関係は、次のように定義されます :

![](https://fiware.github.io/tutorials.Subscriptions/img/entities.png)

青色で強調表示された項目は、外部コンテキスト・プロバイダによって提供されます。

<A name="stock-management-front-end"></A>

## 在庫管理フロントエンド

前の[チュートリアル](https://github.com/FIWARE/tutorials.Accessing-Context/)では
、単純な Node.js Express アプリケーションが作成しました。このチュートリアルでは
、モニタ・ページを使用して最近のリクエストのステータスを表示し、商品を購入するス
トア・ページを使用します。サービスが実行されると、次の URLs からこれらのページに
アクセスできます :

#### イベント・モニタ

イベント・モニタは次の場所にあります : `http://localhost:3000/app/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Subscriptions/img/monitor.png)

#### ストア 002

ストア 002 は次の場所にあります :
`http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![Store](https://fiware.github.io/tutorials.Subscriptions/img/store2.png)

<A name="architecture"></A>

# アーキテクチャ

このアプリケーションは
、[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) という
1 つの FIWARE コンポーネントのみを使用します。アプリケーションが _"Powered by
FIWARE"_ と認定するには、Orion Context Broker を使用するだけで十分です。

現在、Orion Context Broker はオープンソースの
[MongoDB](https://www.mongodb.com/) 技術を利用して、コンテキスト・データの永続性
を維持しています。外部ソースからコンテキスト・データをリクエストするために、単純
な**コンテキスト・プロバイダ NGSI proxy**も追加されています。コンテキストを視覚
化して操作するために、簡単な Express **フロントエンド**・アプリケーションを追加
します。

したがって、アーキテクチャは 4 つの要素で構成されます :

-   [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリ
    クエストを受信する Orion Context Broker サーバ
-   バックエンドの [MongoDB](https://www.mongodb.com/) データベース
    -   Orion Context Broker が、データ・エンティティなどのコンテキスト・データ
        情報、サブスクリプション、登録などを保持するために使用します
-   **コンテキスト・プロバイダ NGSI proxy**は次のようになります :
    -   [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用し
        てリクエストを受信します
    -   独自の API を独自のフォーマットで使用して、公開されているデータソースへ
        のリクエストを行います
    -   [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) 形式でコ
        ンテキスト・データを Orion Context Broker に返します
-   **在庫管理フロントエンド**は以下を行います :
    -   ストア情報を表示します
    -   各ストアで購入できる製品を表示します
    -   ユーザが製品を"購入"して、在庫数を減らすことを可能にします

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコン
テナ化され、公開されたポートから実行されます。

![](https://fiware.github.io/tutorials.Subscriptions/img/architecture.png)

必要な設定情報は、関連する `docker-compose.yml` ファイルの services セクションに
あります。
[以前のチュートリアル](https://github.com/FIWARE/tutorials.Context-Providers/)で
説明しました。

<A name="prerequisites"></A>

# 前提条件

<A name="prerequisites"></A>

## Docker

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com)
を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に
分離することを可能にするコンテナ・テクノロジです。

-   Docker を Windows にインストールするには
    、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってくださ
    い
-   Docker を Mac にインストールするには
    、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker を Linux にインストールするには
    、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行する
ためのツールです
。[YAML file](https://raw.githubusercontent.com/FIWARE/tutorials.Subscriptions/NGSI-v2/docker-compose.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つ
まり、すべてのコンテナ・サービスは 1 つのコマンドで呼び出すことができます
。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac の一部と
してインストールされますが、Linux ユーザ
は[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要
があります。

次のコマンドを使用して、現在の **Docker** バージョンと **Docker Compose** バージ
ョンを確認できます :

```console
docker-compose -v
docker version
```

Docker バージョン 20.10 以降と Docker Compose 1.29 以上を使用していることを確認
し、必要に応じてアップグレードしてください。

<A name="prerequisites"></A>

## Cygwin

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは
[cygwin](http://www.cygwin.com/) をダウンロードして、Windows 上の Linux ディスト
リビューションと同様のコマンドライン機能を提供する必要があります。

<A name="start-up"></A>

# 起動

リポジトリ内で提供される bash スクリプトを実行すると、コマンドラインからすべての
サービスを初期化できます。リポジトリを複製し、以下のコマンドを実行して必要なイメ
ージを作成してください :

```console
git clone https://github.com/FIWARE/tutorials.Subscriptions.git
cd tutorials.Subscriptions
git checkout NGSI-v2

./services create; ./services start;
```

このコマンドは、起動時に以前
の[在庫管理の例](https://github.com/FIWARE/tutorials.Context-Providers)からシー
ドデータをインポートします。

> :information_source: **注 :** クリーンアップをやり直したい場合は、次のコマンド
> を使用して再起動することができます :
>
> ```console
> ./services stop
> ```

<A name="start-up"></A>

# サブスクリプションの使用

チュートリアルを正しく実行するには、cUrl コマンドを入力する前に、ブラウザのタブ
で次のページを使用できることを確認してください。

#### イベント・モニタ

イベント・モニタは次の場所にあります : `http://localhost:3000/app/monitor`

#### ストア

ストアは次の場所にあります :

-   ストア 1 - `http://localhost:3000/app/store/urn:ngsi-ld:Store:001`
-   ストア 2 - `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

<A name="setting-up-a-simple-subscription"></A>

## 単純なサブスクリプションの設定

在庫管理の例で、会社の地域マネージャが製品の価格を変更したいと考えているとします
。新しい価格は、システム内のすべてのストアに反映されます。価格が頻繁に変更されな
いように、新しい情報を常にポーリングするようにシステムを設定することは可能です。
そうすれば、リソースの無駄となり、不必要なデータトラフィックが大量に発生します。

別の方法は、価格が変更されたときにペイロードを "よく知られた" URL に POST するサ
ブスクリプションを作成することです。以下に示すように、`/v2/subscriptions/` エン
ドポイントに POST リクエストを行うことで、新しいサブスクリプションを追加できます
:

#### :one: リクエスト :

```console
curl -iX POST \
  --url 'http://localhost:1026/v2/subscriptions' \
  --header 'content-type: application/json' \
  --data '{
  "description": "Notify me of all product price changes",
  "subject": {
    "entities": [{"idPattern": ".*", "type": "Product"}],
    "condition": {
      "attrs": [ "price" ]
    }
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/subscription/price-change"
    }
  }
}'
```

POST リクエストのボディは、2 つの部分で構成されています。リクエストの `subject`
セクション (`entities` および `conditions` からなる) は、**Product** エンティテ
ィの `price` 属性が変更されたときにサブスクリプションが起動されることを示します
。ボディの `notification` セクションでは、サブスクリプションの条件が満たされると
、影響を受けるすべての **Prodcut** エンティティを含む POST リクエストが、在庫管
理フロントエンド・アプリケーションによって処理される URL
`http://tutorial:3000/subscription/price-change` に送信されることが示されます。

最初の実行では、サブスクリプションが作成されると、Orion Context Broker は
`condition` テストを実行し、以前に実行されていないため、すべての製品が変更された
と仮定します。したがって、次のようにすぐに `subscription/price-change` にリクエ
ストが送信されます。

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/products-subscription.png)

在庫管理フロントエンド・アプリケーション内のコードは、次のように POST リクエスト
を受け取りました :

```javascript
router.post("/subscription/:type", (req, res) => {
    _.forEach(req.body.data, (item) => {
        broadcastEvents(req, item, ["refStore", "refProduct", "refShelf", "type"]);
    });
    res.status(204).send();
});

function broadcastEvents(req, item, types) {
    const message = req.params.type + " received";
    _.forEach(types, (type) => {
        if (item[type]) {
            req.app.get("io").emit(item[type], message);
        }
    });
}
```

このビジネス・ロジックは、登録されたパーティ(現金など)にソケット io イベントを発
行します。

現金は、イベントを受け取った場合にリロードするように設定されていますが、この場合
での、価格はまだ変更されていないため、製品価格は変わりません。例えば、ビールのボ
トルは 0.99 ユーロのままです。

#### `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![](https://fiware.github.io/tutorials.Subscriptions/img/beer-99.png)

ビールのボトルの価格を 0.89 ユーロに下げましょう。これはプログラムではまだできま
せんので、次のように curl コマンドを使用して実行する必要があります :

#### :two: リクエスト :

```console
curl -iX PUT \
  --url 'http://localhost:1026/v2/entities/urn:ngsi-ld:Product:001/attrs/price/value' \
  --header 'Content-Type: text/plain' \
  --data 89
```

**Product** エンティティの属性が更新されるたびに、Orion Context Broker は、その
エンティティに存在する既存のサブスクリプションをチェックし、`condition` テストを
適用します。今回は最後の実行から 1 つの **Product** エンティティのみが変更された
ため、POST リクエストが `subscription/price-change` に送信されます。このリクエス
トはボディ内の 1 つの **Product** のみを含みます。

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/price-change.png)

**Product** エンティティの属性が更新されるたびに、Orion Context Broker は、その
エンティティに存在する既存のサブスクリプションをチェックし、`condition` テストを
適用します。今回は最後の実行から 1 つの **Product** エンティティのみが変更された
ため、POST リクエストが `subscription/price-change` に送信されます。このリクエス
トはボディ内の 1 つの **Product** のみを含みます。

#### `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![](https://fiware.github.io/tutorials.Subscriptions/img/beer-89.png)

<A name="reducing-payload-with--attrs-and-attrsformat"></A>

## `attrs` と `attrsFormat` でペイロードを削減

前の例では、影響を受けた各 **Product** エンティティの完全な詳細なデータが POST
通知と共に送信されました。これはあまり効率的ではありません。

`attrs` 通知メッセージに含める属性のリストを指定する属性を追加することで、渡すデ
ータの量を減らすことができます。他の属性は無視されます。

> **ヒント** : `exceptAttrs` 属性も除外リスト上のものを除くすべての属性を返すた
> めに存在しています。同じサブスクリプションで同時に `attrs` と `exceptAttrs` を
> 使用することはできません。

`attrsFormat` 属性は、エンティティが通知でどのように表されるかを指定します。デフ
ォルトでは冗長なレスポンスが返されます。`keyValues` と `values` は
、`v2/entities` の GET リクエストと同じ方法で動作します。

<A name="reducing-scope-with--expression"></A>

## `expression` でスコープ を縮小

特定の条件下でのみ起動する 2 つのサブスクリプションを作成し、影響を受けるエンテ
ィティのキ ​​ ーと値のペアのみを返します。各ストアの倉庫が、棚の製品の量が閾値レ
ベルを下回るたびに通知を受けたいと考えているとします。

サブスクリプションは、**InventoryItem** の `shelfCount` が更新されるたびにテスト
されますが、`expression` 属性を追加すると、expression が有効なデータを返す場合に
のみサブスクリプションが起動されます。例えば
、`"q": "shelfCount<10;refStore==urn:ngsi-ld:Store:001"` は、`shelfCount`が 10
以下で、アイテムがストア 001 にあることをテストします。これは、他のストアが通知
によって邪魔されないようにビジネス・ロジックを設定できることを意味します。

#### :three: リクエスト :

次のコマンドは、ストア 001 の在庫不足通知です :

```console
curl -iX POST \
  --url 'http://localhost:1026/v2/subscriptions' \
  --header 'Content-Type: application/json' \
  --data '{
  "description": "Notify me of low stock in Store 001",
  "subject": {
    "entities": [{"idPattern": ".*","type": "InventoryItem"}],
    "condition": {
      "attrs": ["shelfCount"],
      "expression": {"q": "shelfCount<10;refStore==urn:ngsi-ld:Store:001"}
    }
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/subscription/low-stock-store001"
    },
    "attrsFormat" : "keyValues"
  }
}'
```

#### :four: リクエスト :

次のコマンドは、ストア 002 の在庫不足通知です :

```console
curl -iX POST \
  --url 'http://localhost:1026/v2/subscriptions' \
  --header 'Content-Type: application/json' \
  --data '{
  "description": "Notify me of low stock in Store 002",
  "subject": {
    "entities": [{"idPattern": ".*", "type": "InventoryItem"}],
    "condition": {
      "attrs": ["shelfCount"],
      "expression": {"q": "shelfCount<10;refStore==urn:ngsi-ld:Store:002"}
    }
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/subscription/low-stock-store002"
    },
    "attrsFormat" : "keyValues"
  }
}'
```

2 つのリクエストは非常に似ています。これは単に、異なる `url' と 'expression' 属
性です。影響を受けた **InventoryItem** エンティティが ストア 001 への参照を持っ
ている場合は、最初の cUrl コマンドが起動し、影響を受けた **InventoryItem** エン
ティティが ストア 002 への参照を持っている場合は、二番目の cUrl コマンドが起動し
ます。

> **ヒント** : 次のように PUT リクエストを行うことで、在庫レベルを直接設定するこ
> とができます :
>
> ```console
> curl -iX PUT \
>  --url 'http://localhost:1026/v2/entities/urn:ngsi-ld:InventoryItem:005/attrs/shelfCount/value' \
>  --header 'Content-Type: text/plain' \
>  --data 5
> ```

ストア 002 からアイテムを購入すると、**InventoryItem** が 10 個以下になると、次
のようになります :

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/low-stock-monitor.png)

ご覧のように、影響を受けた **InventoryItem** のキー・バリューのペアが在庫管理フ
ロンドエンドに渡されました。

ストア自体を見ると :

#### `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![](https://fiware.github.io/tutorials.Subscriptions/img/low-stock-warehouse.png)

アプリケーション内のビジネス・ロジックによって警告が発生しました。

<A name="subscription-crud-actions"></A>

# サブスクリプションの CRUD アクション

サブスクリプションの **CRUD** 操作は、`/v2/subscriptions/` エンドポイントの下に
ある HTTP 動詞にマップされます。

-   **Create** - HTTP POST
-   **Read** - HTTP GET
-   **Update** - HTTP PATCH
-   **Delete** - HTTP DELETE

`<subscription-id>` は、サブスクリプションが作成され、その後の操作で使用される
POST レスポンスのヘッダに返されると自動的に生成されます。

<A name="creating-a-subscription"></A>

### サブスクリプションの作成

この例では、新しいサブスクリプションを作成します。サブスクリプションは、コンテキ
ストが変更され、サブスクリプションの条件(製品価格の変更)が満たされると、URL への
非同期通知を発行します。

`/v2/subscriptions/` エンドポイントに POST リクエストを行うことで、新しいサブス
クリプションを追加できます。

**Product** エンティティの `price` 属性が変更されたときはいつでも、サブスクリプ
ションが発生することがリクエストの `subject` セクションに記載されています。

ボディの `notification` セクションには、影響を受けるすべてのエンティティを含む
POST リクエストが `http://tutorial:3000/subscription/price-change endpoint` エン
ドポイントに送信されることが記載されています。

#### :five: リクエスト :

```console
curl -iX POST \
  --url 'http://localhost:1026/v2/subscriptions' \
  --header 'content-type: application/json' \
  --data '{
  "description": "Notify me of all product price changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*", "type": "Product"
      }
    ],
     "condition": {
      "attrs": [ "price" ]
    }
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/subscription/price-change"
    }
  }
}'
```

<A name="delete-a-subscription"></A>

### サブスクリプションの削除

この例では、id `5ae079b86e4f353c5163c939` を持つサブスクリプションをコンテキスト
から削除します。

サブスクリプションは、`/v2/subscriptions/<subscription-id>` エンドポイントに
DELETE リクエストを行うことで削除できます。

#### :six: リクエスト :

```console
curl -X DELETE \
  --url 'http://localhost:1026/v2/subscriptions/5ae079b86e4f353c5163c939'
```

<A name="update-an-existing-subscription"></A>

### 既存のサブスクリプションの更新

この例では、ID `5ae07c7e6e4f353c5163c93e` を持つ既存のサブスクリプションを修正し
、通知 URL を更新します。

サブスクリプションを更新して、`/v2/subscriptions/<subscription-id>` エンドポイン
トへの PATCH リクエストを行うことができます。

#### :seven: リクエスト :

```console
curl -iX PATCH \
  --url 'http://localhost:1026/v2/subscriptions/5ae07c7e6e4f353c5163c93e' \
  --header 'content-type: application/json' \
  --data '{
    "status": "active",
    "notification": {
        "http": {
            "url": "http://tutorial:3000/notify/price-change"
        }
    }
}'
```

<A name="list-all-subscriptions"></A>

### すべてのサブスクリプションの一覧表示

この例では、`/v2/subscriptions/` エンドポイント に GET リクエストを行うことによ
って、すべてのサブスクリプションを一覧表示します。

各サブスクリプションの通知セクションには、サブスクリプションの条件が最後に満たさ
れた時刻、および POST アクションが成功したかどうかが含まれます。

#### :eight: リクエスト :

```console
curl -X GET \
  --url 'http://localhost:1026/v2/subscriptions'
```

<A name="update-an-existing-subscription"></A>

### サブスクリプションの詳細を読み込む

この例では、特定の ID を持つサブスクリプションの完全な詳細を取得します。

レスポンスには、`notification` セクションに、サブスクリプションの条件が最後に満
たされたことと、POST アクションが成功したかどうかを示す追加の詳細が含まれていま
す。

サブスクリプションの詳細は、`/v2/subscriptions/<subscription-id>` エンドポイント
に GET リクエストを行うことで読み取ることができます。

#### :nine: リクエスト :

```console
curl -X GET \
  --url 'http://localhost:1026/v2/subscriptions/5aead3361587e1918de90aba'
```

<a name="next-steps"></a>

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズ
の[他のチュートリアル](https://www.letsfiware.jp/fiware-tutorials)を読むことで見
つけることができます

---

## License

[MIT](LICENSE) © 2018-2022 FIWARE Foundation e.V.
