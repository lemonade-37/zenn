---
title: "ElastiCache（Redis）の必要性について勉強しながら検討した"
emoji: "🗂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Redis", "Sidekiq", "Docker", "ElastiCache"]
published: true
published_at: 2024-06-10 12:00
---
## 自己紹介
はじめまして、はる（[@lemonade_37](https://twitter.com/lemonade_37)）と申します。
駆け出しエンジニアとして働き始めて約3ヶ月が経過しました🐣


## 概要
メールの自動送信機能を実装するためにSidekiq（Redis）を使用しました。
本番環境でElastiCacheを使用したところ、請求金額が高額になってしまい、Redis周りの構成を見直しました😣
Dockerの基礎的な部分も含めて調べたことについてまとめているため、ご存知の方にとっては冗長に感じる表現である場合があります。

### 環境
- Docker
- Ruby 3.2.3
- Rails 7.1.3
- インフラ環境（見直し前の状態）
    ![](/images/elasticache-redis/image01.png)


## 経緯
色々なバックグラウンド処理の方法を調べて、ActiveJob＋Sidekiq（Redis）　を使うことにしました。
この時の調査とローカルでの実装についてはこちらの記事を参照ください。

https://zenn.dev/lemonade_37/articles/rails-docker-sidekiq

自分はPaaSでしかデプロイしたことが無く、業務でのAWSのインフラ環境は先輩が構築したものでした。
本番環境でもSidekiqやRedisを使う方法を考えたときに、
- アプリサーバーが落ちた時にキューが削除されては困るから、キューを保持しておくセッションサーバーは別で必要なのではないか？
- 本番環境でDBは自分で作らないといけないのと同じで、Redisも本番環境用に作らないといけないのでは？
- 調べていると、ElastiCacheというサービスがあると知る💡

という流れで、しっかり理解しないままElastiCacheを使って実装してしまいました。
ElastiCache導入後1ヶ月ほど経過した段階で、月の料金に数十ドルかかっていることが発覚😇

ElastiCache周りの構成を見直すため、改めて今の構成でわからないところを調査しました。

## ボリュームとは何かちゃんと理解できていない
そもそもDockerについても理解が浅く、ローカルではRedisに保持するキューはボリュームで管理していましたが、ボリュームについてちゃんとわかっていませんでした。
　　
ボリュームとは、**データを永続化できる場所**のことです。
コンテナは、Dockerを使うときに`docker compose up`・`docker compose down`コマンドで都度作成され、削除されますが、その度にDBのデータも消えてしまっては困ります。
　　
DB内のデータ等はボリュームとして持っておくことで、毎回そのデータをコンテナにコピーする形で、同じデータを永続的に扱うことができます。
下記図のようなイメージです。
![](/images/elasticache-redis/image02.png)

:::message
Docker における「マウント」とは、コンテナの外にあるデータを、コンテナの中で利用できる状態にすることを意味します。[1)](#引用記事)
:::


## SidekiqとRedisの違いや関係性
今まで、技術記事等で`Sidekiq（Redis）`と表現していたけどそれぞれの違いについて明確に説明できませんでした😇
　　
Redisとは、**インメモリデータストア**のことです。
Redis内部の説明については、こちらの記事が大変わかりやすかったため、引用させていただきます。[2)](#引用記事)

https://qiita.com/keinko/items/60c844bcf329bd3f4af8#redis%E3%81%AE%E4%BB%95%E7%B5%84%E3%81%BF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8

　　
Sodekiqとは、**キュー管理のツール**で、キューを保持したり、今すぐ実行したり、スケジュール管理したりできます。
Sidekiqは、実行待ちのキューを保持しておく場所としてRedisを使用しています。

## ElastiCacheを使う場合と使わない場合の違い
本番環境でElastiCacheを使わなければいけないと思い込んでいたが、ローカルではボリュームを使用しているため、本番環境でもボリュームを使う方法ではできないのか？
と先輩エンジニアより助言をいただきました。
　　
#### ローカル環境
まず、ローカルでのDocker環境はこのようになっています。

![](/images/elasticache-redis/image03.png)
　　
#### 本番環境（ElastiCache使用）
次に、見直し前の本番環境はこのようになっています。
今まで本番環境の`EC2`・`Ubuntu`・`Docker`等様々な言葉が出てきて、それぞれの親子関係がわからず混乱していましたが、調べたところ、`Ubuntu`としての`EC2`インスタンスを作成して使用することが多いようです。
（厳密には違うかも知れませんが、`EC2`そのもの＝`Ubuntu`というように理解しました）

また、ElastiCacheを使用する場合は、Docker内でRedisコンテナを作成したり、キューをボリュームで管理する必要がなくなります。

![](/images/elasticache-redis/image04.jpg)
　　
#### 本番環境（ElastiCacheなし）
ElastiCacheなしだと下記の図のように置き換わります。
EC2内Docker内でredisコンテナを作成し、キューはボリュームで管理します。
この時、`docker system prune -a`などのコマンドでボリュームを削除しないように注意します。

![](/images/elasticache-redis/image05.jpg)

## 結局、ElastiCacheを使用するかしないかどちらがいいのか？
今回においては、**ElastiCacheなしで、本番環境でもキューをボリュームで管理することにしました。**
今回は、Sidekiqの使用用途が簡単なメール送信のみで、頻度もまだ月1回等少ない状況でした。
また、インフラ環境にそこまでコストをかけられる状況ではありませんでした。
そのため、先輩エンジニアとも相談し、この方法を選択しました。
　　　　
ただし、今後サービス規模が大きくなっていった際には、EC2のサイズを大きくしたり、数を増やすといった対応が必要になります。
その際に、Redisコンテナが2つできてメールが二重送信される可能性があることや、ElastiCacheを使用しRedisをEC2外に外に切り出す方がEC2の容量的にも良い場合もあります。
　　
そういったことを念頭に置いた上で、今はElastiCacheなしで運用することとなりました。


## まとめ
Dockerやインフラ環境の解像度が少し上がりました。
今回は早めに気づきましたが、わからないまま実装することで、不要なコストが発生してそのまま気づかれず大変なことになってしまう恐れがあるため😇
しっかり1つ1つ理解するよう勉強していこうと思いました🏃‍♀️
ここまで読んでいただきありがとうございました！


　
## 参考・引用記事
#### 引用記事
1）[さわって理解するDocker入門 第7回 | オブジェクトの広場](https://www.ogis-ri.co.jp/otc/hiroba/technical/docker/part7.html#:~:text=Docker%20%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E3%80%8C%E3%83%9E%E3%82%A6%E3%83%B3%E3%83%88%E3%80%8D%E3%81%A8%E3%81%AF%E3%80%81%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%AE%E5%A4%96%E3%81%AB%E3%81%82%E3%82%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%82%92%E3%80%81%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%AE%E4%B8%AD%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%A7%E3%81%8D%E3%82%8B%E7%8A%B6%E6%85%8B%E3%81%AB%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%82%92%E6%84%8F%E5%91%B3%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)
2）[初心者による初心者のためのRedis解説](https://qiita.com/keinko/items/60c844bcf329bd3f4af8#redis%E3%81%AE%E4%BB%95%E7%B5%84%E3%81%BF%E3%81%AE%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8)

#### 参考記事
- [Docker、ボリューム(Volume)について真面目に調べた](https://qiita.com/gounx2/items/23b0dc8b8b95cc629f32)
- [３部: ボリューム｜実践 Docker - ソフトウェアエンジニアの「Docker よくわからない」を終わりにする本](https://zenn.dev/suzuki_hoge/books/2022-03-docker-practice-8ae36c33424b59/viewer/3-4-volume)
- [３部: 構成の全体図｜実践 Docker - ソフトウェアエンジニアの「Docker よくわからない」を終わりにする本](https://zenn.dev/suzuki_hoge/books/2022-03-docker-practice-8ae36c33424b59/viewer/3-1-summary)
- [Sidekiqってどんなキック？ | 働くひとと組織の健康を創る iCARE](https://dev.icare.jpn.com/dev_cat/sidekiq/)
- [Sidekiq(サイドキック)とは何なのか、調べてみた | Offers Tech Blog](https://zenn.dev/overflow_offers/articles/20230130-how-to-use-sidekiq)
- [SidekiqはRedisに何を書き込んでいるのか](https://qiita.com/hosopy/items/d2c87b6489991091ddab)
- [[WIP]RedisとSidekiqの説明](https://qiita.com/aaaaanochira/items/1d2df839ed6bc875ef6f)
- [インメモリーDB製品はバッチで有効 | 日経クロステック（xTECH）](https://xtech.nikkei.com/it/article/COLUMN/20051207/225867/)
- [Ubuntuとは | Ubuntu Japanese Team](https://www.ubuntulinux.jp/ubuntu)
- [ステップ 1: Ubuntu Amazon EC2 インスタンスを作成する - Amazon Kinesis Video Streams](https://docs.aws.amazon.com/ja_jp/kinesisvideostreams/latest/dg/gs-ubuntu.html)
