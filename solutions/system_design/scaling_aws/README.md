# AWSで何百万のユーザにスケールするシステムを設計しよう

訳注: 前提って訳したけど仮定のほうがよかったなあ

*注意: この文書は重複をさけるために[システム設計目次](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md)の隣接分野に直接リンクを張ってあります。一般に話すポイント、トレードオフ、代替案についてはリンク先のコンテンツをご覧ください。

## Step 1: ユースケースと制約をざっくり決めよう

> 問題の要件と範囲の情報を集めよう。ユースケースと想定される制約をはっきりさせるために質問をしよう。システムの前提について議論しよう。

今回は質問をする相手がいないので、ユースケースと制約については示しておくことにします。

### ユースケース

この問題を解くには繰り返しのアプローチを取ります:
 1)**ベンチマーク/負荷試験**する、 2)ボトルネックを**プロファイル**する、 3)代替案とトレードオフを評価しながらボトルネックに対処する、4)これを繰り返す。
 これが基本的な設計をスケーラブルな設計に発展させるための良いパターンです。

AWSの背景知識があるだとかAWSの知識が必要とされる職業に応募するだとかでなければ、AWS特有の知識は必要とされません。しかしながら、**この演習で議論される原則のほとんどはAWSのエコシステムの外にも一般的に適用できるものです。**

#### 今回は次のユースケースのみを扱う問題に取り組みます

* **ユーザ**は読み込みと書き込みのリクエストをする
    * **サービス**は処理、データの保存と結果の返却をする
* **サービス**は少数のユーザから何百万ものユーザにまでサービスを提供できるよう発展していかねばならない
    * たくさんのユーザとリクエストを捌くためのアーキテクチャの発展をするための一般的なスケーリングのパターンについて議論しましょう
* **サービス**は高い可用性(availability)を持つ

### 制約と前提

#### 稼動している状態についての前提

* トラフィックは均等に分布しない
* リレーショナルデータが必要
* 1ユーザから何千万ものユーザにスケールする
    * ユーザの増加を次のように記す:
        * Users+
        * Users++
        * Users+++
        * ...
    * 10Mユーザ
    * 一ヶ月に1G書き込み
    * 一ヶ月に100G読み込み
    * 100:1の読み込み書き込み比
    * 1KBコンテンツ/書き込み

#### 使用量を計算する

**使用量を封筒裏計算(ざっくり計算すること)する必要があるかどうかは面接官に聞いてはっきりさせましょう**

* 一ヶ月に1TBの新規コンテンツ
    * 1KB/書き込み * 1G書き込み/月
    * 3年で36TBの新規コンテンツ
    * ほとんどの書き込みは既存のものの更新ではなく新規コンテンツのためにおこる
* 平均400書き込み/秒
* 平均40K読み込み/秒

便利な変換ガイド:
* 2.5M 秒/月
* 1リクエスト/秒 = 2.5Mリクエスト/月
* 40リクエスト/秒 = 100Mリクエスト/月
* 400リクエスト/秒 = 1Gリクエスト/月 (訳注: もともとbillionって書いてありましたがSI補助のほうが日本人的には楽なので…。10の9乗です。)


## Step 2: ハイレベルな(おおまかな)設計を作ろう

> 重要な要素をすべて含んだハイレベルデザインをざっくり作ろう。

![Imgur](http://i.imgur.com/B8LDKD7.png)

## Step 3: コア部品を設計しよう

> それぞれのコア部品の詳細に分け入ろう。

### ユースケース: ユーザは書き込みと読み込みの要求をする

#### 目標

* 1-2ユーザなら基本的な設定で十分です
    * 簡単のため1マシン(box)でやる
    * 必要なときに垂直スケーリングする
    * ボトルネック特定のため監視する

#### 1マシンからはじめる

* **ウェブサーバ**をEC2上に
    * ユーザのデータを保存する
    * [**MySQLデータベース**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%AA%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%8A%E3%83%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%83%9E%E3%83%8D%E3%82%B8%E3%83%A1%E3%83%B3%E3%83%88%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0-rdbms)

**垂直スケーリング**をつかう:

* 単に強力なマシンを使う
* どうやってスケールアップするか決めるためにメトリックに注意する
    * ボトルネックを特定するため基本的なモニタリングを使う: CPU、メモリ、IO、ネットワーク、その他
    * Cloudwatch, top(訳注: プロセスの状況確認), nagios(訳注: ネットワークサービス諫死), statsd(訳注: ログの統計解析？), graphite, その他
* 垂直スケーリングはとても高額いなりうる
* 冗長性やフェイルオーバーはない

*トレードオフ、代替案、さらなる詳細:*
* **垂直スケーリング**の代替案は[**水平スケーリング**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E6%B0%B4%E5%B9%B3%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%AA%E3%83%B3%E3%82%B0)です。

#### SQLから始め、NoSQLを検討する

制約によるとリレーショナルデータが必要です。1マシンで**MySQLデータベース**を使うところからスタートしてみましょう。

**トレードオフ、代替案、さらなる詳細**
* See the [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) section
* Discuss reasons to use [SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

#### パブリックな静的IPを割り当てる

* エラスティックIPは再起動してもIPアドレスのかわらないパブリックなエンドポイントとして使えます
* フェイルオーバーに便利です。ただ新しいIPアドレスにドメインを振り向けるだけです。


#### DNSをつかう

Route53といった**DNS**を追加し、ドメインをにインスタンスのパブリックIPにマップしましょう。

**トレードオフ、代替案、さらなる詳細**

* See the [Domain name system](https://github.com/donnemartin/system-design-primer#domain-name-system) section

#### ウェブサーバをセキュアにする

* 必要なポートだけを開けましょう
    * ウェブサーバはここからのインカミングリクエストだけに応答する:
        * HTTPの80番
        * HTTPSの443番
        * ホワイトリストに入っているIPにだけSSHの22番
    * アウトバウンド接続を開始するのはできないようにする

**トレードオフ、代替案、さらなる詳細**

* See the [Security](https://github.com/donnemartin/system-design-primer#security) section

## Step 4: 設計をスケールさせる

> 制約のもとでボトルネックを特定し対処する

### Users+

![Imgur](http://i.imgur.com/rrfjMXB.png)

#### 前提

ユーザ数は増加しはじめ、マシンへの負荷が増加しています。**ベンチマーク/負荷試験**と**プロファイリング**は**MySQLデータベース**がメモリとCPUリソースをどんどん占有し、他方でユーザコンテンツがディスクを圧迫していることを示唆しています。

これまでは**垂直スケーリング**で問題に対処することができました。不幸なことに、今回はそれは高くなりすぎであり、またそれでは**MySQLデータベース**と**ウェブサーバ**を個別にスケールさせることができません。

#### 目標

* 1マシンへの負荷を軽減して、個別のスケーリングができるようにしましょう
    * 静的コンテンツを分離して**オブジェクトストア**に保存する
    * **MySQLデータベース**を別のマシンに移動する
* 欠点
    * これらの変更は複雑さを増し、**ウェブサーバ**が**オブジェクトストア**と**MySQLデータベース**を指すように変更しなければならない
    * 新しい部品をセキュアにするために追加のセキュリティ対策が必要
    * AWSのコストも増えるかもしれませんが、自分で動揺のシステムを管理するコストと比較検討すべきでしょう


#### 静的コンテンツを分離して保存しよう

* 静的コンテンツを保存するためにS3のようなマネージドな**オブジェクトストア**を使うことを考えましょう
    * 高度にスケーラブルで信頼性がある
    * サーバ側で暗号化
* 静的コンテンツをS3に移す
    * ユーザファイル
    * JS
    * CSS
    * 画像
    * 動画

#### MySQLデータベースを分離したマシンに移そう

* RDS(Amazon Relational Database Service)といったマネージドな**MySQLデータベース**を利用することを考えましょう
    * 管理やスケーリングがシンプル
    * 複数のアベイラビリティゾーンが利用できる
    * 保存データの暗号化(encryption at rest)

#### システムをセキュアにする

* データを通信時、保存時に暗号化する
* バーチャルプライベートクラウド(VPC)を使う(訳注: 自社でのみ運用するクラウドをプライベートクラウドという)
    * インターネットとトラフィックをやりとりっできるように、1つの**ウェブサーバ**にパブリックサブネットを設定する
        * 訳注: パブリックサブネットは、プライベートクラウドで外部と通信できるマシンを特定するためのサブネットらしい
    * 外部からのアクセスを防ぐために、その他すべてにプライベートサブネットを適用する
    * それぞれの部品はホワイトリストに入っているIPからのみポートを開ける
* 動揺のパターンはこの演習ののこりにでてくる新しい部品にも適用されます

**トレードオフ、代替案、さらなる詳細**

* See the [Security](https://github.com/donnemartin/system-design-primer#security) section

### Users++

![Imgur](http://i.imgur.com/raoFTXM.png)

#### 前提



Our **Benchmarks/Load Tests** and **Profiling** show that our single **Web Server** bottlenecks during peak hours, resulting in slow responses and in some cases, downtime.  As the service matures, we'd also like to move towards higher availability and redundancy.

#### Goals

* The following goals attempt to address the scaling issues with the **Web Server**
    * Based on the **Benchmarks/Load Tests** and **Profiling**, you might only need to implement one or two of these techniques
* Use [**Horizontal Scaling**](https://github.com/donnemartin/system-design-primer#horizontal-scaling) to handle increasing loads and to address single points of failure
    * Add a [**Load Balancer**](https://github.com/donnemartin/system-design-primer#load-balancer) such as Amazon's ELB or HAProxy
        * ELB is highly available
        * If you are configuring your own **Load Balancer**, setting up multiple servers in [active-active](https://github.com/donnemartin/system-design-primer#active-active) or [active-passive](https://github.com/donnemartin/system-design-primer#active-passive) in multiple availability zones will improve availability
        * Terminate SSL on the **Load Balancer** to reduce computational load on backend servers and to simplify certificate administration
    * Use multiple **Web Servers** spread out over multiple availability zones
    * Use multiple **MySQL** instances in [**Master-Slave Failover**](https://github.com/donnemartin/system-design-primer#master-slave-replication) mode across multiple availability zones to improve redundancy
* Separate out the **Web Servers** from the [**Application Servers**](https://github.com/donnemartin/system-design-primer#application-layer)
    * Scale and configure both layers independently
    * **Web Servers** can run as a [**Reverse Proxy**](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
    * For example, you can add **Application Servers** handling **Read APIs** while others handle **Write APIs**
* Move static (and some dynamic) content to a [**Content Delivery Network (CDN)**](https://github.com/donnemartin/system-design-primer#content-delivery-network) such as CloudFront to reduce load and latency

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

### Users+++

![Imgur](http://i.imgur.com/OZCxJr0.png)

**Note:** **Internal Load Balancers** not shown to reduce clutter

#### Assumptions

Our **Benchmarks/Load Tests** and **Profiling** show that we are read-heavy (100:1 with writes) and our database is suffering from poor performance from the high read requests.

#### Goals

* The following goals attempt to address the scaling issues with the **MySQL Database**
    * Based on the **Benchmarks/Load Tests** and **Profiling**, you might only need to implement one or two of these techniques
* Move the following data to a [**Memory Cache**](https://github.com/donnemartin/system-design-primer#cache) such as Elasticache to reduce load and latency:
    * Frequently accessed content from **MySQL**
        * First, try to configure the **MySQL Database** cache to see if that is sufficient to relieve the bottleneck before implementing a **Memory Cache**
    * Session data from the **Web Servers**
        * The **Web Servers** become stateless, allowing for **Autoscaling**
    * Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>
* Add [**MySQL Read Replicas**](https://github.com/donnemartin/system-design-primer#master-slave-replication) to reduce load on the write master
* Add more **Web Servers** and **Application Servers** to improve responsiveness

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

#### Add MySQL read replicas

* In addition to adding and scaling a **Memory Cache**, **MySQL Read Replicas** can also help relieve load on the **MySQL Write Master**
* Add logic to **Web Server** to separate out writes and reads
* Add **Load Balancers** in front of **MySQL Read Replicas** (not pictured to reduce clutter)
* Most services are read-heavy vs write-heavy

*Trade-offs, alternatives, and additional details:*

* See the [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) section

### Users++++

![Imgur](http://i.imgur.com/3X8nmdL.png)

#### Assumptions

Our **Benchmarks/Load Tests** and **Profiling** show that our traffic spikes during regular business hours in the U.S. and drop significantly when users leave the office.  We think we can cut costs by automatically spinning up and down servers based on actual load.  We're a small shop so we'd like to automate as much of the DevOps as possible for **Autoscaling** and for the general operations.

#### Goals

* Add **Autoscaling** to provision capacity as needed
    * Keep up with traffic spikes
    * Reduce costs by powering down unused instances
* Automate DevOps
    * Chef, Puppet, Ansible, etc
* Continue monitoring metrics to address bottlenecks
    * **Host level** - Review a single EC2 instance
    * **Aggregate level** - Review load balancer stats
    * **Log analysis** - CloudWatch, CloudTrail, Loggly, Splunk, Sumo
    * **External site performance** - Pingdom or New Relic
    * **Handle notifications and incidents** - PagerDuty
    * **Error Reporting** - Sentry

#### Add autoscaling

* Consider a managed service such as AWS **Autoscaling**
    * Create one group for each **Web Server** and one for each **Application Server** type, place each group in multiple availability zones
    * Set a min and max number of instances
    * Trigger to scale up and down through CloudWatch
        * Simple time of day metric for predictable loads or
        * Metrics over a time period:
            * CPU load
            * Latency
            * Network traffic
            * Custom metric
    * Disadvantages
        * Autoscaling can introduce complexity
        * It could take some time before a system appropriately scales up to meet increased demand, or to scale down when demand drops

### Users+++++

![Imgur](http://i.imgur.com/jj3A5N8.png)

**Note:** **Autoscaling** groups not shown to reduce clutter

#### Assumptions

As the service continues to grow towards the figures outlined in the constraints, we iteratively run **Benchmarks/Load Tests** and **Profiling** to uncover and address new bottlenecks.

#### Goals

We'll continue to address scaling issues due to the problem's constraints:

* If our **MySQL Database** starts to grow too large, we might consider only storing a limited time period of data in the database, while storing the rest in a data warehouse such as Redshift
    * A data warehouse such as Redshift can comfortably handle the constraint of 1 TB of new content per month
* With 40,000 average read requests per second, read traffic for popular content can be addressed by scaling the **Memory Cache**, which is also useful for handling the unevenly distributed traffic and traffic spikes
    * The **SQL Read Replicas** might have trouble handling the cache misses, we'll probably need to employ additional SQL scaling patterns
* 400 average writes per second (with presumably significantly higher peaks) might be tough for a single **SQL Write Master-Slave**, also pointing to a need for additional scaling techniques

SQL scaling patterns include:

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

To further address the high read and write requests, we should also consider moving appropriate data to a [**NoSQL Database**](https://github.com/donnemartin/system-design-primer#nosql) such as DynamoDB.

We can further separate out our [**Application Servers**](https://github.com/donnemartin/system-design-primer#application-layer) to allow for independent scaling.  Batch processes or computations that do not need to be done in real-time can be done [**Asynchronously**](https://github.com/donnemartin/system-design-primer#asynchronism) with **Queues** and **Workers**:

* For example, in a photo service, the photo upload and the thumbnail creation can be separated:
    * **Client** uploads photo
    * **Application Server** puts a job in a **Queue** such as SQS
    * The **Worker Service** on EC2 or Lambda pulls work off the **Queue** then:
        * Creates a thumbnail
        * Updates a **Database**
        * Stores the thumbnail in the **Object Store**

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

### SQL scaling patterns

* [Read replicas](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

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
