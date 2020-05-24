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

**ベンチマーク/負荷試験**と**プロファイリング**は(いまのところ1つの)**ウェブサーバ**がピーク時にボトルネックとなり、遅いレスポンスや時にはダウンタイムを引き起こしていることを明らかにしました。サービスが成熟するについれて、高い可用性や冗長性の方向にも進みたいものです。

#### 目標

* 次の目標は**ウェブサーバ**のスケーリングの問題に取り組もうとするものです
    * **ベンチマーク/負荷試験**と**プロファイリング**を踏まえ、次の1、2個の技法を実装すれば十分でしょう
* 単一障害点(SPF, Single Points of Failrure)に対処し増大する負荷に対応するため[**水平スケーリング**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E6%B0%B4%E5%B9%B3%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%AA%E3%83%B3%E3%82%B0)を使う
    * Amazon ELBやHAProxy(訳注：ロードバランサの機能を提供できるOSS)といった[**ロードバランサ**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%AD%E3%83%BC%E3%83%89%E3%83%90%E3%83%A9%E3%83%B3%E3%82%B5%E3%83%BC)を追加する
        * ELBは高可用である
        * もしも自分の**ロードバランサ**を設定しているなら、複数サーバを[アクティブ-アクティブ](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%A2%E3%82%AF%E3%83%86%E3%82%A3%E3%83%96%E3%82%A2%E3%82%AF%E3%83%86%E3%82%A3%E3%83%96)や[アクティブ-パッシブ](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%A2%E3%82%AF%E3%83%86%E3%82%A3%E3%83%96%E3%83%91%E3%83%83%E3%82%B7%E3%83%96)で複数のアベイラビリティゾーンに設置すると可用性が上がります
        * **ロードバランサ**の側でSSLを数量してバックエンドサーバでの計算負荷を軽減し、認証管理を単純化します。
    * 複数の**ウェブサーバ**を複数のアベイラビリティゾーンに展開する
    * 複数の**MySQL**インスタンスを複数のアベイラビリティゾーンをまたがって[**マスタースレーブフェイルオーバー**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC%E3%82%B9%E3%83%AC%E3%83%BC%E3%83%96-%E3%83%AC%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)で使い、冗長性を向上させる
* **ウェブサーバ**を[**アプリケーションサーバ**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E5%B1%A4)から切り出す
    * 両方のレイヤを独立にスケールし設定する
    * **ウェブサーバ**は[**リバースプロキシ**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%AA%E3%83%90%E3%83%BC%E3%82%B9%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7web%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC)として稼動することができる
    * たとえば、**アプリケーションサーバ**のいくらかが**Read API**を、残りが**Write API**を担当するようにできる

* 静的(一部動的)コンテンツをCloudFrontのような[**コンテンツデリバリネットワーク(CDN)**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%B3%E3%83%B3%E3%83%86%E3%83%B3%E3%83%84%E3%83%87%E3%83%AA%E3%83%90%E3%83%AA%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AFcontent-delivery-network)に移し負荷とレイテンシを軽減する


**トレードオフ、代替案、さらなる詳細**

* See the linked content above for details

### Users+++

![Imgur](http://i.imgur.com/OZCxJr0.png)

**注意:** **内部ロードバランサ**はごちゃごちゃしないために描いてません

#### 仮定

**ベンチマーク/負荷試験**と**プロファイリング**は読み込みが重く(書き込みの100倍)、データベースは高い読み込みリクエストからくる低性能に苦しんでいます。


#### 目標

* 次の目標は**MySQLデータベース**のスケーリングの問題に対処しようとするものです
    * **ベンチマーク/負荷試験**と**プロファイリング**によれば、1つか2つの技法を実装すれば十分でしょう
* 次のデータをElasticcacheといった[**メモリキャッシュ**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5)に移動し、負荷とレイテンシを軽減する:
    * **MySQL**からよくアクセスされるコンテンツを移す
        * まず、**メモリキャッシュ**を実装する前に**MySQLデータベース**のキャッシュを設定してボトルネックを解消するのに十分かを見るべきでしょう
    * **ウェブサーバ**からセッションデータを移す
        * **ウェブサーバ**はステートレスになり、**オートスケーリング**ができるようになる
    ** 1MBをメモリからシーケンシャルに読むには約250msかかる一方、SSDから読むには約4倍、ディスクから読むには約80倍かかります。<sup><a href=https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E5%85%A8%E3%81%A6%E3%81%AE%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9E%E3%83%BC%E3%81%8C%E7%9F%A5%E3%82%8B%E3%81%B9%E3%81%8D%E3%83%AC%E3%82%A4%E3%83%86%E3%83%B3%E3%82%B7%E3%83%BC%E5%80%A4>1</a></sup>
* [**MySQL読み込みレプリケーション**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC%E3%82%B9%E3%83%AC%E3%83%BC%E3%83%96-%E3%83%AC%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)を追加し、書き込みマスターへの負荷を軽減する
* さらに**ウェブサーバ**と**アプリケーションサーバ**を追加してレスポンスを上げる



**トレードオフ、代替案、さらなる詳細**

* See the linked content above for details

#### SQL読み込みレプリケーションを追加する

* **メモリキャッシュ**を追加してスケールさせる他にも、**MySQL読み込みレプリケーション**が**MySQL書き込みマスター**の負荷を軽減するのに役立ちます
* **ウェブサーバ**にロジックを追加して、書き込みと読み込みを分割する
* **MySQL読み込みレプリケーション**の前に**ロードバランサ**を置く(ごちゃごちゃを避けるために図示していない)
    * 訳注: 内部ロードバランサは省略ってそういうことか…
* ほとんどのサービスは読み込みが重いか書き込みが重いかです



* In addition to adding and scaling a **Memory Cache**, **MySQL Read Replicas** can also help relieve load on the **MySQL Write Master**
* Add logic to **Web Server** to separate out writes and reads
* Add **Load Balancers** in front of **MySQL Read Replicas** (not pictured to reduce clutter)
* Most services are read-heavy vs write-heavy

**トレードオフ、代替案、さらなる詳細**

* See the [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) section

### Users++++

![Imgur](http://i.imgur.com/3X8nmdL.png)

#### 仮定

**ベンチマーク/負荷試験**と**プロファイリング**は、トラフィックがアメリカの業務時間中に急上昇し、ユーザが退勤したころに急減するということを示しました。実際の負荷に応じてサーバを自動的に立ち上げたり落としたりすればコストを削減できるでしょう。うちは小さい店なので、**オートスケーリング**とその他操作のためにできるだけDevOpsを自動化したいと思います。

#### 目標

* **オートスケーリング**を追加し、必要なだけプロビジョンできるようにする
    * トラフィックの急上昇についていく
    * 使っていないインスタンスを落としてコストを削減する
* DevOpsを自動化する
    * Chef, Puppet, Ansible, etc
* ボトルネックに対処するため引き続きメトリックを監視する
    * **Host level** - Review a single EC2 instance
    * **Aggregate level** - Review load balancer stats
    * **Log analysis** - CloudWatch, CloudTrail, Loggly, Splunk, Sumo
    * **External site performance** - Pingdom or New Relic
    * **Handle notifications and incidents** - PagerDuty
    * **Error Reporting** - Sentry



#### オートスケーリングを追加する

* AWS**オートスケーリング**といったマネージドなサービスを利用することを検討する
    * それぞれの**ウェブサーバ**に1グループ、**アプリケーションサーバ**タイプにも1つグループを作り、それぞれのグループを複数のアベイラビリティーゾーンに配置する
    * インスタンスの最小数と最大数を設定する
    * CloudWatchの情報をつかってスケールアップとダウンを引き起こす
        * 予想できる負荷には時刻メトリックを使うか、
        * 時間にわたるメトリックを使う:
            * CPU負荷
            * レイテンシ
            * ネットワークトラフィック
            * 独自メトリック
    * 欠点
        * オートスケーリングは複雑にしてしまう
        * システムが適切に需要増に応じてスケールアップしたり、需要減に応じてスケールダウンするのに時間がかかるかもしれない


### Users+++++

![Imgur](http://i.imgur.com/jj3A5N8.png)

**注意:** **内部ロードバランサ**はごちゃごちゃしないために描いてません

#### 仮定

サービスが制約のところに並べた状況に近づいてきたので、**ベンチマーク/負荷試験**と**プロファイリング**を実行して新しいボトルネックを見つけることを繰り返します。

#### 目標

ひきつづきスケーリングの問題に取り組み制約の問題を解決します:

* もし**MySQLデータベース**が大きくなりすぎたようなら、限られた期間のデータだけをデータベースに残し、残りはReshiftといったデータウェアハウスにしまう
    * Redshiftのようなデータウェアハウスは一ヶ月1TB程度の新規コンテンツであれば易々と取り扱えます
* 平均秒間40Kの読み込みリクエストについては、人気コンテンツへの読み込みトラフィックは**メモリキャッシュ**をスケールすることで対処でき、またこれは不均等に分布したトラフィックとトラフィックの急上昇を捌くにも有効です
    * **SQL読み込みレプリケーション**はキャッシュミスを処理するのに問題が起こるかもしれず、SQLをスケールさせるための別のパターンを利用する必要があるかもしれません
* 秒間平均400の書き込み要求(ピーク時はもっと多いでしょう)は1つの**SQL書き込みマスタースレーブ**には苦しいかもしれず、さらにスケールさせる技法が必要かもしれません。
    * 訳注: 1KB/書き込みなので、400KB/秒の書き込みとなる。

SQL scaling patterns include:

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

さらに高頻度の読み込み書き込み要求に対応するには、データをDynamoDBといった適切な[**NoSQLデータベース**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#nosql)に移すことを考えるべきでしょう。

さらに[**アプリケーションサーバ**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E5%B1%A4)を分割して個別にスケールさせることもできます。バッチ処理や計算でリアルタイムである必要がないものは**キュー**や**ワーカ**を使って[**非同期**](https://github.com/donnemartin/system-design-primer/blob/master/README-ja.md#%E9%9D%9E%E5%90%8C%E6%9C%9F%E5%87%A6%E7%90%86)に実行できます。

* 例えば写真サービスでは、写真のアップロードとサムネイル作成は分割できます:
    * **クライアント**は写真をアップロード
    * **アプリケーションサーバ**は仕事をSQS(Amazon SQS)といった**キュー**に追加
    * EC2やLambdaといった**ワーカサービス**が**キュー**から仕事を引っ張ってきて:
        * サムネイルをつくる
        * **データベース**を更新
        * サムネイルを**オブジェクトストア**に保存



**トレードオフ、代替案、さらなる詳細**

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
