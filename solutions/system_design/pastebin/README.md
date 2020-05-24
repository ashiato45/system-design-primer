# Design Pastebin.com (or Bit.ly)

*Note: This document links directly to relevant areas found in the [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) to avoid duplication.  Refer to the linked content for general talking points, tradeoffs, and alternatives.*

訳注: pastebin.comは、短かい文章を投稿でき、それに短かいURLを付与してくれるサービスです。

**Bit.lyを設計せよ**は似た問題ですが、短縮していない元URLを覚えておくかわりにpastebinはコンテンツを保持する必要があります。

## Step 1: 大体のユースケースと想定制約

> 問題の要件と範囲の情報を集めよう。ユースケースと想定される制約をはっきりさせるために質問をしよう。システムの前提について議論しよう。

今回は質問をする相手がいないので、ユースケースと制約については示しておくことにします。

### ユースケース

#### 今回は次のユースケースのみを扱う問題に取り組みます
(訳注: ランダム生成されたlinkのことをpasteとよんでいるようです)
* **ユーザ**は文章を入力し、ランダムに生成されたリンクを得る
    * 失効
        * デフォルト設定では失効しない
        * オプションで失効する時限がつけられる
* **ユーザ**はpasteのURLを入力して、内容を見る
* **ユーザ**は匿名である
* **サービス**はページのanalyticsを記録する
    * 月々の訪問の統計
* **サービス**は失効したpasteを消去する
* **サービス**には高い可用性(availability)がある

#### 今回取り組まないこと

* **ユーザ**はアカウントを登録する
    * **ユーザ**はメールを確認する
* **ユーザ**が登録ずみのアカウントにログインする
    * **ユーザ**が文書を編集する
* **ユーザ**が閲覧可能不可能を設定する
* **ユーザ**がshort linkを設定できる

### 制約と前提

#### 稼動している状態についての前提

* トラフィック(一定時間の通信量)は均等に分布しない
* short linkをたどるのは高速である
* pasteは文章のみ
* 閲覧数のanalyticsはリアルタイムである必要はない
* 10Mぐらいのユーザ
* 一ヶ月に10Mのpasteの書き込み
* 一ヶ月に100Mのpasteの読み込み
* 読み書きの比は10:1


#### 使用量を計算する

**使用量を封筒裏計算(ざっくり計算すること)する必要があるかどうかは面接官に聞いてはっきりさせましょう**

* paste1個あたりのサイズ
    * コンテンツは1KB/paste
    * `shortlink` - 7バイト
    * `expiration_length_in_minutes` - 4バイト
        * 訳注: 4バイトだと2の32乗分で、log(2^32)=32log(2)=10なので、1e10分=1e8時間=4e6日=1e5ヶ月=1000年が最大になる。int使うのが簡単だしまあそうか。
    * `created_at` - 5バイト
        * 訳注: MySQLだとcreated_atはDATETIME型で、精度によっては5バイトで済むらしい。どうせ概算なんだし8バイトということにしておいていいような気がするが…
    * `paste_path` - 255バイト
    * 合計 = ～1.27 KB
* 一ヶ月に12.7GBのpasteコンテンツ
    * 1.27KB/paste * 10M pastes/月
        * 訳注: (1.27 * 10)(K * M)B/月 = 12.7GB/月
    * 3年で～450GBの新しいpasteコンテンツ
        * 訳注: 3年ってどこから出てきたんだ？
    * 3年で360Mのshort link
        * 訳注: 10M pastes/月 * 12月/年 * 3年 = 360M pastes
    * 既存のを上書きするのではなく、ほとんどが新しいpasteだと仮定する
* 平均で一秒あたり4pasteの書き込み 
    * 訳注: 10M paste/月 * (1/30)月/日 * (1/86400)日/秒 = (10M / 2400000) paste/秒 = 10/2.5 paste/秒 = 4 paste/秒。1月は大体2.5M秒なのね。
* 平均で一秒あたり40の読み込みリクエスト

便利な変換ガイド:
* 2.5M 秒/月
* 1リクエスト/秒 = 2.5Mリクエスト/月
* 40リクエスト/秒 = 100Mリクエスト/月
* 400リクエスト/秒 = 1Gリクエスト/月 (訳注: もともとbillionって書いてありましたがSI補助のほうが日本人的には楽なので…。10の9乗です。)

## Step 2: ハイレベルな(おおまかな)設計を作ろう

> 重要な要素をすべて含んだハイレベルデザインをざっくり作ろう。

![Imgur](http://i.imgur.com/BKsBnmG.png)

訳注: なんでAnalyticsはObject storeにのびてるの？

## Step 3: コア部品を設計しよう

> それぞれのコア部品の詳細に分け入ろう。

### ユースケース: ユーザが文章を入力して、ランダム生成されたリンクを得る

[リレーショナルデータベース](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%AA%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%8A%E3%83%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%83%9E%E3%83%8D%E3%82%B8%E3%83%A1%E3%83%B3%E3%83%88%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0-rdbms)を、生成されたURLをpasteファイルを含むファイルサーバとパスにマップする巨大なハッシュテーブルとして利用することができるでしょう。

ファイルサーバへマップするかわりに、Amazon S3や[NoSQLドキュメントストア](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E3%82%B9%E3%83%88%E3%82%A2)といった、マネージドな(サービスとして提供されている)**オブジェクトストア**を利用することもできるでしょう。

巨大なハッシュテーブルとして機能するリレーショナルデータベースの代用品としては、[NoSQLキーバリューストア](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%AD%E3%83%BC%E3%83%90%E3%83%AA%E3%83%A5%E3%83%BC%E3%82%B9%E3%83%88%E3%82%A2)を利用することができるでしょう。これに関しては、[SQLとNoSQLのトレードオフ](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#sql%E3%81%8Bnosql%E3%81%8B)について議論する必要があるでしょう。次では、リレーショナルデータベースを利用するアプローチについて議論しています。

* **クライアント**は[リバースプロキシ](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%AA%E3%83%90%E3%83%BC%E3%82%B9%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7web%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC)として動作している**ウェブサーバ**にpaste作成リクエストを送信します。
* **ウェブサーバ**は**書き込みAPI**サーバにこのリクエストを転送します。
* **書き込みAPI**サーバは次のことをします:
    * 唯一性のあるURLを生成する
        * **SQLデータベース**を見て重複を探し、唯一性を確認する
        * もしURLが唯一でないなら、別のURLを生成する
        * もしカスタムURLをサポートするなら、ユーザから与えられたものを使う(やはり重複は確認する)
    * **SQLデータベース**の`pastes`テーブルに保存する
    * pasteのデータを**オブジェクトストア**に保存する
    * URLを返す

**どのぐらいのコードを書くことが期待されているのか面接官に聞いてはっきりさせよう。**

`pastes`テーブルは次のような構造を持つでしょう:
```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

ルックアップを高速化し(テーブル全体をなめる代わりにlog時間で)、メモリにデータを保持するために`shortlink`と`created_at`に[インデックス](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%A4%E3%83%B3%E3%83%87%E3%83%83%E3%82%AF%E3%82%B9%E3%82%92%E5%8A%B9%E6%9E%9C%E7%9A%84%E3%81%AB%E7%94%A8%E3%81%84%E3%82%8B)を作りましょう。1MBをメモリからシーケンシャルに読むには約250msかかる一方、SSDから読むには約4倍、ディスクから読むには約80倍かかります。<sup><a href=https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E5%85%A8%E3%81%A6%E3%81%AE%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9E%E3%83%BC%E3%81%8C%E7%9F%A5%E3%82%8B%E3%81%B9%E3%81%8D%E3%83%AC%E3%82%A4%E3%83%86%E3%83%B3%E3%82%B7%E3%83%BC%E5%80%A4>1</a></sup>


唯一のURLを生成するには、こんなことができるでしょう:

* ユーザのIPアドレスとタイムスタンプから[**MD5**](https://ja.wikipedia.org/wiki/MD5)ハッシュを生成する
    * MD5は128bitのハッシュ値を生成するのによく使われるハッシュ関数
    * MD5は一様に分布する
    * かわりに、ランダム生成されたデータのMD5ハッシュを使うこともできる
* [**Base62**](https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/)はMD5ハッシュをエンコードする
    * Base62はURLに適している`[a-zA-Z0-9]`にエンコードし、特殊文字をエスケープする必要をなくしている
    * オリジナルの入力に対してハッシュの結果は1つしかなく、Base62は決定的(deterministic、ランダム性がない)
    * Base64はまた別のエンコーディング法だが、`+`と`/`がさらにでるためにURLとして使うには問題がある
    * 以下の[Base62疑似コード](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener)はO(k)時間で動作する。ここで、kは桁数=7。
        * 訳注: 62進法変換とすべきか迷った

```python
def base_encode(num, base=62):
    digits = []
    while num > 0
      remainder = modulo(num, base)
      digits.push(remainder)
      num = divide(num, base)
    digits = digits.reverse
```

* 出力の先頭7文字をとると、62^7とおりの値ができ、これは3年で360Mのshortlinkという制約を扱うのに十分でしょう:

```python
url = base_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

公開された[**REST API**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#representational-state-transfer-rest)を利用しましょう:

```
$ curl -X POST --data '{ "expiration_length_in_minutes": "60", \
    "paste_contents": "Hello World!" }' https://pastebin.com/api/v1/paste
```
訳注: -XでHTTPメソッドを指定

Response:

```
{
    "shortlink": "foobar"
}
```

内部通信のためには、[遠隔手続呼出(RPC)](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E9%81%A0%E9%9A%94%E6%89%8B%E7%B6%9A%E5%91%BC%E5%87%BA-rpc)を使いましょう。

### ユースケース: ユーザがpasteのURLを入力し、コンテンツを見る

* **クライアント**は**ウェブサーバ**へのpasteを取得するためのリクエストを送信する
* **ウェブサーバ**はリクエストを**読み込みAPI**サーバに転送する
* **読み込みAPI**サーバは次のことをする:
    * 生成されたURLを**SQLデータベース**から探す
        * もしURLが**SQLデータベース**にあるなら、**オブジェクトストア**からpasteコンテンツを取得する
        * そうでないなら、ユーザにエラーメッセージを返す

REST API:

```
$ curl https://pastebin.com/api/v1/paste?shortlink=foobar
```

Response:

```
{
    "paste_contents": "Hello World"
    "created_at": "YYYY-MM-DD HH:MM:SS"
    "expiration_length_in_minutes": "60"
}
```

### ユースケース: サービスがページ訪問を解析する(訳注: Service tracks analytics of pagesだったが良い訳がわからん)

リアルタイムの解析は必要ないので、単に訪問数を生成するように**ウェブサーバ**のログを**MapReduce**すればよい。

**どのぐらいコードを書くことが要求されているか、面接官に聞いてはっきりさせましょう**

```python
class HitCounts(MRJob):

    def extract_url(self, line):
        """Extract the generated url from the log line."""
        ...

    def extract_year_month(self, line):
        """Return the year and month portions of the timestamp."""
        ...

    def mapper(self, _, line):
        """Parse each log line, extract and transform relevant lines.

        Emit key value pairs of the form:

        (2016-01, url0), 1
        (2016-01, url0), 1
        (2016-01, url1), 1
        """
        url = self.extract_url(line)
        period = self.extract_year_month(line)
        yield (period, url), 1

    def reducer(self, key, values):
        """Sum values for each key.

        (2016-01, url0), 2
        (2016-01, url1), 1
        """
        yield key, sum(values)
```

### ユースケース: サービスが失効したpasteを削除する

失効したpasteを削除するには、単に現在のタイムスタンプより古い失効したタイムスタンプのエントリを**SQLデータベース**をなめて削除すればよい。失効したエントリ全てはテーブルから削除(あるいは失効とマーク)する。

## Step 4: 設計をスケールさせる

> 制約のもとでボトルネックを特定して対処しましょう。

![Imgur](http://i.imgur.com/4edXG0T.png)

**重要: 初期設計から最終設計にいきなりジャンプするのはやめましょう！**

次のことを繰り返し行うことを言います: 1)**ベンチマーク/負荷試験**する、 2)ボトルネックを**プロファイル**する、 3)代替案とトレードオフを評価しながらボトルネックに対処する、4)これを繰り返す。最初の設計をどうやって繰り返しスケールさせていくかというサンプルとして、[AWSで何百万ものユーザにシステムをスケールさせるように設計する(英語版)](../scaling_aws/README.md)を見よ。

最初の設計でどんなボトルネックに出会すか、それらにどんな風に取り組めるかを議論するのが重要です。たとえば、複数の**ウェブサーバ**と**ロードバランサ**を追加することによってどんな問題に対処できるでしょう。**CDN**は？**マスタースレーブ レプリケーション**は？それぞれの代替案や**トレードオフ**は？

設計を完成させ、スケーラビリティの問題に取り組むための部品を紹介しましょう。内部のロードバランサはごちゃごちゃを避けるために除いてあります。

*議論の繰り返しを避けるために*、主な話題、トレードオフ、代替案のため[システム設計目次](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0%E8%A8%AD%E8%A8%88%E7%9B%AE%E6%AC%A1)を参照してください:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
* [Horizontal scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Web server (reverse proxy)](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [API server (application layer)](https://github.com/donnemartin/system-design-primer#application-layer)
* [Cache](https://github.com/donnemartin/system-design-primer#cache)
* [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL write master-slave failover](https://github.com/donnemartin/system-design-primer#fail-over)
* [Master-slave replication](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Consistency patterns](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [Availability patterns](https://github.com/donnemartin/system-design-primer#availability-patterns)

(訳注:リンクははりなおしてない)

**Analytics Database**としてAmazon RedshiftやGoogle Bigqueryといったデータウェアハウスのソリューションを利用することができるでしょう。

Amazon S3といった**オブジェクトストア**を使えば一ヶ月12.7GBという制約をかるがる扱うことができるでしょう。

*平均*秒間40の読み込みリクエスト(ピークではもっとある)に対応するには、人気のあるコンテンツのトラフィックはデータベースのかわりに**メモリキャッシュ**で扱うべきでしょう。**メモリキャッシュ**は不均等に分散したトラフィックとトラフィックの急激な上昇を扱うのにも便利です。**SQL読み込みレプリケーション**は、レプリケーションが書き込みのレプリケーションがうまくいっている限り、キャッシュミスに対応できるべきです。

*平均*秒間4の書き込みリクエスト(ピークではもっとある)が**SQL書き込みマスタースレーブ**で実行できなければいけません。さもなければ、SQLをスケールさせる追加のパターンを利用しなければいけません:

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

データを**NoSQLデータベース**に移すことも検討すべきでしょう。

## 追加で話すべきこと

> 問題の範囲や残り時間によって深掘りすべき追加の話題です。

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process

## 訳者: よくわからんこと/計算してないこと
* レプリケーションは何重にすべきなんだろう。
    * 読み込み要求は1個で間に合いそうな気がする。
        * DBには360Mのpasteがあるはずなので、そこから必要なものを探せなければいけない。インデックスが張ってあるのでlog_2(360M)が大体8なので全く間に合う気がする。