# Subscriptions[<img src="https://img.shields.io/badge/NGSI-LD-d6604d.svg" width="90"  align="left" />](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.07.01_60/gs_cim009v010701p.pdf)[<img src="https://fiware.github.io/tutorials.Subscriptions/img/fiware.png" align="left" width="162">](https://www.fiware.org/)<br/>

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Subscriptions.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/ngsi-ld-tutorials.svg)](https://ngsi-ld-tutorials.rtfd.io)
<!-- prettier-ignore -->

このチュートリアルでは、NGSI-LD ユーザに、コンテキスト・データ・サブスクリプションを作成および管理する方法について
説明します。チュートリアルは、
ユーザが [NGSI-LD](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.07.01_60/gs_cim009v010701p.pdf)
サブスクリプション/通知のパラダイムを理解し、独自のコード内で NGSI サブスクリプションを使用する方法を説明するため、
以前の例で作成したエンティティと [スマート・ファーム](https://github.com/FIWARE/tutorials.Getting-Started/tree/NGSI-LD)
・アプリケーションに基づいて構築されています。

このチュートリアルでは、[cUrl](https://ec.haxx.se/) コマンドを組み合わせて、ブラウザ内で実行されるデバイスと
アクションを示します。cUrl コマンドは 、[Postman documentation](https://fiware.github.io/tutorials.Subscriptions/ngsi-ld)
としても利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/7f7b555ed4169e27bcef)
[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/FIWARE/tutorials.Subscriptions/tree/NGSI-LD)

## コンテンツ

<details>
<summary>詳細 <b>(クリックして拡大)</b></summary>

-   [状態の変更をサブスクリプション](#subscribing-to-changes-of-state)
    -   [スマート・アグリフード・システム内のエンティティ](#entities-within-a-smart-agrifood-system)
    -   [ファーム管理情報システム・フロントエンド](#farm-management-information-system-frontend)
-   [アーキテクチャ](#architecture)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [起動](#start-up)
-   [サブスクリプションの使用](#using-subscriptions)
    -   [単純なサブスクリプションの設定](#setting-up-a-simple-subscription)
-   [MQTT サーバへの通知の送信](#sending-notifications-to-an-mqtt-server)
    -   [MQTT サブスクリプションの設定](#setting-up-an-mqtt-subscription)
-   [サブスクリプションの CRUD アクション](#subscription-crud-actions)
    -   [サブスクリプションの作成](#creating-a-subscription)
    -   [サブスクリプションの削除](#delete-a-subscription)
    -   [既存のサブスクリプションの更新](#update-an-existing-subscription)
    -   [すべてのサブスクリプションの一覧表示](#list-all-subscriptions)
    -   [サブスクリプションの詳細を読み込む](#read-the-detail-of-a-subscription)
-   [次のステップ](#next-steps)

</details>

<A name="subscribing-to-changes-of-state"></A>

# 状態の変更をサブスクリプション

> 'Another sandwich!' said the King.
>
> 'There's nothing but hay left now,' the Messenger said, peeping into the bag.
>
> 'Hay, then,' the King murmured in a faint whisper.
>
> Alice was glad to see that it revived him a good deal. 'There's nothing like eating hay when you're faint,' he
> remarked to her, as he munched away.
>
> — Lewis Carroll (Through the Looking-Glass and What Alice Found There)

FIWARE プラットフォームでは、エンティティは、実世界に存在する物理的または概念的オブジェクトの状態を表します。
すべてのスマートなソリューションは、特定の瞬間にこれらのオブジェクトの現在の状態を知る必要があります。

これらの各エンティティのコンテキストは常に変化しています。 たとえば、スマート・ファームの例では、動物や車両が
移動したり、土壌が乾いたり、タスクがファームに割り当てられて完了したりすると、コンテキストが変化します。
IoT センサ・データに基づくスマート・ソリューションの場合、システムは常に現実世界の変化に反応するため、
この問題はさらに差し迫っています。

これまで、システムの状態を変更するために使用したすべての操作は**同期**していました。変更はユーザまたは
アプリケーションによって直接行われ、結果が通知されました。Orion Context Broker は、**非同期**の
通知メカニズムも提供しています。アプリケーションは、コンテキスト情報の変更をサブスクライブして、
何か起きたときに通知を受け取ることができます。これは、アプリケーションが継続的にポーリングまたは
クエリ・リクエストを繰り返す必要がないことを意味します。

したがって、サブスクリプション・メカニズムを使用すると、システム内のコンポーネント間で渡されるリクエスト量と
データ量の両方が削減されます。このネットワーク・トラフィックの減少は、全体的な応答性を改善します。

<A name="entities-within-a-smart-agrifood-system"></A>

## スマート・アグリフード・システム内のエンティティ

エンティティ間のリレーションシップは、次のように定義されます :

![](https://fiware.github.io/tutorials.Subscriptions/img/ngsi-ld-entities.png)

<A name="farm-management-information-system-frontend"></A>

## ファーム管理情報システム・フロントエンド

以前のチュートリアルでは、単純な Node.js Express アプリケーションが作成されました。このチュートリアルでは、
モニタ・ページを使用して最近のリクエストのステータスを監視し、デバイス・ページを使用してファーム上のマシンを
変更します。サービスが実行されると、これらのページには次の URL からアクセスできます。

#### イベント・モニタ

イベント・モニタは次の場所にあります : `http://localhost:3000/app/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Subscriptions/img/monitor.png)

#### デバイス・モニタ

このチュートリアルの目的のために、一連のダミーの農業用 IoT デバイスが作成され、Context Broker に接続されます。
使用されているアーキテクチャとプロトコルの詳細は、
[IoT センサ・チュートリアル](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-LD) にあります。
各デバイスの状態は、UltraLight デバイス・モニタの Web ページにあります:
`http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Subscriptions/img/farm-devices.png)

<A name="architecture"></A>

# アーキテクチャ

このアプリケーションは、
[Orion-LD Context Broker](https://fiware-orion.readthedocs.io/en/latest/),
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/),
というの2つの FIWARE コンポーネントのみを使用します。アプリケーションを _"Powered by FIWARE"_ と認定するには、
NGSI-LD Context Broker を使用するだけで十分です。

現在、NGSI-LD Context Broker はオープンソースの [MongoDB](https://www.mongodb.com/) 技術を利用して、コンテキスト・
データの永続性を維持しています。さらに、チュートリアル・アプリケーションを使用します。

したがって、アーキテクチャは 4 つの要素で構成されます :

-   FIWARE
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は、
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
    を使用してリクエストを受信します
-   FIWARE
    [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    は、
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
    を使用してノースバウンド・リクエストを受信し、それらを
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    に変換します
-   [MongoDB](https://www.mongodb.com/) データベース :
    -   **Orion Context Broker** が、データ・エンティティ、サブスクリプション、レジストレーションなどのコンテキスト・
        データ情報を保持するために使用します
    -   デバイスの URLs や Keys などのデバイス情報を保持するために **IoT Agent** によって使用されます
-   **チュートリアル・アプリケーション** は次のことを行います:
    -   HTTP上で実行される [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
        プロトコルを使用して、ダミーの[農業用 IoT デバイス](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-LD)
        のセットとして機能します

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコンテナ化され、公開されたポートから
実行されます。

![](https://fiware.github.io/tutorials.Subscriptions/img/architecture-ld.png)

必要な設定情報は、関連する `docker-compose.yml` ファイルの services セクションにあります。
以前のチュートリアルで説明しました。

<A name="prerequisites"></A>

# 前提条件

<A name="prerequisites"></A>

## Docker

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com) を使用して実行されます。
**Docker** は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

-   Docker を Windows にインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の手順に
    従ってください
-   Docker を Mac にインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の手順に
    従ってください -   Docker を Linux にインストールするには、[こちら](https://docs.docker.com/install/)
    の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。
[YAML file](https://raw.githubusercontent.com/FIWARE/tutorials.Subscriptions/NGSI-LD/docker-compose/orion-ld.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つまり、すべてのコンテナ・サービスは
1つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac
の一部としてインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている
手順に従う必要があります。

次のコマンドを使用して、現在の **Docker** バージョンと **Docker Compose** バージョンを確認できます :

```console
docker-compose -v
docker version
```

Docker バージョン 20.10 以降と Docker Compose 1.29 以上を使用していることを確認し、
必要に応じてアップグレードしてください。

<A name="prerequisites"></A>

## Cygwin

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/)
をダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。

<A name="start-up"></A>

# 起動

リポジトリ内で提供される bash スクリプトを実行すると、コマンドラインからすべての
サービスを初期化できます。リポジトリを複製し、以下のコマンドを実行して必要なイメ
ージを作成してください :

```console
git clone https://github.com/FIWARE/tutorials.Subscriptions.git
cd tutorials.Subscriptions
git checkout NGSI-LD

./services create;
./services orion;
```

このコマンドは、起動時に前のファーム管理情報システムの例からシードデータをインポートし、ファームに一連のダミー
IoT デバイスをプロビジョニングします。

> :information_source: **注 :** クリーンアップをやり直したい場合は、次のコマンド
> を使用して再起動することができます :
>
> ```console
> ./services stop
> ```

<a name="using-subscriptions"></a>

# サブスクリプションの使用

チュートリアルを正しく実行するには、cUrl コマンドを入力する前に、ブラウザの別々のタブで次の2つのページを
使用できることを確認してください。

#### ファーム管理情報システム (FMIS System)

農場周辺のさまざまな建物の詳細は、チュートリアル・アプリケーションにあります。
`http://localhost:3000/app/farm/urn:ngsi-ld:Building:farm001` を開いて、関連する充填センサ (filling sensor)
とサーモスタット (thermostat) を備えた建物を表示します。

![](https://fiware.github.io/tutorials.Subscriptions/img/fmis.png)

#### イベント・モニタ

イベント・モニタは次の場所にあります : `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/low-stock-farm.png)

<A name="setting-up-a-simple-subscription"></A>

## 単純なサブスクリプションの設定

ファーム管理情報システム内で、干し草のレベルが設定レベルを下回ったときに、農家が請負業者に干し草を納屋に
補充することを望んでいると想像してください。 請負業者が常に新しい情報をポーリングするようにシステムを
設定することは可能ですが、干し草はあまり頻繁に除去されないため、リソースの浪費になり、不要なデータ・
トラフィックが大量に発生します。

別の方法は、値が変更されるたびに "既知の" URL にペイロードを POST するサブスクリプションを作成することです。
以下に示すように、`/ngsi-ld/v1/subscriptions/` エンドポイントに POST リクエストを行うことで、
新しいサブスクリプションを追加できます:

#### :one: リクエスト:

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
-H 'NGSILD-Tenant: openiot' \
--data-raw '{
  "description": "Notify me of low feedstock on Farm:001",
  "type": "Subscription",
  "entities": [{"type": "FillingLevelSensor"}],
  "watchedAttributes": ["filling"],
  "q": "filling>0.6;filling<0.8;controlledAsset==urn:ngsi-ld:Building:farm001",
  "notification": {
    "attributes": ["filling", "controlledAsset"],
    "format": "keyValues",
    "endpoint": {
      "uri": "http://tutorial:3000/subscription/low-stock-farm001",
      "accept": "application/json"
    }
  },
   "@context": "http://context/ngsi-context.jsonld"
}'
```

POST リクエストのボディは2つの部分で構成され、リクエストの最初のセクション (`entities`, `type`,
`watchedAttributes`, `q` で構成）は、サブスクリプションの `filling` 属性がチェックされるたびに
チェックされることを示しています。 **FillingLevelSensor** エンティティが変更されます。これは `q`
パラメータによってさらに洗練され、実際のサブスクリプションは `filling` 属性が0.8を下回った場合のみ、
**Building** `urn:ngsi-ld:Building:farm001` にリンクされた **FillingLevelSensor** エンティティに対して
のみ起動されます。

ボディの `notification` セクションには、サブスクリプションの条件が満たされると、影響を受けるすべての
**FillingLevelSensor** エンティティを含んだ POST リクエストが URL
`http://tutorial:3000/subscription/low-stock-farm001`に送信されると記載されています。
これは請負業者自身のシステムによって処理されます。

IoT デバイスは現在、建物とは別のテナントを使用してプロビジョニングされているため、サブスクリプションは
`NGSILD-Tenant` ヘッダを使用していることに注意してください。テナントを使用すると、コンテキスト・データを
別々のデータベースに分散でき、複数のアプリケーション・クライアントが同じ Context Broker にアクセスできますが、
独自のデータセットは分離されます。

デバイスモニター `http://localhost:3000/app/farm/urn:ngsi-ld:Building:farm001` に移動し、納屋から干し草の
除去を開始します。納屋が半分空になるまで何も起こりません。その後、次のようにリクエストが
`subscription/low-stock-farm001` に送信されます:

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/low-stock-farm.png)

#### サブスクリプション・ペイロード:

```json
{
    "id": "urn:ngsi-ld:Notification:5fd0f3824eb81930c97005d8",
    "type": "Notification",
    "subscriptionId": "urn:ngsi-ld:Subscription:5fd0ee554eb81930c97005c1",
    "notifiedAt": "2020-12-09T15:55:46.520Z",
    "data": [
        {
            "id": "urn:ngsi-ld:Device:filling001",
            "type": "FillingLevelSensor",
            "controlledAsset": "urn:ngsi-ld:Building:farm001",
            "filling": 0.59
        }
    ]
}
```

ファーム管理情報システムのコードは、次のように 受信した POST リクエストを処理します:

```javascript
const NOTIFY_ATTRIBUTES = ["controlledAsset", "type", "filling", "humidity", "temperature"];

router.post("/subscription/:type", (req, res) => {
    monitor("notify", req.params.type + " received", req.body);
    _.forEach(req.body.data, (item) => {
        broadcastEvents(req, item, NOTIFY_ATTRIBUTES);
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

このビジネスロジックは、登録された関係者 (納屋を補充する請負業者など) にソケット I/O イベントを発行します。

#### :two: リクエスト:

この2番目のサブスクリプションは、`filling` level (充填レベル) が0.6〜0.4のときに発生します。`format` 属性は、
NGSI-LD 正規化フォーマット (normalized format) を使用してサブスクライバーに通知するように変更されました。

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
-H 'NGSILD-Tenant: openiot' \
--data-raw '{
  "description": "Notify me of low feedstock on Farm:001",
  "type": "Subscription",
  "entities": [{"type": "FillingLevelSensor"}],
  "watchedAttributes": ["filling"],
  "q": "filling>0.4;filling<0.6;controlledAsset==urn:ngsi-ld:Building:farm001",
  "notification": {
    "attributes": ["filling", "controlledAsset"],
    "format": "normalized",
    "endpoint": {
      "uri": "http://tutorial:3000/subscription/low-stock-farm001-ngsild",
      "accept": "application/json"
    }
  },
   "@context": "http://context/ngsi-context.jsonld"
}'
```

#### サブスクリプション・ペイロード:

`low-stock-farm001-ngsild` イベントが発生すると、レスポンスは次のようになります:

```json
{
    "id": "urn:ngsi-ld:Notification:5fd0fa684eb81930c97005f3",
    "type": "Notification",
    "subscriptionId": "urn:ngsi-ld:Subscription:5fd0f69b4eb81930c97005db",
    "notifiedAt": "2020-12-09T16:25:12.193Z",
    "data": [
        {
            "id": "urn:ngsi-ld:Device:filling001",
            "type": "FillingLevelSensor",
            "filling": {
                "type": "Property",
                "value": 0.25,
                "unitCode": "C62",
                "observedAt": "2020-12-09T16:25:12.000Z"
            },
            "controlledAsset": {
                "type": "Relationship",
                "object": "urn:ngsi-ld:Building:farm001",
                "observedAt": "2020-12-09T16:25:12.000Z"
            }
        }
    ]
}
```

`accept` 属性が `application/json` に設定されているため、`@context` はペイロード・ボディ内の属性ではなく
`Link` ヘッダとして送信されます。

#### :three: リクエスト:

Context broker は、追加のカスタム・ペイロード形式 (通常は `x-` で始まる) を提供する場合があります。Orion-LD
Broker は、レガシー・システムに下位互換性のある **NGSI-v2** ペロード・オプションを提供します。

この3番目のサブスクリプションは、`filling` レベルが0.4未満のときに発生します。`format` 属性は、NGSI-v2
正規化フォーマットを使用してサブスクライバーに通知するように変更されました。

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
-H 'NGSILD-Tenant: openiot' \
--data-raw '{
  "description": "Notify me of low feedstock on Farm:001",
  "type": "Subscription",
  "entities": [{"type": "FillingLevelSensor"}],
  "watchedAttributes": ["filling"],
  "q": "filling<0.4;controlledAsset==urn:ngsi-ld:Building:farm001",
  "notification": {
    "attributes": ["filling", "controlledAsset"],
    "format": "x-ngsiv2-normalized",
    "endpoint": {
      "uri": "http://tutorial:3000/subscription/low-stock-farm001-ngsiv2",
      "accept": "application/json"
    }
  },
   "@context": "http://context/ngsi-context.jsonld"
}'
```

#### サブスクリプション・ペイロード:

`low-stock-farm001-ngsiv2` イベントが発生すると、レスポンスは次のように正規化された NGSI-v2
ペイロードになります:

```json
{
    "subscriptionId": "urn:ngsi-ld:Subscription:5fd1f31e8b9b83697b855a5d",
    "data": [
        {
            "id": "urn:ngsi-ld:Device:filling001",
            "type": "https://uri.etsi.org/ngsi-ld/default-context/FillingLevelSensor",
            "https://w3id.org/saref#fillingLevel": {
                "type": "Property",
                "value": 0.33,
                "metadata": {
                    "unitCode": "C62",
                    "accuracy": {
                        "type": "Property",
                        "value": 0.05
                    },
                    "observedAt": "2020-12-10T10:11:57.000Z"
                }
            },
            "https://uri.etsi.org/ngsi-ld/default-context/controlledAsset": {
                "type": "Relationship",
                "value": "urn:ngsi-ld:Building:farm001",
                "metadata": {
                    "observedAt": "2020-12-10T10:11:57.000Z"
                }
            }
        }
    ]
}
```

ご覧のとおり、デフォルトでは、属性は URN の長い名前を使用して返されます。Orion-LD Context Broker が
ペイロードに圧縮操作を事前に適用するようにリクエストすることもできます。

-   `x-nsgiv2-keyValues` - URN 属性名を持つキーと値のペア
-   `x-nsgiv2-keyValues-compacted` - 短い名前の属性エイリアスを持つキーと値のペア
-   `x-ngsiv2-normalized` - URN 属性名を持つ NGSI-v2 正規化ペイロード
-   `x-ngsiv2-normalized-compacted`- 短い名前の属性エイリアスを持つ NGSI-v 2正規化ペイロードのペア

使用可能なカスタム形式のセットは、Context Broker によって異なります。

<a name="sending-notifications-to-an-mqtt-server"></a>

# MQTT サーバへの通知の送信

"MQTTは、IoT (the internet of Things) で使用されるパブリッシュ/サブスクライブ・ベースのメッセージング・プロトコルです。TCP/ IP
プロトコル上で機能し、"小さなコード・フットプリント" が必要とされるか、またはネットワーク帯域幅に制限がある、リモート・
ロケーションとの接続用に設計されています。目標は、帯域幅効率が高く、バッテリー電力をほとんど使用しないプロトコルを提供すること
です。"<sup>[1](#footnote1)</sup> NGSI-LD Context brokers は、HTTP 経由で通知を送信するのと同じくらい簡単に MQTT 経由で通知を送信
できます。

<a name="setting-up-an-mqtt-subscription"></a>

## MQTT サブスクリプションの設定

この `keyValues` サブスクリプションは、`filling` レベルが 0.4 と 0.2 の間にあるときに起動します。`endpoint` 属性が MQTT
プロトコルを使用するように変更されました。

#### :four: リクエスト:

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
-H 'NGSILD-Tenant: openiot' \
--data-raw '{
  "description": "Notify me of low feedstock on Farm:001",
  "type": "Subscription",
  "entities": [{"type": "FillingLevelSensor"}],
  "watchedAttributes": ["filling"],
  "q": "filling>0.2;filling<0.4;controlledAsset==urn:ngsi-ld:Building:farm001",
  "notification": {
    "attributes": ["filling", "controlledAsset"],
    "format": "keyValues",
    "endpoint": {
      "uri": "mqtt://mosquitto:1883/entities",
      "accept": "application/json",
      "notifierInfo": [
        {
          "key": "MQTT-QoS",
          "value": "1"
        }
      ]
    }
  },
   "@context": "http://context/ngsi-context.jsonld"
}'
```

### MQTT サブスクライバを開始 (:new: ターミナル)

通信回線が開いていることを確認するために、特定のトピックをサブスクライブし、メッセージが公開されたときに
何かを受信できることを確認できます。

**新しいターミナル**を開き、次のように新しい実行中の `mqtt-subscriber` Docker コンテナを作成します:

```console
docker run -it --rm --name mqtt-subscriber \
  --network fiware_default efrecon/mqtt-client sub -h mosquitto -t "/#"
```

これで、端末はイベントを受信する準備が整います。

<a name="subscription-crud-actions"></a>

# サブスクリプションの CRUD アクション

サブスクリプションの **CRUD** 操作は、`/ngsi-ld/v1/subscriptions/` エンドポイントの下で予想される
HTTP 動詞にマップされます。

-   **Create** - HTTP POST
-   **Read** - HTTP GET
-   **Update** - HTTP PATCH
-   **Delete** - HTTP DELETE

`<subscription-id>` は、サブスクリプションが作成され、POST レスポンスのヘッダに返されるときに自動生成され、
その後、他の操作で使用されます。

<a name="creating-a-subscription"></a>

### サブスクリプションの作成

この例では、新しいサブスクリプションを作成します。サブスクリプションは、コンテキストが変更され、
サブスクリプションの条件 (製品価格の変更) が満たされるたびに、URL への非同期通知を発行します。

`/ngsi-ld/v1/subscriptions/` エンドポイントに POST リクエストを行うことで、新しいサブスクリプションを
追加できます。

リクエストの `subject` セクションには、`Product` エンティティの価格属性が変更されるたびに
サブスクリプションが発生することが記述されています。

ボディの `notification` セクションには、影響を受けるすべてのエンティティを含む POST リクエストが
`http://tutorial:3000/subscription/price-change` エンドポイントに送信されることが記述されています。

#### :four: リクエスト:

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/subscriptions/' \
-H 'Content-Type: application/ld+json' \
--data-raw '{
  "description": "Notify me of all product price changes",
  "type": "Subscription",
  "entities": [{"type": "Product"}],
  "watchedAttributes": ["price"],
  "notification": {
    "format": "keyValues",
    "endpoint": {
      "uri": "http://tutorial:3000/subscription/price-change",
      "accept": "application/json"
    }
  },
   "@context": "http://context/ngsi-context.jsonld"
}'
```

<a name="delete-a-subscription"></a>

### サブスクリプションの削除

この例では、`id=urn:ngsi-ld:Subscription:5fd228838b9b83697b855a72` のサブスクリプションをコンテキスト
から削除します。

サブスクリプションは、`/ngsi-ld/v1/subscriptions/<subscription-id>` エンドポイントに DELETE
リクエストを行うことで削除できます。

#### :five: リクエスト:

```console
curl -X DELETE \
  --url 'http://localhost:1026/ngsi-ld/v1/subscriptions/urn:ngsi-ld:Subscription:5fd228838b9b83697b855a72'
```

<a name="update-an-existing-subscription"></a>

### 既存のサブスクリプションの更新

この例では、ID が `urn:ngsi-ld:Subscription:5fd228838b9b83697b855a72` の既存のサブスクリプションを
修正し、通知 URL を更新します。

サブスクリプションは、`/ngsi-ld/v1/subscriptions/<subscription-id>` エンドポイントに PATCH
リクエストを行うことで更新できます。

#### :six: リクエスト:

```console
curl -iX PATCH \
  --url 'http://localhost:1026/ngsi-ld/v1/subscriptions/urn:ngsi-ld:Subscription:5fd228838b9b83697b855a72' \
  --header 'content-type: application/json' \
  --data '{
  "notification": {
    "format": "normalized",
    "endpoint": {
      "uri": "http://tutorial:3000/subscription/price-change",
      "accept": "application/json"
    }
  }
}'
```

<a name="list-all-subscriptions"></a>

### すべてのサブスクリプションの一覧表示

この例では、 `/ngsi-ld/v1/subscriptions/` エンドポイントに GET リクエストを送信してすべてのサブスクリプションを
一覧表示します。サブスクリプションのリストは、`NGSILD-Tenant` ヘッダで定義されたテナント (または、
`NGSILD-Tenant` ヘッダが送信されない場合はデフォルトのテナント) に制限されます。

各サブスクリプションの `notification` セクションには、サブスクリプションの条件が最後に満たされた時刻、
および関連付けられた POSTアクションが成功したかどうかも含まれます。

#### :seven: リクエスト:

```console
curl -X GET \
  --url 'http://localhost:1026/ngsi-ld/v1/subscriptions/
```

<a name="read-the-detail-of-a-subscription"></a>

### サブスクリプションの詳細を読み込む

この例では、特定の ID を持つサブスクリプションの完全な詳細を取得します。

レスポンスには、サブスクリプションの条件が最後に満たされた時刻、および関連付けられた POST アクションが
成功したかどうかを示す `notification` セクションの追加の詳細が含まれます。

サブスクリプションの詳細は、`/ngsi-ld/v1/subscriptions/<subscription-id>` エンドポイントに GET
リクエストを行うことで読み取ることができます。

#### :eight: リクエスト:

```console
curl -X GET \
  --url 'http://localhost:1026/ngsi-ld/v1/subscriptions/5aead3361587e1918de90aba'
```

<a name="next-steps"></a>

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズ
の[他のチュートリアル](https://www.letsfiware.jp/ngsi-ld-tutorials)を読むことで見
つけることができます

---

## License

[MIT](LICENSE) © 2021-2023 FIWARE Foundation e.V.

### 脚注

<a name="footnote1"></a>

-   [Wikipedia: MQTT](https://en.wikipedia.org/wiki/MQTT) - サービス間のすべてのメッセージのディスパッチを担当する
    中央通信ポイント (MQTT broker と呼ばれる)
